# UN Ocean Commitments: Dataset & Extraction Methodology

## Overview
This repository contains a fully extracted, enriched, and locally backed-up dataset of the **United Nations Ocean Commitments** database (originally hosted at `sdgs.un.org/partnerships/action-networks/ocean-commitments`). 

The final dataset contains **2,650 unique commitments**, saved in `final_ocean_commitments_enriched.json`.

Because the official UN database utilizes volatile, non-deterministic sorting and unstable deep-pagination, standard linear scraping techniques fail to capture the complete dataset. This project utilizes a highly resilient, multi-stage "Pincer Movement" architecture to outmaneuver the server limits, trap shifting data, and decouple network fetching from HTML parsing.

---

## The Challenge: Shifting Sands & Ghost Pages
Scraping the UN Ocean Commitments registry presented two severe technical hurdles:

1. **Server Timeouts on Deep Pagination:** The database reliably serves the first ~100 pages of results but begins to critically stall, timeout, or return empty "ghost pages" when paginating deep into the dataset (Pages 120-148).
2. **Non-Deterministic Sorting (Page Drift):** Hundreds of commitments share identical tie-breaking metadata (e.g., upload timestamps). When queried sequentially from Page 1 to 148, the SQL database sorts them in one specific order. When queried in reverse (from Page 148 to 1), the tie-breakers flip. 
    * *Diagnostic testing revealed an average "Page Drift" of 30 pages per item, with some commitments jumping up to 68 pages depending on the direction of the crawl.* * A standard one-way scraper would permanently miss nearly 10% of the dataset as items "drifted" out of its path.

---

## Methodology: The 4-Phase Extraction Pipeline

To guarantee 100% data extraction without corruption, the architecture was split into four distinct phases:

### Phase 1: URL Discovery (The "Pincer Movement")
Instead of scraping details directly from the listing pages, we deployed a Playwright-based click-through scraper solely to harvest unique URLs. To counter the "Page Drift," we ran two separate instances:
* **The Forward Crawl:** Started at Page 1 and clicked "Next" until the server became unstable (Page 118).
* **The Reverse Crawl:** Jumped directly to Page 148 via URL injection, anchored the database state, and clicked "Previous" counting backward to Page 1. 

### Phase 2: Overlap Analysis & Merging
By analyzing the clean portions of both runs, we identified a massive "Safe Zone" overlap (Pages 78 through 118). Combining the unique URLs from both the Forward and Reverse runs successfully trapped the drifting items, resulting in a master list of **2,650 unique commitment URLs**.

### Phase 3: Decoupled Detail Scraping & Offline Archiving
With the master URL list secured, we abandoned Playwright's heavy browser automation. 
Using a lightweight, multi-threaded `requests` engine (`scrape_details_only.py`), we fetched the detail pages for all 2,650 URLs. 
* **The Offline Archive:** Every single fetched page was saved as a raw `.html` file to a local directory (`local_html_archive`). This decoupled the network latency from the parsing logic and protected the dataset against future UN server outages.

### Phase 4: Local HTML Enrichment
With the entire database safely on the local hard drive, a dedicated parsing script (`enrich_local_data.py`) was used to extract complex, highly-variable fields (like visually tagged SDGs and multi-line Entity headers) using `BeautifulSoup`. Because this ran locally, parsing all 2,650 files took seconds rather than hours, allowing for rapid iteration and bug fixing.

---

## Data Dictionary

The final output is `final_ocean_commitments_enriched.json`. It contains an array of 2,650 JSON objects.

| Field | Type | Description |
| :--- | :--- | :--- |
| `url` | String | The canonical UN URL for the commitment. |
| `ocean_action_id` | String | The unique UN identifier (e.g., `#OceanAction41008`). |
| `title` | String | The official title of the commitment. |
| `description` | String | The full text description of the project/commitment. |
| `sdg14_targets` | Array [Str] | Specific SDG 14 sub-targets addressed (e.g., `["14.2", "14.a"]`). |
| `entity` | String | The leading government, NGO, or organization responsible. |
| `ocean_basins` | Array [Str] | Geographic regions targeted (e.g., `["South Pacific", "Global"]`). |
| `sdgs` | Array [Str] | All SDGs connected to the project, extracted via text, icons, and links (e.g., `["1", "13", "14"]`). |

---

## Repository Structure

* **`final_ocean_commitments_enriched.json`**: The final, master dataset.
* **`local_html_archive/`**: Directory containing 2,650 raw HTML files. Files are named using the URL slug and an MD5 hash (e.g., `commitment-name_a1b2c3d4e5.html`).
* **`scrape_ocean_commitments_v13.py`**: The Playwright Pincer Movement script (handles both `--forward` and `--reverse` listing-page navigation).
* **`scrape_details_only.py`**: The pure `requests` scraper that reads a list of URLs, fetches the HTML, and saves the local archive.
* **`enrich_local_data.py`**: The offline `BeautifulSoup` parser that reads the local HTML archive and injects complex fields (Entity, Ocean Basins, SDGs) into the JSON.
* **`analyze_commitments.py`**: A lightweight analytics script to generate terminal reports on the most common Entities, Basins, and SDGs.