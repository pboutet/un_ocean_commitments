
# UN Ocean Conference (UNOC) Master Commitments Dataset

**Last Updated:** March 27, 2026

## 📖 Overview
This dataset contains a comprehensive, enriched list of Voluntary Commitments made towards the implementation of Sustainable Development Goal (SDG) 14 (Life Below Water). 

The primary goal of this dataset is to merge raw scraped commitment data with the official **"Compiled VC list since UNOC 2022_20250904"** document, resulting in a clean, highly structured, and machine-readable Excel file. The data has been enriched with language detection, duplicate flagging, and one-hot encoding for complex list-based attributes to enable immediate pivot-table and dashboard analysis.

## 📁 File Manifest
* **Final Excel Dataset:** `final_ocean_commitments_encoded.xlsx` (Use this for Excel/Tableau/PowerBI analysis)
* **Final JSON Dataset:** `final_ocean_commitments_enriched_MDtagged_languageTagged_dupsTagged.json` (Use this for programmatic access; contains native nested lists instead of one-hot encoded columns)
* **Compiled Markdown document of priority UNOC 2025 commitments:** `/Users/pboutet/Documents/github_repos/vc_data_split/render_scraping_merging/Compiled VC list since UNOC 2022_20250904.docx.md`
---

## 🛠️ Data Pipeline & History
This dataset was generated through a multi-step Python pipeline designed to enrich and clean the raw data:

1. **Markdown Cross-Referencing:** * The raw JSON data was cross-referenced against a compiled Markdown document of priority UNOC 2025 commitments.
   * **Matching Logic:** Records were primarily matched using the `#OceanActionXXXXX` ID. 
   * **Orphan Rescue:** If an ID was missing in the Markdown file, the pipeline attempted a secondary "Text Rescue" by searching the Markdown text for the exact title of the commitment.
2. **Language Tagging:** * The `lingua-language-detector` library was used to scan the combined title and description of every commitment. The dataset features over 20 distinct languages, with English, Spanish, and French being the most prominent.
3. **Duplicate Tagging (No-Drop Policy):** * To preserve valuable context (e.g., multi-phase projects or joint commitments by different entities), duplicates were **not deleted**.
   * Instead, case-insensitive checks were run on Titles and Descriptions. Records that share identical text were flagged using `has_duplicate_title` and `has_duplicate_description`.
4. **Data Cleaning & One-Hot Encoding:**
   * **Artifact Removal:** Spurious web-scraping artifacts in the ocean basins field (e.g., "N/A" and "Website/More information") were scrubbed.
   * **Encoding:** Python lists for SDG 14 Targets, Ocean Basins, and general SDGs were exploded into binary "One-Hot" indicator columns (0 or 1). This flattens the data, making it natively compatible with Excel filters and pivot tables.
5. **Excel Sanitation:**
   * During final export, hidden unprintable control characters (common in scraped, multi-lingual text) were systematically stripped from the descriptions to prevent Excel file corruption.

---

## 📊 Data Dictionary (Column Definitions)

### 1. Core Metadata
* **`url`**: The source link to the commitment on the UN SDGs platform.
* **`ocean_action_id`**: The unique identifier for the commitment (e.g., `#OceanAction41142`).
* **`title`**: The official name of the voluntary commitment.
* **`description`**: A detailed summary of the project goals, impact, and methodology.
* **`entity`**: The primary organization, government, or group leading the commitment.

### 2. Pipeline Enrichment & Validation Tags
* **`in_markdown_doc`**: (`True`/`False`) Indicates whether this commitment was found in the official UNOC 2025 compiled Markdown document.
* **`match_method`**: Explains how the record was found in the document (`ID Match` or `Text Rescue (No ID in MD)`).
* **`md_bullet_number`**: The exact numbered bullet point where this commitment appears in the original compiled document.
* **`md_title_verified`**: (`True`/`False`) A secondary validation confirming that the JSON `title` was found verbatim inside the matched Markdown bullet text.
* **`md_title_mismatch_warning`**: Contains a warning message if the Ocean Action ID matched, but the title in the document was written differently than the title in the database.
* **`language`**: The primary language of the commitment detected by the NLP model (e.g., `ENGLISH`, `SPANISH`, `FRENCH`).
* **`has_duplicate_title`**: (`True`/`False`) Flags if another commitment in this dataset shares the exact same title (case-insensitive).
* **`has_duplicate_description`**: (`True`/`False`) Flags if another commitment in this dataset shares the exact same description text.

### 3. SDG 14 Targets (One-Hot Encoded)
*Indicates which specific targets of SDG 14 (Life Below Water) the commitment addresses. Columns contain a `1` if applicable, `0` otherwise.*
* **`target_14.1`**: Prevent and significantly reduce marine pollution.
* **`target_14.2`**: Sustainably manage and protect marine and coastal ecosystems.
* **`target_14.3`**: Minimize and address the impacts of ocean acidification.
* **`target_14.4`**: Effectively regulate harvesting and end overfishing, IUU fishing, etc.
* **`target_14.5`**: Conserve at least 10 per cent of coastal and marine areas.
* **`target_14.6`**: Prohibit certain forms of fisheries subsidies.
* **`target_14.7`**: Increase the economic benefits to SIDS and LDCs.
* **`target_14.a`**: Increase scientific knowledge, develop research capacity, and transfer marine technology.
* **`target_14.b`**: Provide access for small-scale artisanal fishers to marine resources and markets.
* **`target_14.c`**: Enhance the conservation and sustainable use of oceans and their resources by implementing international law (UNCLOS).

### 4. Ocean Basins (One-Hot Encoded)
*Geographic focus areas of the commitment. Columns contain a `1` if applicable, `0` otherwise.*
* **`basin_Global`**
* **`basin_North Atlantic`**
* **`basin_South Atlantic`**
* **`basin_North Pacific`**
* **`basin_South Pacific`**
* **`basin_Indian Ocean`**
* **`basin_Arctic Ocean`**
* **`basin_Southern Ocean`**

### 5. Sustainable Development Goals (One-Hot Encoded)
*Indicates intersecting SDGs beyond SDG 14. Columns contain a `1` if applicable, `0` otherwise.*
* **`sdg_1`** through **`sdg_17`**: Represents the 17 UN Sustainable Development Goals (e.g., `sdg_13` is Climate Action, `sdg_5` is Gender Equality).

---
## 💡 Notes for Analysts
* **Filtering for the UNOC 2025 List:** To view only the highly curated list of commitments compiled for the recent conference, filter the `in_markdown_doc` column to `True`.
* **Analyzing Duplicates:** If `has_duplicate_title` is `True`, check the `entity` column. Often, this indicates multiple organizations signing onto a single global initiative (e.g., "UN Global Compact Sustainable Ocean Principles").
* **Target Summation:** Because the targets, basins, and SDGs are one-hot encoded, you can easily find the total number of commitments addressing a specific area by simply summing the respective column in Excel.