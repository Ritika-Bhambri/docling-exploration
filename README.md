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
























  








