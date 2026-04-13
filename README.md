# Docling Exploration
## Introduction
In this experiment I have explored the Docling CLI and used it to parse a PDF and export it to multiple formats. I have also tried various flags to become familiar with the basic commands and functionality of Docling, which is part of the RAG support in Ramalama.

### Documents Used ###

For this task i have chosen the [Pytorch Conference brochure](https://events.linuxfoundation.org/wp-content/uploads/2026/03/sponsor_pytconf26_eu_030526.pdf) and the [Attention Is All You Need](https://arxiv.org/pdf/1706.03762) paper. I chose the brochure because it has diverse elements images, multi-column format, multi column table with rich formatting,  and styled text which is a great way to evaluate docling's performance across different formats and also to test features like table extraction and OCR. 

I also wanted to test the `--enrich formula` feature flag but since the brochure has no mathematical formulas i used thr **Attention Is All You Need** research paper for that. 

### Errors Encountered ###

During the course of this experiment when I was testing the `--force OCR` flag on the Pytorch Conference brochure i encountered a **memory allocation failure**  from Pytorch because my system's available RAM + virtual memory was exhausted at that moment. 
![docling-error](https://github.com/user-attachments/assets/31efdbcd-d5fe-4a06-a820-4e864904905b)

I figured out the following reasons for this
- Docling's pipeline with `--force-ocr` is quite memory intensive because it loads heavy models for document understanding, table structure, etc.
- Force OCR processes every page as images, which spikes memory usage.
- A 7-page brochure rich in images/graphics can still trigger high peak usage.

I tried to manage this by closing all other tabs and applications and using pypdfium2 backend which is much lighter than the default. 
```python
docling documents/docling_brochure.pdf --to md --force-ocr --output outputs/force_ocr/ --pdf-backend pypdfium2
```
I still got the same error, which meant that I had to either downgrade the quality of the PDF being used for the experiments and use something without tables/less rich in images or not test certain flags i wanted to. To avoid this I used Google Colab to do my task so that I can use the brochure and do all the experiments seamlessly.

I have uploaded the notebook with cleanly marked cells to document each experiment along with the source PDFs and all the outputs in this repository.

## Installation ## 
Installed Docling using pip 
![Docling-Installation](https://github.com/user-attachments/assets/52bf34b7-3c14-4819-9c3f-db8ec3da3d1e)

Checked the version
![Docling-version](https://github.com/user-attachments/assets/73e452ee-09a7-4364-8331-124b4896063c)

I also explored the `--help` flag to understand all the commands
![docling-help](https://github.com/user-attachments/assets/cf4d89e4-dd4a-4b45-8ff3-a28da5bf7cb1)

## Converting the PDF into different formats ##

### Baseline(Markdown) Format

```python
%%time
!docling documents/docling_brochure.pdf \
  --output outputs/baseline/

!du -h outputs/baseline/docling_brochure.md
!wc -l outputs/baseline/docling_brochure.md
```
This was the output of the above code
![Baseline-Markdown1](https://github.com/user-attachments/assets/59197b9b-0cb3-4f47-96b2-3c898b180ac5)

**Observations**

- Overall output is clean and readable Markdown.
- Wall time : 2min 26 secs
- CPU time : user 347 ms + sys 56.4 ms = total 403 ms
- Generated file: 1.2 MB, 281 lines
- Table structure is fully preserved. I did not observe any broken cells or merged rows.
- Some minor visual disruption was observed. Some icon images appear slightly misaligned from the original PDF which uses tight icons + text blocks
- 
   ![Baseline-visual](https://github.com/user-attachments/assets/a351d136-1a6b-400f-b3bb-fda339ffaa3a)

### JSON 

What the `--to json` flag does is that instead of serialising to a human-readable format, it serialises the entire internal DoclingDocument object directly to JSON. This is Docling's native data format the code for converting the original PDF to JSON

```python
%%time
!docling documents/docling_brochure.pdf \
  --to json \
  --output outputs/json/

!du -h outputs/json/docling_brochure.json
```

this was the output 

![JSON1](https://github.com/user-attachments/assets/1dea3ffc-60af-4200-b727-5d49843594ee)

JSON Schema 
![JSON-Output](https://github.com/user-attachments/assets/e379bb83-b982-4983-aa8a-342601efaccb)

#### JSON Structure Analysis
I wanted to understand the internal document representation created by Docling and also to verify that the structural elements (headings, tables, images) were correctly detected and separated so I wrote a small analysis script: 
```python
import json

with open("outputs/json/docling_brochure.json") as f:
    doc = json.load(f)

print(f"Top-level keys : {list(doc.keys())}")
print(f"Text blocks    : {len(doc.get('texts', []))}")
print(f"Tables         : {len(doc.get('tables', []))}")
print(f"Pictures       : {len(doc.get('pictures', []))}")
print(f"Pages          : {len(doc.get('pages', {}))}")
```
**The Output**
```bash

Top-level keys : ['schema_name', 'version', 'name', 'origin', 'furniture', 'body', 'groups', 'texts', 'pictures', 'tables', 'key_value_items', 'form_items', 'pages']
Text blocks    : 122
Tables         : 1
Pictures       : 25
Pages          : 7
```

**Obsevations**

- Top-level keys show that Docling uses a comprehensive DoclingDocument schema. Important fields include:

  - `texts` — Individual text items (paragraphs, headings, lists, etc.)
  - `tables` — Structured tables with cell information
  - `pictures` — Detected images and icons
  - `furniture`, `body`, `groups` — Internal layout and hierarchy information
  - `pages` — Page-level metadata

- It correctly identified **1 Table** , **25 Images** and **7 Pages** corresponding to the original brochure
- The hierarchical structure of the document (title → sections → subsections → promotional blocks) is preserved in the JSON schema (with text, level, type, children, etc).
- Wall time: 2 min 7 s
- CPU time: user 270 ms + sys 38.5 ms = total 308 ms
- File size: 6.5 MB
- The file size is significantly larger than all other formats because the JSON contains all image data as base64 and all the structural metadata. It's the most information-dense format.

### PlainText ###

the `--to text` extracts only the raw text content, discarding all structural markup. 

```python
%%time
!docling documents/docling_brochure.pdf \
  --to text \
  --output outputs/text/

!du -h outputs/text/docling_brochure.txt
```

**Output**

![Plaintext1](https://github.com/user-attachments/assets/e93d79fd-a0e0-4f99-8af8-327f0cf0a38a)



this is what the `.txt` file looked like

![PlainText-Output](https://github.com/user-attachments/assets/3d58113a-a990-4f28-9db7-99e055bfc398)

**Observations**
- Wall time: 2 min 5 s
- CPU time: user 281 ms + sys 35.7 ms = total 316 ms
- File size: 28 KB
- The file very small, because it's just characters with no structural or visual data. I also noticed that there were no `<!-- image -->` comments, no base64 data, no image references whatsoever. This is because this format does not export images.


### Doctags ###

DocTags is Docling's own markup language. It is a structured text format that uses XML-like tags to annotate document elements. It was designed specifically for training vision-language models on document understanding tasks.
```python
%%time
!docling documents/docling_brochure.pdf \
  --to doctags \
  --output outputs/doctags/

!du -h outputs/doctags/
!head -40 outputs/doctags/docling_brochure.doctags
```
**Output**


<img width="1191" height="642" alt="image" src="https://github.com/user-attachments/assets/b4468906-6e5c-4d36-bbbd-af90d4af8b9a" />

**Observation**
- Wall time: 2 min 3 s
- CPU time: user 260 ms + sys 43.7 ms = total 304 ms
- Output size: 28 KB
- The Output produced was a lightweight file(28 KB) with semantic structure and document heirarchy well preserved.
- This output looked a lot different than the rest of the documents generated thus far so i wanted to take a deeper look into the output. I found that every piece of content was wraped in small XML-style tags. The location co-ordinates like `<loc_153>` `<loc_210>` tell exactly where that text or image sits on the page.
- DocTags may be useful for RAG applications that require lightweight processing with basic structure and region aware retrieval but for most RAG systems Markdown or JSON format are much easier to work with.

### Image Modes

#### HTML Embedded 

```python
%%time
!docling documents/docling_brochure.pdf \
  --to html \
  --output outputs/html_embedded/

!du -h outputs/html_embedded/docling_brochure.html
```
**Output**

This is what the rendered file looked like

<img width="1032" height="860" alt="image" src="https://github.com/user-attachments/assets/32782a35-21e6-44aa-91d4-cc3ae7563115" />


### HTML Placeholder

```python
%%time
!docling documents/docling_brochure.pdf \
  --to html \
  --image-export-mode placeholder \
  --page-batch-size 1 \
  --pdf-backend pypdfium2 \
  --output outputs/html_placeholder/ 2>&1 | tee outputs/html_placeholder/log.txt

!du -h outputs/html_placeholder/docling_brochure.html
```


**Output**

This is what the rendered file looked like

<img width="1032" height="843" alt="image" src="https://github.com/user-attachments/assets/12cd689e-1743-4379-aa17-395d9748c111" />

#### Image Mode Size Comparision

I wrote some code to compare both the image handling formats
```python
emb = os.path.getsize("outputs/html_embedded/docling_brochure.html")
ph  = os.path.getsize("outputs/html_placeholder/docling_brochure.html")

print(f"Embedded    : {emb:,} bytes ({emb/1024:.1f} KB)")
print(f"Placeholder : {ph:,} bytes ({ph/1024:.1f} KB)")
print(f"\nEmbedded is {emb/ph:.0f}x larger than placeholder")
```

**Output**

```bash
Embedded    : 1,178,502 bytes (1150.9 KB)
Placeholder : 18,286 bytes (17.9 KB)

Embedded is 64x larger than placeholder
```

**Observation**

**1. HTML Embedded**

  - Maintains good visual layout & headings and keeps all icons and images visible. 
  - Extremely large file size (64× bigger) because every image is converted to base64 strings.
  - These base64 blocks create a lot of noise when chunking and embedding 
  - This format may be helpful when using multimodal RAG with vision models.

**2. HTML Placeholder**

  - The output file is very small and lightweight (18 KB).
  - It has clean text and structure with minimal noise as it removes visual clutter that doesn't add value to text-based RAG systems which means that content of this format is easy to chunk.
  - This format is useful for text-based RAG systems that involve searching over documents and question answering.

## Table Structure Experiments

Docling has 2 table flags `--table-mode accurate` and `--table-mode fast` . After the layout model identifies a table region, a second model called *TableFormer* reconstructs the row boundaries, column separators, and content mapping. 

### Accurate Mode 

```python
%%time
!docling documents/docling_brochure.pdf \
  --to md \
  --table-mode accurate \
  --output outputs/table_accurate/

!wc -l outputs/table_accurate/docling_brochure.md
```

**Output**

<img width="897" height="700" alt="image" src="https://github.com/user-attachments/assets/fc681433-d440-407f-8539-8efa2610b5d0" />



### Fast Mode

```python
%%time
!docling documents/docling_brochure.pdf \
  --to md \
  --table-mode fast \
  --output outputs/table_fast/

!wc -l outputs/table_fast/docling_brochure.md
```

**Output**

`--fast` flag generated a table with several inconsistencies. Here are a few images of the table generated that had merged cells and the content of one cell bleeding into the next cell.


<img width="627" height="700" alt="image" src="https://github.com/user-attachments/assets/f1c4aa28-d844-4ab1-ab85-64cd9f02c3a1" />



<img width="895" height="650" alt="image" src="https://github.com/user-attachments/assets/0c185279-e4c3-4ed1-8857-31efa2034605" />



### Table Corruption Proof

Both runs completed without errors or warnings. Both output files are similar in size. The corruption only became visible when I inspected specific rows directly. Here is the code i wrote to explicity demonstrate the table corruption in `--fast` mode

```python
!echo "=== ACCURATE ==="
!grep -n "Social Media\|Attendee Registration\|Sponsorship Cost" \
  outputs/table_accurate/docling_brochure.md

!echo "=== FAST ==="
!grep -n "Social Media\|Attendee Registration\|Sponsorship Cost" \
  outputs/table_fast/docling_brochure.md
```

**Output**

```bash

=== ACCURATE ===
146:| Attendee Registration Contact List: Opt-in only                                                                                                                                                                                                              | ✔ (List provided pre and post event)                                                   | ✔ (List provided post event)                                                           |                                                                                        |                                                                                        |                                                                                        |                                                                                        |
147:| Social Media Promotion: From PyTorch X handle. All custom posts must be approved by the PyTorch Foundation.                                                                                                                                                  | 1 Custom Post, 1 Group Post, and 1 Re-Post                                             | 1 Group Post and 1 Re-Post                                                             | 1 Group Post                                                                           |                                                                                        |                                                                                        |                                                                                        |
160:| Sponsorship Cost                                                                                                                                                                                                                                             | $50,000                                                                                | $35,000                                                                                | $18,000                                                                                | $8,000                                                                                 | $4,000                                                                                 | $4,000                                                                                 |
=== FAST ===
147:| Attendee Registration Contact List: Opt-in only Social Media Promotion: From PyTorch X handle. All custom                                                                         | ✔ (List provided pre and post event) 1 Custom Post, 1 Group Post, and                  | ✔ (List provided post event) 1 Group Post and                                          | 1 Group Post                                                                           |                                                                                        |                                                                                   |                                                                                   |
161:| Sponsorship Cost                                                                                                                                                                  |                                                                                        |                                                                                        |                                                                                        |                                                                                        |                                                                                   |                                                                                   |



```


### Analysis

**Table Mode: --table-mode accurate vs --table-mode fast**

**Performance Results:**

| Mode | Wall Time | Line Count |
|---|---|---|
| Accurate | 2m 3s | 281 |
| Fast | 1m 53s | 283 |

Fast mode is 10 seconds faster but it also generates 2 extra lines that are phantom rows created by cell overflow.

**Three distinct failures:**

**Failure 1** — Row merger: Lines 146 and 147 in accurate mode are two separate rows. 
In fast mode they collapse into one, both row descriptions were concatenated inside a single cell 
with no separator.

**Failure 2** — Content overflow: The Diamond tier cell in the merged row contains values 
from both rows joined together, cut off mid-sentence. 1 Re-Post disappeared entirely.

**Failure 3** — Pricing row emptied: The Sponsorship Cost row exists in both outputs but 
in fast mode every pricing cell is empty. $50,000, $35,000, $18,000, $8,000, $4,000, 
$4,000 are all absent. The values were displaced into phantom rows that have no corresponding 
real row in the document.

### RAG Implication

A RAG system querying "What is the cost of a Diamond sponsorship?" would retrieve the Sponsorship Cost row from fast mode output and find empty cells. It would either return nothing or hallucinate a value from training data. It silently corrupts the table, no errors were raised at any stage.

`--table-mode fast` processes the documents in lesser time but it is less accurate. It produces structurally invalid output for complex tables. For documents with large, complex tables that carry the important knowledge `--table-mode accurate` is the right choice.


## OCR Related Experiments

Docling's OCR engine(Rapid OCR) handles text extraction from image based regions. Two flags control it's OCR behaviour

- `--no-ocr` - disables ocr entirely, text comes only from the PDF's embedded text layer
- `--force-ocr` - forces OCR on every region of every page regardless of whether embedded text exists

### `--no-ocr` 

```python
%%time
!docling documents/docling_brochure.pdf \
  --to md \
  --no-ocr \
  --output outputs/no_ocr/

!du -h outputs/no_ocr/docling_brochure.md
!wc -l outputs/no_ocr/docling_brochure.md
```

**Output**

- Wall time: 1minute 9 seconds
- Output is 1.2MB with 281 lines identical to the baseline which means that no extra lines or phantom rows were added.
- The table generated in the rendered file was clean and retained original structure.

### `--force-ocr`

```python
%%time
!docling documents/docling_brochure.pdf \
  --to md \
  --force-ocr \
  --profiling \
  --output outputs/force_ocr/ 2>&1 | tee outputs/force_ocr/log.txt
```

**Output**

- Wall time : 14 minutes and 37 seconds
- although the table structure was retained but it was not able to detect the check marks in the third last and second last rows
- Not only  `--force-ocr` took much longer to process the document, it also degraded the quality of the document.

<img width="940" height="427" alt="Screenshot 2026-04-10 193703" src="https://github.com/user-attachments/assets/3782141f-9076-4d30-8817-cc0373ebd58a" />

### OCR Profiling

After running `--force-ocr`, I wanted to understand exactly where the 14 minutes and 37 seconds went and which stage is responsible for this. The `--profiling` flag breaks down pipeline execution time per stage. Here is the code I wrote for it.

```python
import re

with open("outputs/force_ocr/log.txt") as f:
    raw = f.read()

clean = re.sub(r'\033\[[0-9;]*m', '', raw)
for line in clean.split('\n'):
    if line.strip():
        print(line)
```

This is what the output looked like 

![OCR-Profiling1](https://github.com/user-attachments/assets/a1d04b96-f828-4152-8e84-a96a5dc70b0c)


**Reading the profiling table:**

| Stage | Total Time | % of Runtime |
|---|---|---|
| page_preprocessing | ~7.6s | 0.9% |
| **ocr** | **~846s** | **98.8%** |
| layout | ~34s | 4.0% |
| table_structure | ~41s | 4.8% |
| doc_assemble | ~2s | 0.2% |
| **Total pipeline** | **~856s** | **100%** |

OCR consumed 846 out of 856 seconds - **98.8% of total runtime**. Every other stage combined - layout analysis, table structure recognition, page preprocessing, document assembly took 10 seconds. OCR on 7 pages consumed the other 846.

This happened because `--force-ocr` bypasses the embedded text layer and runs three neural network models on every page region sequentially.


### Observation

**One peculiar thing i observed for both the OCR modes was that completly miss the footnotes and small text. Both the flags missed the startups, startup sponsorships and non-profit sponsorships footnote as seen in the original brochure below.**


<img width="916" height="171" alt="Screenshot 2026-04-10 193212" src="https://github.com/user-attachments/assets/1befb642-ce52-4257-9f59-536ccfa3d930" />


## Enrich formula flag

To test this flag I used the 'Attention Is All You Need' paper. 

What the `--enrich-formula` flag does internally is that after the standard pipeline extracts text, this flag activates an additional model pass over regions classified as mathematical formulas. The model is trained to interpret mathematical notions and output in a structured format like LaTeX syntax or MathML. 

This is required for PDFs rich in scientific notations and mathematical formulas because PDF stores math as arbitrary positioned characters. for example, the integral sign, the fraction bar and variables stored as seperate positioned glyphs with no inherent relationship. The enrichment model reconstructs the mathematical meaning from the spatial arrangement of glyphs

```python
%%time
!docling documents/Transformer_Paper.pdf \
  --to md \
  --enrich-formula \
  --output outputs/enrich_formula/
```

### Without `--Enrich-formula`

Here is the baseline paper without `--enrich code`. The **Attention formula** is garbled PDF glyph positions were extracted as a flat sequence of characters without preserving the structural relation between them.  


![Transformer-baseline](https://github.com/user-attachments/assets/b76c3372-3fbd-4132-907e-b145f6c4423e)



### With `--Enrich-formula`

 Transformer paper with `--enrich-code`. The **Attention formula** is LaTeX formatted which an LLM can parse correctly and a Markdown renderer can display properly.

 ![Transformer-enriched](https://github.com/user-attachments/assets/ac6386c5-4569-4ec7-bc73-17629f672cbd)




















- 
























  








