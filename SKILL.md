---
name: ocr-pdf-to-searchable
description: Turn a scanned/image-only PDF into a searchable PDF (original images kept + aligned invisible OCR text layer) using PyMuPDF + Tesseract on Windows. Use when a user asks to "OCR a PDF", "make a PDF searchable", or "extract text from a scanned PDF" and a proper tool like OCRmyPDF/Ghostscript is NOT installed. Handles very large books and oversized foldout pages.
---

# OCR a PDF into a Searchable PDF (PyMuPDF + Tesseract, Windows)

## When to use
- User wants a scanned/image PDF made searchable (Ctrl+F works) or wants its text extracted.
- Environment has **no OCRmyPDF / Ghostscript** but has Tesseract + PyMuPDF (`fitz`) + Pillow.
- Typical Windows setup: Tesseract at `C:/Program Files/Tesseract-OCR/tesseract.exe`; Python with `fitz`, `pillow`, `pypdf`.

## Key facts / gotchas
- Native Windows Python does **NOT** understand Git-Bash `/d/...` paths. Always use `D:/...` or `D:\\...` (Windows-style) inside Python. Bash-only commands can use `/d/...`.
- A PDF "already has a text layer" from old digitization is often **worse** than fresh Tesseract OCR (bad spacing, missing pages). Re-OCR usually improves it. Still confirm on a content page first.
- **fitz `page.get_pixmap(dpi=300)` itself raises `FzErrorLimit: code=5: Overly large image`** for oversized foldout plates (huge physical pages → giant bitmap). This happens BEFORE Tesseract and silently crashes a `multiprocessing` run. Fix: render adaptively — try 300 DPI, on `FzErrorLimit`/`Exception` retry at lower DPI (250→200→150→120→100→72) until it succeeds. Then downsample the PIL image for Tesseract (below).
- **Tesseract / Leptonica reject "Overly large image"** when a rendered page exceeds ~32767 px per side or ~1e8 pixels. Foldout plates/diagrams in old books hit this. Fix: downsample before OCR (see below). Otherwise those pages fail AND may be dropped.
- **Multiprocessing crash (critical):** if a worker raises, the Pool tries to pickle the traceback, which references the fitz `Document`/`Page` SWIG object and dies with `TypeError: cannot pickle 'swig_runtime_data...SwigPyObject'`. This aborts the whole run. **Always wrap the per-page worker body in try/except and return `(pno, text, err_str)`** so no exception ever crosses the process boundary.
- `tesseract input.png outbase -l eng --psm 3 -c preserve_interword_spaces=1 pdf` emits a single-page searchable PDF (image + invisible aligned text). Merge these per-page PDFs with `fitz.insert_pdf`.

## Robust pipeline (per page)
1. **Render adaptively** (handles fitz's own "Overly large image" limit):
   ```python
   pix = None
   for d in (300, 250, 200, 150, 120, 100, 72):
       try:
           pix = page.get_pixmap(dpi=d)
           break
       except Exception:
           pix = None
   if pix is None:
       raise RuntimeError("get_pixmap failed at all DPI levels")
   ```
2. Convert to PIL: `img = Image.frombytes("RGB",[pix.width,pix.height],pix.samples)`.
3. **Downsample if too large** (prevents Tesseract "Overly large image"):
   ```python
   MAXSIDE, MAXAREA = 16000, 100_000_000
   w,h = img.size
   if w>MAXSIDE or h>MAXSIDE or w*h>MAXAREA:
       s = min(MAXSIDE/w, MAXSIDE/h, (MAXAREA/(w*h))**0.5)
       img = img.resize((max(1,int(w*s)), max(1,int(h*s))), Image.LANCZOS)
   ```
4. Save PNG, run Tesseract to a per-page PDF (`... pdf`).
5. **Never drop a page**: in the merge step, for any page whose OCR PDF is missing/failed, copy the ORIGINAL page instead: `out.insert_pdf(src_doc, from_page=pno, to_page=pno)`. This guarantees page count stays identical.

## Scale & speed
- ~5 s/page single-threaded at 300 DPI. Parallelize with `multiprocessing.Pool` (workers open their own `fitz` doc via initializer; pass page indices). Use ~6 workers.
- Make it **resumable**: skip pages whose output PDF already exists (`if os.path.exists(pg) and size>2000: skip`). Lets you fix a bug and re-run without redoing good pages.
- Output size ≈ sum of page images; for 600-page books expect 1–3 GB. Fine on modern disks.

## Verify before delivering
```python
d = fitz.open(out_pdf)
n = d.page_count
text_pages = sum(1 for i in range(n) if d[i].get_text().strip())
# assert text_pages ~= n and a known keyword is found on a content page
```
Ignore `FT_New_Memory_Face ... unknown file format` warnings — they're harmless and text extraction still works.

## Reference implementation
Full script: `D:/ocr_work/ocr_build.py` (6-worker, per-page Tesseract PDF → fitz merge, downsample + copy-original fallback, resume support).
