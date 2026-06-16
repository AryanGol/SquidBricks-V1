<p align="center">
  <img src="SquidBricks-Copy.png" alt="SquidBricks logo" width="520">
</p>

<h1 align="center">SquidBricks: Open-Access Biomedical Literature Screening</h1>

<p align="center">
  A Google Colab workflow for searching PubMed Central, keyword-screening evidence, reranking candidate paragraphs, and using an LLM to classify relevant PMC articles.
</p>

---

## Overview

This repository contains the notebook:

```text
open_access_screening_final.ipynb
```

The notebook is designed for semi-automated biomedical literature screening. It starts with a PubMed Central (PMC) query, PMC URL, or a list of PMCIDs/PMIDs. It then downloads open-access BioC XML files, parses the full text into passages, filters passages by section and keyword terms, ranks the most relevant text chunks, and uses an LLM classifier to identify relevant articles.

The workflow is split into two phases:

1. **CPU phase**: search, download, parse, section filtering, keyword filtering, and checkpoint creation.
2. **GPU phase**: restore checkpoint, tokenize, chunk, rerank, group chunks, run LLM classification, and export results.

---

## What the system does

SquidBricks helps users screen open-access biomedical articles by building a staged evidence pipeline:

1. Searches or accepts article IDs from PubMed Central.
2. Downloads open-access articles in BioC XML format.
3. Parses article text into structured passages.
4. Lets the user choose which article sections to keep.
5. Filters passages using user-provided keywords or phrases.
6. Saves a checkpoint before switching from CPU to GPU.
7. Tokenizes and chunks candidate passages safely for model input limits.
8. Reranks chunks using MedCPT or another CrossEncoder.
9. Groups high-scoring chunks by article and token budget.
10. Uses an LLM to classify whether each chunk contains the evidence requested by the user.
11. Removes remaining chunks from articles that have already been marked relevant.
12. Saves final outputs, including relevant PMC IDs and audit files.

---

## Requirements

The notebook is intended to run in **Google Colab**.

### CPU-phase packages

The notebook installs:

```bash
pandas
requests
tqdm
openpyxl
ipywidgets
numpy
```

### GPU-phase packages

After switching to GPU, the notebook installs:

```bash
transformers
sentence-transformers
accelerate
sentencepiece
```

### Optional credentials

You may use the workflow without paid API access if you choose the free LongT5 option. Optional credentials can improve speed or allow external LLM classification.

| Credential | Required? | Purpose |
|---|---:|---|
| NCBI email | Yes | Identifies your requests to NCBI services. |
| NCBI API key | No | Reduces the request delay from 0.35 seconds to 0.12 seconds. |
| OpenAI API key | Optional | Used only if the OpenAI classifier option is selected. |
| Gemini API key | Optional | Used only if the Gemini classifier option is selected. |
| DeepSeek API key | Optional | Used only if the DeepSeek classifier option is selected. |

---

## Quick start

1. Open `open_access_screening_final.ipynb` in Google Colab.
2. Make sure the runtime is set to **CPU** at the beginning.
3. Run the CPU-phase cells from Step 1 through Step 10.
4. Save the CPU checkpoint to Google Drive or download it.
5. Change the Colab runtime to **GPU**.
6. Run the GPU setup cells.
7. Restore the CPU checkpoint.
8. Run reranking, grouping, LLM review, and final export cells.

---

## Folder structure

The notebook creates a project folder under `/content`:

```text
/content/open_access_screening/
├── downloaded_articles/
│   └── PMC*.xml
├── output/
│   ├── *.csv
│   ├── medcpt_score_groups/
│   ├── medcpt_score_subgroups/
│   ├── medcpt_score_subgroups_pending/
│   └── llm_results/
└── src/
```

The exact project folder name can be changed in Step 3 of the notebook.

---

## Step-by-step workflow and outputs

### Step 1. Switch runtime to CPU and install CPU packages

**User action:**  
Set the Colab runtime to CPU or no hardware accelerator, then run the install cell.

**What the system does:**  
Installs lightweight packages needed for searching, downloading, parsing XML, using widgets, and saving tabular outputs.

**Output files:**  
No output files are created in this step.

---

### Step 2. Load interface helpers

