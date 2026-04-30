# Akuna Capital Options 101: Notes (Split by Chapter)

This folder contains my textbook-style notes based on introductory options market making concepts, plus a **chapter-split** version of the compiled PDF. Link to Complete PDF: https://drive.google.com/file/d/1kk4VDinYDHIUoRZm5yMgoM_Rij1aO_Sr/view?usp=sharing

## What’s in here

- **Source (LaTeX)**: `main.tex`
- **Compiled PDF**: `Akuna_Capital_Options_101.pdf`
- **Chapter PDFs (split output)**: `chapters_pdf/`

## Chapter PDFs

- **Cover page only**: `00_cover_page.pdf`
- **Front matter (everything after cover up to chapter 1)**: `00_front_matter_rest.pdf`
- **Per-chapter PDFs**: `01_*.pdf`, `02_*.pdf`, …
- **Index of all outputs + page ranges**: `INDEX.txt`

### Note on “merged” chapters

If two chapter titles begin on the **same physical PDF page**, they will appear in a **single output file** (PDFs can’t be split mid-page cleanly without rasterizing).

## Rebuild the full PDF (from LaTeX)

From this directory:

```bash
pdflatex -interaction=nonstopmode main.tex
pdflatex -interaction=nonstopmode main.tex
```

(Running twice ensures the table of contents is correct.)

## Recreate the chapter-split PDFs

The split outputs were generated from `Akuna_Capital_Options_101.pdf` by reading its bookmarks and writing one PDF per top-level bookmark range.

If you want to regenerate the split files, run this from the repo directory:

```bash
python - <<'PY'
from PyPDF2 import PdfReader, PdfWriter
from pathlib import Path

pdf_path = Path("Akuna_Capital_Options_101.pdf")
out_dir = Path("chapters_pdf")
out_dir.mkdir(parents=True, exist_ok=True)

# clean old outputs
for p in out_dir.glob("*.pdf"):
    p.unlink()
for p in out_dir.glob("*.txt"):
    p.unlink()

reader = PdfReader(str(pdf_path))
outline = reader.outline

def is_dest(obj):
    return isinstance(obj, dict) and "/Title" in obj and "/Page" in obj

def page_num(dest_dict):
    return reader.get_destination_page_number(dest_dict)

# collect top-level outline entries: (title, start_page_index)
raw = [(item["/Title"], page_num(item)) for item in outline if is_dest(item)]

# group titles by start page (preserve order)
page_to_titles = {}
order = []
for title, pn in raw:
    if pn not in page_to_titles:
        page_to_titles[pn] = []
        order.append(pn)
    if title not in page_to_titles[pn]:
        page_to_titles[pn].append(title)

starts = sorted(order)
chapters = []
for i, pn in enumerate(starts):
    next_pn = starts[i + 1] if i + 1 < len(starts) else len(reader.pages)
    chapters.append((page_to_titles[pn], pn, next_pn - 1))

# merge consecutive entries with identical title lists (rare, but possible)
merged = []
for titles, start, end in chapters:
    if merged and merged[-1][0] == titles and merged[-1][2] + 1 == start:
        merged[-1] = (merged[-1][0], merged[-1][1], end)
    else:
        merged.append((titles, start, end))

# cover/front matter pages are everything before first chapter start
first_start = merged[0][1] if merged else 0

# cover page only
w = PdfWriter()
w.add_page(reader.pages[0])
with (out_dir / "00_cover_page.pdf").open("wb") as f:
    w.write(f)

# remaining front matter (pages 2 .. first chapter start)
if first_start > 1:
    w = PdfWriter()
    for p in range(1, first_start):
        w.add_page(reader.pages[p])
    with (out_dir / "00_front_matter_rest.pdf").open("wb") as f:
        w.write(f)

index_lines = [
    "00_cover_page.pdf\tpage 1",
    f"00_front_matter_rest.pdf\tpages 2-{first_start}",
]

for idx, (titles, start, end) in enumerate(merged, start=1):
    joined = "__".join(titles)
    safe = "".join(c for c in joined if c.isalnum() or c in (" ", "-", "_")).strip().replace(" ", "_")
    fname = f"{idx:02d}_{safe}.pdf"

    w = PdfWriter()
    for p in range(start, end + 1):
        w.add_page(reader.pages[p])
    with (out_dir / fname).open("wb") as f:
        w.write(f)

    index_lines.append(f"{fname}\t{' | '.join(titles)}\tpages {start + 1}-{end + 1}")

(out_dir / "INDEX.txt").write_text("\\n".join(index_lines) + "\\n", encoding="utf-8")
print("Wrote outputs to", out_dir)
PY
```

