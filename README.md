# UN Ocean Conference (UNOC) Commitments: End-to-End Extraction & Enrichment Pipeline

**Last Updated:** March 27, 2026

## 📖 Overview
This repository contains a fully extracted, enriched, and locally backed-up dataset of the **United Nations Ocean Commitments** database (originally hosted at `sdgs.un.org/partnerships/action-networks/ocean-commitments`) as of March 26, 2026. 

The primary goal of this project was to capture the complete volatile UN database, merge it with the official **"Compiled VC list since UNOC 2022_20250904"** priority document, and produce a clean, highly structured, and machine-readable Excel file. The final dataset contains **2,650 unique commitments** and has been enriched with language detection, duplicate flagging, and one-hot encoding for immediate pivot-table and dashboard analysis.

---

## 📁 Repository Structure & File Manifest

### Final Data Outputs
* **`final_ocean_commitments_encoded.xlsx`**: The master Excel dataset. (Use this for Excel/Tableau/PowerBI analysis; features one-hot encoded indicator columns).
* **`final_ocean_commitments_enriched_MDtagged_languageTagged_dupsTagged.json`**: The master JSON dataset. (Use this for programmatic access; contains native nested arrays instead of one-hot encoded columns).
* **`Compiled VC list since UNOC 2022_20250904.docx.md`**: Compiled Markdown document of priority UNOC 2025 commitments. 

### Local Archives & Scripts
* **`local_html_archive/`**: Directory containing 2,650 raw HTML files. Files are named using the URL slug and an MD5 hash (e.g., `commitment-name_a1b2c3d4e5.html`) to protect against UN server outages.
* **`scrape_ocean_commitments_v13.py`**: The Playwright "Pincer Movement" script (handles both `--forward` and `--reverse` listing-page navigation).
* **`scrape_details_only.py`**: The pure `requests` scraper that fetches the HTML and saves the local archive.
* **`enrich_local_data.py`**: The offline `BeautifulSoup` parser that injects complex fields into the JSON.
* **`analyze_commitments.py`**: A lightweight analytics script to generate terminal reports.

---

## 🏗️ Part 1: Data Extraction Methodology

Because the official UN database utilizes volatile, non-deterministic sorting and unstable deep-pagination, standard linear scraping techniques fail to capture the complete dataset. This project utilizes a highly resilient, multi-stage "Pincer Movement" architecture.

### The Scraping Challenge: Shifting Sands & Ghost Pages
1. **Server Timeouts on Deep Pagination:** The database reliably serves the first ~100 pages of results but begins to critically stall or return empty "ghost pages" when paginating deep into the dataset (Pages 120-148).
2. **Non-Deterministic Sorting (Page Drift):** Hundreds of commitments share identical tie-breaking metadata. When queried sequentially from Page 1 to 148, the SQL database sorts them in one specific order. When queried in reverse, the tie-breakers flip. An average "Page Drift" of 30 pages per item meant a standard one-way scraper would permanently miss nearly 10% of the dataset.

### Phase 1: URL Discovery (The "Pincer Movement")
To counter "Page Drift," we ran two separate Playwright instances solely to harvest unique URLs:
* **The Forward Crawl:** Started at Page 1 and clicked "Next" until the server became unstable.
* **The Reverse Crawl:** Jumped directly to Page 148 via URL injection and clicked "Previous" counting backward to Page 1. 

### Phase 2: Overlap Analysis & Merging
By analyzing the clean portions of both runs, we identified a massive "Safe Zone" overlap. Combining the unique URLs successfully trapped the drifting items, resulting in a master list of **2,650 unique URLs**.

### Phase 3: Decoupled Detail Scraping & Offline Archiving
Using a lightweight, multi-threaded `requests` engine, we fetched the detail pages for all 2,650 URLs and saved them as raw `.html` files to a local directory. This decoupled the network latency from the parsing logic.

### Phase 4: Local HTML Parsing
A dedicated script was used to extract complex, highly-variable fields (like visually tagged SDGs and multi-line Entity headers) using `BeautifulSoup` directly from the local hard drive, allowing for rapid iteration.

---

## 🛠️ Part 2: Data Enrichment & Formatting

Once the raw JSON was secured, a Python pipeline was deployed to clean, enrich, and format the data for enterprise analysis.