**User action:**  
Run the helper cell.

**What the system does:**  
Loads Python libraries, display helpers, styled status cards, metric cards, warning cards, and utility functions used throughout the notebook.

**Output files:**  
No output files are created in this step.

---

### Step 3. Set project folders

**User action:**  
Choose the project folder name, or keep the default:

```text
open_access_screening
```

**What the system does:**  
Creates the main working folders:

```text
/content/open_access_screening/
content/open_access_screening/downloaded_articles/
content/open_access_screening/output/
content/open_access_screening/src/
```

**Output files:**  
No CSV output is created yet, but the folder structure is created.

---

### Step 4. Open PMC and provide search input

**User action:**  
Choose one input mode:

1. Paste a PMC search query.
2. Paste a PMC search-results URL.
3. Paste PMCIDs or PMIDs.

**What the system does:**  
The system stores the exact user input and prepares it for the download step. If a PMC URL is pasted, it extracts the search query from the URL. If IDs are pasted, it separates PMCIDs from PMIDs.

**Output file:**

```text
output/pmc_search_input_config.csv
```

**What this file contains:**  
The selected input mode, the final PMC query if applicable, counts of provided IDs if applicable, and a preview of the pasted input.

---

### Step 5. Enter NCBI settings

**User action:**  
Enter an email address and optionally provide an NCBI API key.

**What the system does:**  
Stores NCBI request settings and sets a safe request delay. With an API key, the delay is shorter.

**Output files:**  
No output files are created in this step.

---

### Step 6. Count PMC results and download BioC XML

**User action:**  
Confirm whether to download the articles and optionally choose how many records to download.

**What the system does:**  

If a query or URL was provided, the system:

1. Counts matching PMC records.
2. Fetches PMC IDs.
3. Downloads available BioC XML files.

If PMIDs were provided, the system first converts them to PMCIDs when possible.

Downloaded XML files are saved in:

```text
downloaded_articles/
```

**Output files:**

```text
output/pmc_search_summary.csv
output/pmc_search_pmcids.csv
output/bioc_download_status.csv
output/bioc_download_status_with_search_info.csv
output/bioc_unavailable_or_failed_articles.csv
output/provided_pmid_to_pmcid_conversion.csv
downloaded_articles/PMC*.xml
```

**What these files contain:**

| File | Description |
|---|---|
| `pmc_search_summary.csv` | Search mode, query, result count, and query translation, or ID conversion summary. |
| `pmc_search_pmcids.csv` | PMCIDs selected for download. |
| `bioc_download_status.csv` | Download status for each PMCID. |
| `bioc_download_status_with_search_info.csv` | Download status merged with search/source information. |
| `bioc_unavailable_or_failed_articles.csv` | Articles that could not be downloaded as open-access BioC XML. |
| `provided_pmid_to_pmcid_conversion.csv` | PMID-to-PMCID conversion results when PMIDs are supplied. |
| `downloaded_articles/PMC*.xml` | The downloaded BioC XML article files. |

---

### Step 7. Parse downloaded XML articles into passages

**User action:**  
Run the parsing cell after XML files are downloaded.

**What the system does:**  
Reads all downloaded BioC XML files and converts them into passage-level rows. It standardizes section names, extracts text, counts characters, records section and passage metadata, and reports parsing errors if any XML files fail.

**Output files:**

```text
output/step1_all_passages.csv
output/step1_parse_errors.csv
output/step1_section_summary.csv
```

**What these files contain:**

| File | Description |
|---|---|
| `step1_all_passages.csv` | One row per parsed passage, including PMCID, section type, passage name, text, character count, and source XML path. |
| `step1_parse_errors.csv` | XML files that could not be parsed and their error messages. |
| `step1_section_summary.csv` | Counts of passages and articles by normalized section type. |

---

### Step 8. Choose article sections for analysis

**User action:**  
Choose whether to keep all sections, keep selected sections, or remove selected sections. Also choose the minimum character count for passages.

**What the system does:**  
Filters parsed passages by selected section type and removes short passages below the minimum character threshold.

**Output files:**

```text
output/section_selection.csv
output/step2_selected_sections.csv
output/step3_min_characters.csv
output/step3_min_characters_section_summary.csv
```

**What these files contain:**

