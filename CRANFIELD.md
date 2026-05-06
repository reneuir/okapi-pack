# BM25 Retrieval on Cranfield with OKAPI-PACK

This guide walks through building a BM25 index over the [Cranfield](https://ir-datasets.com/cranfield.html) dataset and evaluating retrieval quality (NDCG) — all inside the `mam10eks/okapi-pack:0.0.1-dev` container.

The companion notebook [`cranfield_bm25.ipynb`](cranfield_bm25.ipynb) contains the full runnable code.

---

## Hardware requirement

The OKAPI-PACK binaries are **32-bit x86**. They run correctly on real x86 hardware. On Apple Silicon Macs (ARM), QEMU emulation is used but some binaries crash.

**Recommended: use [GitHub Codespaces](https://github.com/features/codespaces)** — click *Code → Codespaces → + Create codespace* in this repo. Codespaces runs on x86 and starts the devcontainer automatically.

---

## Overview

```
Cranfield docs            exchange format         OKAPI database
(ir_datasets)    ──────►  cranfield.exch   ──────►  bibfile + indexes
                                                         │
Cranfield queries                                        │  i1+ (BM25)
(ir_datasets)   ──────────────────────────────────────►  │
                                                         ▼
                                               ranked lists per query
                                                         │
Cranfield qrels                                          │  ir_measures
(ir_datasets)   ──────────────────────────────────────►  │
                                                         ▼
                                                    NDCG@10 / @1000
```

---

## 1. Install Python dependencies

```bash
pip install ir_datasets ir_measures
```

`ir_datasets` loads Cranfield; `ir_measures` computes NDCG. Both wrap well-known tools (Terrier / trec_eval) so you don't have to implement metrics yourself.

---

## 2. Environment variables

OKAPI tools look for database descriptor files on `BSS_PARMPATH`:

```bash
export BSS_PARMPATH=/okapi-pack/okapi-pack/databases
export BSS_TEMPPATH=/tmp
```

You can also set these in Python via `os.environ` — the notebook does this automatically.

---

## 3. Database descriptor files

OKAPI needs four config files per database, all stored in `$BSS_PARMPATH`.

### `cranfield` — main descriptor

```
name=cranfield
lastbibvol=0
bib_basename=cranfield.bib
bib_dir=/data/cranfield/bibfiles/
bibsize=2047
real_bibsize=0
display_name=cranfield
explanation=Cranfield Collection 1400 documents
nr=0
nf=2
f_abbrev=DN
f_abbrev=TX
rec_mult=4
fixed=0
db_type=ai
has_lims=0
maxreclen=65536
ni=2
last_ixvol=0
ix_stem=/data/cranfield/bibfiles/cranfield
ix_volsize=2047
ix_type=9
last_ixvol=0
ix_stem=/data/cranfield/bibfiles/cranfield
ix_volsize=2047
ix_type=9
```

Key fields:
- `nf=2` — two fields: DN (document number) and TX (title + text)
- `db_type=ai` — abstract/indexing style (no paragraph file; simpler than `text`)
- `ni=2` — two indexes: keyword (0) and docno (1)
- `ix_type=9` — standard keyword index supporting BM25
- `bib_dir` / `ix_stem` — absolute paths to the directory where OKAPI stores its binary files; the notebook sets these to `/data/cranfield/bibfiles/`

### `cranfield.field_types`

```
1 LITERAL_NC
2 TEXT
```

Field 1 (DN) is a literal identifier (case-folded). Field 2 (TX) is free text.

### `cranfield.search_groups`

```
kw 1 0 words3 sstem gsl.empty 2 0 -1
dn 1 1 literal nostem gsl.empty 1 0 -1
```

Each line: `<index_name> <v1> <index_num> <token_type> <stem_type> <gsl_file> <field_nums...> -1`

- `kw` (index 0): keyword search on field 2, Lovins suffix stemmer (`sstem`), no stoplist (`gsl.empty`)
- `dn` (index 1): exact docno lookup on field 1

### `db_avail` — append one line

```
cranfield *
```

This grants all users access. The notebook appends this automatically if not already present.

---

## 4. Convert Cranfield to OKAPI exchange format

OKAPI's `convert_runtime` reads a stream of records from stdin. Each record is:

```
<docno>\x1e<title> <text>\x1d\n
```

- `\x1e` (hex 1E) = field separator (after field 1 = docno)
- `\x1d` (hex 1D) = record terminator

```python
import ir_datasets

FIELD_MARK  = '\x1e'  # 0x1E
RECORD_MARK = '\x1d'  # 0x1D

dataset = ir_datasets.load("cranfield")

with open('/data/cranfield/cranfield.exch', 'w', encoding='utf-8') as f:
    for doc in dataset.docs_iter():
        content = f"{doc.title} {doc.text}".replace(FIELD_MARK, ' ').replace(RECORD_MARK, ' ')
        f.write(f"{doc.doc_id}{FIELD_MARK}{content}{RECORD_MARK}\n")
```

> Sanitize `\x1e` / `\x1d` from document text to avoid format corruption.

---

## 5. Build the OKAPI database

```bash
convert_runtime -c $BSS_PARMPATH cranfield < /data/cranfield/cranfield.exch
```

On success you will see output like:
```
Fd 1, l 5, fcno 1
...
Reached end of input, last rec 1400
Total bibsize XXXXXX
1400 records output this run, total 1400
```

This creates the binary bibfile (`cranfield.bib`, `cranfield.bibdir`) in `bib_dir` and updates the descriptor with `nr=1400` and `real_bibsize=...`.

---

## 6. Build indexes

Two indexes are needed: the keyword index (for BM25 search) and the docno index (for lookup by ID).

```bash
# Keyword index (index 0) — include document lengths for BM25 normalisation
ix1 -doclens -c $BSS_PARMPATH cranfield 0 | ixf -c $BSS_PARMPATH cranfield 0

# Docno index (index 1)
ix1 -c $BSS_PARMPATH cranfield 1 | ixf -c $BSS_PARMPATH cranfield 1
```

After indexing, the bibfiles directory should contain files like:
```
cranfield.bib
cranfield.bibdir
cranfield.0.pi   cranfield.0.si   cranfield.0.oi   cranfield.0.di   cranfield.0.dlens
cranfield.1.pi   cranfield.1.si   cranfield.1.oi   cranfield.1.di
```

The `-doclens` flag writes `cranfield.0.dlens` — without it BM25 cannot normalise by document length.

---

## 7. BM25 query via `i1+`

`i1+` is the BSS (Binary Search System) interactive interface. It reads commands from stdin and writes responses to stdout, one response per command (flushed immediately with `-flush`).

### BSS command sequence for one query

```
CHOOSE cranfield
FIND t=aerodynam           → S0 np=342 t=aerodynam
WEIGHT                     → 4.217
FIND t=bluff               → S1 np=18 t=bluff
WEIGHT                     → 8.504
FIND s=0 w=4.217 s=1 w=8.504 op=bm25 target=1000
SHOW format=197 n=1000     → cran-0001 12.721
                              cran-0042  9.330
                              ...
```

Key BSS commands:
| Command | Description |
|---------|-------------|
| `CHOOSE <db>` | Open a database (resets all sets) |
| `FIND t=<term>` | Look up one term; returns set number and posting count (np) |
| `WEIGHT` | Compute Robertson–Sparck Jones IDF weight for the last FIND's np |
| `FIND s=<n> w=<w> ... op=bm25` | Weighted BM25 combination of sets |
| `SHOW format=197 n=<k>` | Output top-k: `<docno> <score>` per line |

> `FIND t=<term>` applies the index's stemmer (Lovins `sstem`) automatically.
> If a term is not in the index, `np=0` and weight ≈ 0 — safe to include.

### Two-pass Python implementation

Because `WEIGHT` must run before we know the weight values to put in `FIND ... op=bm25 w=<float>`, we use two `i1+` calls:

```python
import subprocess
import re

BSS_PARMPATH = '/okapi-pack/okapi-pack/databases'

def run_bm25_query(query_text: str, db: str = 'cranfield', top_k: int = 1000):
    """Run BM25 on `query_text`, return [(docno, score), ...] sorted by score desc."""
    terms = [t for t in query_text.lower().split() if t]
    if not terms:
        return []

    # ── Pass 1: look up each term and get its IDF weight ─────────────────────
    cmds1 = [f"CHOOSE {db}"]
    for term in terms:
        cmds1.append(f"FIND t={term}")
        cmds1.append("WEIGHT")

    r1 = subprocess.run(
        ["i1+", "-silent", "-flush", "-c", BSS_PARMPATH],
        input="\n".join(cmds1) + "\n",
        capture_output=True, text=True
    )

    # Parse: alternating FIND output / WEIGHT output
    set_weights = []          # [(set_num, weight), ...]
    lines = r1.stdout.splitlines()
    i = 0
    while i < len(lines):
        m = re.match(r"S(\d+)\s+np=\d+", lines[i])
        if m and i + 1 < len(lines):
            s = int(m.group(1))
            try:
                w = float(lines[i + 1].strip())
                set_weights.append((s, w))
                i += 2
                continue
            except ValueError:
                pass
        i += 1

    if not set_weights:
        return []

    # ── Pass 2: combine with BM25 and retrieve results ───────────────────────
    cmds2 = [f"CHOOSE {db}"]
    for term in terms:
        cmds2.append(f"FIND t={term}")   # recreate sets (same order → same nums)
    combine = "FIND " + " ".join(f"s={s} w={w:.4f}" for s, w in set_weights)
    combine += f" op=bm25 target={top_k}"
    cmds2.append(combine)
    cmds2.append(f"SHOW format=197 n={top_k}")

    r2 = subprocess.run(
        ["i1+", "-silent", "-flush", "-c", BSS_PARMPATH],
        input="\n".join(cmds2) + "\n",
        capture_output=True, text=True
    )

    # SHOW format=197 output: "<docno> <score>" per line
    results = []
    for line in r2.stdout.splitlines():
        parts = line.strip().split()
        if len(parts) == 2:
            try:
                results.append((parts[0], float(parts[1])))
            except ValueError:
                pass

    return sorted(results, key=lambda x: -x[1])
```

---

## 8. Run all queries

```python
import ir_datasets
from ir_measures import ScoredDoc

dataset = ir_datasets.load("cranfield")

run = []
for query in dataset.queries_iter():
    ranked = run_bm25_query(query.text)
    for rank, (docno, score) in enumerate(ranked):
        run.append(ScoredDoc(query_id=query.query_id, doc_id=docno, score=score))
```

---

## 9. Evaluate with NDCG

```python
import ir_measures
from ir_measures import nDCG

measures = [nDCG@10, nDCG@100, nDCG@1000]
results = ir_measures.calc_aggregate(measures, dataset.qrels_iter(), run)
for measure, value in results.items():
    print(f"{measure}: {value:.4f}")
```

Expected output for BM25 on Cranfield (k1=1.2, b=0.75, default OKAPI parameters):
```
nDCG@10:   ~0.42
nDCG@100:  ~0.45
nDCG@1000: ~0.47
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `Segmentation fault` in `convert_runtime` | Running on ARM / QEMU | Use GitHub Codespaces (x86) |
| `Can't read record 1` in `ix1` | Same QEMU issue | Use GitHub Codespaces |
| `ixinit(): can't open primary index` | Indexes not built | Re-run `ix1 \| ixf` steps |
| `DB_NOT_AVAIL` from `i1+` | `cranfield *` missing from `db_avail` | Append line and retry |
| `WEIGHT` returns `0.000` for all terms | No `-doclens` flag on `ix1` | Rebuild index 0 with `-doclens` |
| NDCG ≈ 0 | Parse error in query output | Check raw `i1+` output format; adjust parser |

---

## Reference: BSS command quick-reference

Full documentation is inside the container at `/okapi-pack/okapi-pack/docs/bss_commands`.

```
INFO version              # print BSS version
INFO databases            # list available databases
INFO database             # info about open database
CHOOSE <db>               # open database
FIND t=<term>             # lookup term (default index = kw)
FIND attr=dn t=<docno>    # lookup by docno
WEIGHT [n=<np>]           # IDF weight for last FIND
FIND s=0 w=<w> s=1 w=<w> op=bm25 target=<k>   # BM25 combine
SHOW format=197 n=<k>     # docno + weight, top k
SHOW format=1 n=1         # full document text, first result
DELETE all                # free all sets
```