### Phase 5: Markdown Cross-Referencing
* The raw JSON data was cross-referenced against a compiled Markdown document of priority UNOC 2025 commitments.
* **Matching Logic:** Records were primarily matched using the `#OceanActionXXXXX` ID. 
* **Orphan Rescue:** If an ID was missing in the Markdown file, the pipeline attempted a secondary "Text Rescue" by searching the Markdown text for the exact title of the commitment.

### Phase 6: Language Tagging
* The `lingua-language-detector` library was used to scan the combined title and description of every commitment. The dataset features over 20 distinct languages.

### Phase 7: Duplicate Tagging (No-Drop Policy)
* To preserve valuable context (e.g., multi-phase projects or joint commitments by different entities), duplicates were **not deleted**.
* Case-insensitive checks were run on Titles and Descriptions. Records sharing identical text were flagged using boolean columns.

### Phase 8: Data Cleaning & One-Hot Encoding
* **Artifact Removal:** Spurious web-scraping artifacts in the ocean basins field (e.g., "N/A" and "Website/More information") were scrubbed.
* **Encoding:** Python arrays for SDG 14 Targets, Ocean Basins, and general SDGs were exploded into binary "One-Hot" indicator columns (0 or 1), making the data natively compatible with Excel pivot tables.

### Phase 9: Excel Sanitation
* Hidden unprintable control characters (common in scraped, multi-lingual text) were systematically stripped from the descriptions to prevent Excel file corruption during the final export.

---

## 📊 Data Dictionary (Column Definitions)

### 1. Core Metadata
| Field | Type | Description |
| :--- | :--- | :--- |
| `url` | String | The canonical UN URL for the commitment. |
| `ocean_action_id` | String | The unique identifier for the commitment (e.g., `#OceanAction41142`). |
| `title` | String | The official name of the voluntary commitment. |
| `description` | String | A detailed summary of the project goals, impact, and methodology. |
| `entity` | String | The primary organization, government, or group leading the commitment. |

### 2. Pipeline Enrichment & Validation Tags
| Field | Type | Description |
| :--- | :--- | :--- |
| `in_markdown_doc` | Boolean | Indicates whether this commitment was found in the official UNOC 2025 compiled Markdown document. |
| `match_method` | String | Explains how the record was found in the document (`ID Match` or `Text Rescue (No ID in MD)`). |
| `md_bullet_number` | String/Int | The exact numbered bullet point where this commitment appears in the original compiled document. |
| `md_title_verified` | Boolean | A validation confirming that the JSON `title` was found verbatim inside the matched Markdown bullet text. |
| `md_title_mismatch_warning` | String | Warning message if the Ocean Action ID matched, but the title in the document was written differently than the title in the database. |
| `language` | String | The primary language of the commitment detected by the NLP model (e.g., `ENGLISH`, `SPANISH`, `FRENCH`). |
| `has_duplicate_title` | Boolean | Flags if another commitment in this dataset shares the exact same title (case-insensitive). |
| `has_duplicate_description`| Boolean | Flags if another commitment in this dataset shares the exact same description text. |

### 3. Indicator Columns (One-Hot Encoded in Excel)
*Note: In the JSON file, these remain stored as native nested Arrays `[]`.*

* **SDG 14 Targets (`target_14.1` to `target_14.c`)**: Indicates which specific sub-targets of SDG 14 the commitment addresses (e.g., `target_14.3` for Ocean Acidification). Columns contain a `1` if applicable, `0` otherwise.
* **Ocean Basins (`basin_Global`, `basin_North Atlantic`, etc.)**: Geographic focus areas of the commitment. Columns contain a `1` if applicable, `0` otherwise.
* **Sustainable Development Goals (`sdg_1` to `sdg_17`)**: Indicates intersecting SDGs beyond SDG 14 (e.g., `sdg_13` is Climate Action). Columns contain a `1` if applicable, `0` otherwise.

---

## 💡 Notes for Analysts
* **Filtering for the UNOC 2025 List:** To view only the highly curated list of commitments compiled for the recent conference, filter the `in_markdown_doc` column to `True`.
* **Analyzing Duplicates:** If `has_duplicate_title` is `True`, check the `entity` column. Often, this indicates multiple organizations signing onto a single global initiative (e.g., "UN Global Compact Sustainable Ocean Principles").
* **Target Summation:** Because the targets, basins, and SDGs are one-hot encoded in the Excel file, you can easily find the total number of commitments addressing a specific area by simply summing the respective column.