| File | Description |
|---|---|
| `section_selection.csv` | The section-selection mode, selected sections, removed sections, and minimum character threshold. |
| `step2_selected_sections.csv` | Passages kept after section filtering. |
| `step3_min_characters.csv` | Passages kept after applying the minimum character filter. |
| `step3_min_characters_section_summary.csv` | Section-level summary after the minimum character filter. |

---

### Step 9. Enter keywords or phrases for screening

**User action:**  
Enter one keyword or phrase per line.

**What the system does:**  
Builds case-insensitive regex patterns from the submitted terms, searches the selected passages, stores matched terms, and extracts short match contexts around the keywords.

**Output files:**

```text
output/step4_keyword_matched.csv
output/step4_keyword_pattern_summary.csv
```

**What these files contain:**

| File | Description |
|---|---|
| `step4_keyword_matched.csv` | Passages containing at least one keyword or phrase, with matched terms and surrounding context. |
| `step4_keyword_pattern_summary.csv` | Counts and summaries for each keyword pattern. |

---

### Step 10. Save CPU checkpoint before GPU steps

**User action:**  
Save the checkpoint to Google Drive, download it, or both.

**What the system does:**  
Creates a ZIP checkpoint of the project folder so the GPU runtime can resume without repeating the CPU work. This is important because changing the Colab runtime clears `/content`.

**Output files:**

```text
output/cpu_checkpoint_manifest.csv
/content/open_access_screening_cpu_checkpoint_YYYYMMDD_HHMMSS.zip
```

Optionally, the ZIP is also copied to:

```text
/content/drive/MyDrive/emnlp_open_access_screening_checkpoints/
```

**What these files contain:**

| File | Description |
|---|---|
| `cpu_checkpoint_manifest.csv` | Checkpoint metadata, project path, output path, keyword-match row count, unique PMC count, and creation time. |
| `open_access_screening_cpu_checkpoint_*.zip` | A zipped copy of the project folder for restoring in the GPU runtime. |

---

### Step 11. Switch to GPU runtime

**User action:**  
Change Colab runtime to GPU.

**What the system does:**  
Prepares for model-based steps. The notebook installs deep-learning packages and checks whether a GPU is available.

**Output files:**  
No output files are created in this step.

---

### Step 11A. Choose LLM decision model

**User action:**  
Choose one classifier option:

1. Free LongT5, no API key.
2. DeepSeek Reasoner API.
3. OpenAI API.
4. Gemini API.

**What the system does:**  
Stores the LLM provider, model name, token budget, batching mode, and API-key requirements. Free LongT5 uses a smaller input budget and one row per model call. API models use a larger estimated input-token budget.

**Output files:**  
No output files are created in this step.

---

### Step 12. Restore CPU checkpoint in GPU runtime

**User action:**  
Restore the checkpoint from Google Drive, upload the ZIP file, or confirm that files are already present.

**What the system does:**  
Unpacks the checkpoint, reloads the keyword-matched rows, restores section-selection metadata, and recreates project paths for the GPU phase.

**Input files required:**

```text
output/step4_keyword_matched.csv
output/step4_keyword_pattern_summary.csv
output/section_selection.csv
```

**Output files:**  
No new output files are created in this step.

---

### Step 13. Choose reranker and chunking settings

**User action:**  
Choose a reranker and chunk overlap. Options include MedCPT CrossEncoder, a custom CrossEncoder, or skipping reranking.

**What the system does:**  
Stores the reranker model, token limit, overlap ratio, and subgroup token strategy based on the selected LLM model.

**Output files:**  
No output files are created in this step.

---

### Step 14. Enter reranking query

**User action:**  
Enter one or more reranking queries describing the evidence that should be prioritized.

**What the system does:**  
Stores the reranking query or queries. These queries are paired with text chunks during reranking.

**Output files:**  
No output files are created in this step.

---

### Step 15. Tokenize and create safe chunks

**User action:**  
Run the tokenization and chunking cell.

**What the system does:**  
Uses the PubMedBERT tokenizer to tokenize keyword-matched passages. It creates chunks that fit the reranker token limit, applies overlap between chunks, and preserves token offsets for auditability.

**Output files:**

```text
output/step5_tokenized_paragraphs_pubmedbert.csv
output/step6_chunks.csv
```

**What these files contain:**

| File | Description |
|---|---|
| `step5_tokenized_paragraphs_pubmedbert.csv` | Keyword-matched passages with token counts, token IDs, offsets, query text, and tokenizer metadata. |
| `step6_chunks.csv` | Model-safe chunks with chunk names, token ranges, chunk text, overlap settings, and source passage metadata. |

---

### Step 16. Rerank chunks

**User action:**  
Run the reranking cell.

**What the system does:**  
Scores each chunk against the reranking query using MedCPT or the selected CrossEncoder. If reranking is skipped, all scores are set to zero.

**Output file:**

```text
output/step7_chunks_scored_medcpt.csv
```

**What this file contains:**  
All chunks with reranker scores, best-matching query, and rank.

---

### Step 17. Create ranked groups and token-budgeted subgroups

**User action:**  
Run the grouping cell.

**What the system does:**  
Ranks chunks within each article. Each article contributes its best chunk to group 1, second-best chunk to group 2, and so on. Large groups are split into subgroup CSV files so each LLM request stays within the selected input-token budget. A pending-subgroups folder is created for review.

**Output files and folders:**

```text
output/medcpt_score_groups/group_001.csv
output/medcpt_score_groups/group_002.csv
...
output/medcpt_score_subgroups/group_001_subgroup_001.csv
output/medcpt_score_subgroups_pending/group_001_subgroup_001.csv
output/step8_medcpt_score_group_summary.csv
output/step9_medcpt_score_subgroup_summary.csv
```

**What these files contain:**

| File or folder | Description |
|---|---|
| `medcpt_score_groups/` | Ranked chunk groups. Group 1 contains the best chunk per article, group 2 contains the second-best chunk per article, and so on. |
| `medcpt_score_subgroups/` | Token-budgeted subgroup files used for LLM classification. |
| `medcpt_score_subgroups_pending/` | Active pending subgroup files. This folder is updated as reviews are applied. |
| `step8_medcpt_score_group_summary.csv` | Summary of ranked groups, row counts, article counts, and token counts. |
| `step9_medcpt_score_subgroup_summary.csv` | Summary of subgroups, split status, row counts, article counts, and token budgets. |

---

### Step 18. Confirm LLM classifier settings

**User action:**  
Run the confirmation cell and provide an API key if required by the selected model.

**What the system does:**  
Confirms provider, model name, input-token budget, batch size, output-token limit, temperature, and timeout settings.

**Output files:**  
No output files are created in this step.

---

### Step 19. Write the LLM screening question

**User action:**  
Write three fields:

1. The screening question.
2. The rule for a `True` decision.
3. The rule for a `False` decision.

**What the system does:**  
Stores the instructions that will be sent to the LLM classifier. These instructions define what counts as relevant evidence.

**Output files:**  
No output files are created in this step.

---

### Step 20. Load LLM classification functions

**User action:**  
Run the function-definition cells.

**What the system does:**  
Defines functions to prepare prompts, call the selected LLM provider, parse JSON responses, normalize True/False decisions, save LLM results, and list pending subgroup files.

**Output folder prepared:**

```text
output/llm_results/
```

**Output files created later by these functions:**

```text
output/llm_results/group_XXX_subgroup_YYY_llm_results.csv
output/llm_relevant_pmc_ids.csv
output/llm_processed_subgroups.csv
output/llm_pending_subgroup_summary.csv
```

---

### Step 21. Apply LLM review and remove already-relevant articles

**User action:**  
Apply a reviewed LLM result file after inspecting or editing it.

**What the system does:**  
Reads a subgroup result file, saves PMC IDs marked `True`, records the processed subgroup, and rebuilds the pending subgroup folder. If an article is already marked relevant, all remaining chunks from that article are removed from future pending files.

**Output files updated:**

```text
output/llm_relevant_pmc_ids.csv
output/llm_processed_subgroups.csv
output/llm_pending_subgroup_summary.csv
output/medcpt_score_subgroups_pending/*.csv
```

**What these files contain:**

| File or folder | Description |
|---|---|
| `llm_relevant_pmc_ids.csv` | Unique PMC IDs marked relevant, source subgroup, source review file, final decision, and evidence. |
| `llm_processed_subgroups.csv` | Subgroups that have already been reviewed and applied. |
| `llm_pending_subgroup_summary.csv` | Current pending-subgroup status after removing already-relevant articles. |
| `medcpt_score_subgroups_pending/` | Rebuilt pending subgroup files after filtering out articles already marked relevant. |

---

### Step 22. Run one LLM subgroup first

**User action:**  
Run one pending subgroup and inspect the output CSV before applying it.

**What the system does:**  
Classifies the next pending subgroup but does not immediately apply the decisions. This allows manual inspection and correction before updating the final relevant-PMC list.

**Output file:**

```text
output/llm_results/group_XXX_subgroup_YYY_llm_results.csv
```

**What this file contains:**  
The subgroup rows plus LLM decision, evidence, source subgroup name, and review metadata.

---

### Step 23. Optional: Apply the first review

**User action:**  
After checking the first result file, apply it if the decisions are acceptable.

**What the system does:**  
Updates the relevant PMC list and pending subgroup files using the result file from Step 22.

**Output files updated:**

```text
output/llm_relevant_pmc_ids.csv
output/llm_processed_subgroups.csv
output/llm_pending_subgroup_summary.csv
output/medcpt_score_subgroups_pending/*.csv
```

---

### Step 24. Optional: Run the rest automatically

**User action:**  
Run the automatic loop only if you are comfortable applying model decisions without manually reviewing every subgroup.

**What the system does:**  
Processes all pending subgroup files, applies decisions, removes articles already marked relevant, and continues until no pending subgroup files remain.

**Output files updated:**

```text
output/llm_results/*_llm_results.csv
output/llm_relevant_pmc_ids.csv
output/llm_processed_subgroups.csv
output/llm_pending_subgroup_summary.csv
output/medcpt_score_subgroups_pending/*.csv
```

**Final status shown in notebook:**  

- Number of processed subgroups.
- Number of relevant PMC IDs found.
- Number of pending subgroup files remaining.

---

### Step 25. Download final outputs

**User action:**  
Run the final export cell.

**What the system does:**  
Creates a ZIP archive containing the full output folder and triggers a Colab download.

**Output file:**

```text
output/open_access_screening_outputs.zip
```

**What this file contains:**  
All CSV outputs, result folders, summaries, LLM classification outputs, relevant PMC ID lists, pending files, and intermediate audit files from the full run.

---

## Main final outputs

The most important final files are:

| File | Purpose |
|---|---|
| `output/llm_relevant_pmc_ids.csv` | Final list of relevant PMC IDs identified by the LLM classifier. |
| `output/llm_results/*_llm_results.csv` | Detailed LLM decisions for each reviewed subgroup. |
| `output/llm_processed_subgroups.csv` | Log of processed subgroup files. |
| `output/llm_pending_subgroup_summary.csv` | Status of remaining pending subgroup files. |
| `output/step7_chunks_scored_medcpt.csv` | Reranked evidence chunks with scores. |
| `output/step4_keyword_matched.csv` | Keyword-matched passages before model reranking and LLM classification. |
| `output/open_access_screening_outputs.zip` | Archive of all generated outputs. |

---

## Recommended review strategy

For careful screening:

1. Run only one LLM subgroup first.
2. Open the generated file in `output/llm_results/`.
3. Check the `llm_decision` and `llm_evidence` columns.
4. Correct the CSV manually if needed.
5. Apply the reviewed file.
6. Continue manually or use the automatic loop for the remaining subgroup files.

This approach keeps the pipeline auditable and reduces the chance of accepting incorrect model decisions.

---

## Notes

- The notebook only downloads articles available as open-access BioC XML.
- Articles that are not available as BioC XML are listed in `bioc_unavailable_or_failed_articles.csv`.
- Changing the Colab runtime clears `/content`, so saving the CPU checkpoint before switching to GPU is essential.
- The final LLM decision depends on the screening question and True/False rules provided by the user.
- Keep `SquidBricks-Copy.png` in the same GitHub folder as `README.md` so the logo appears at the top of the GitHub page.

---

## Repository files

```text
README.md
SquidBricks-Copy.png
open_access_screening_final.ipynb
```
