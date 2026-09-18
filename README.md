[![DOI](https://zenodo.org/badge/1375260830.svg)](https://doi.org/10.5281/zenodo.22833959)

# TriNetX & Epic Cosmos Literature Analysis

A bibliometric pipeline that identifies published studies using the **TriNetX** and **Epic Cosmos** federated EHR platforms, enriches them with journal- and article-level metrics, classifies them by clinical topic, and produces summary figures.

Primary artifact: `ver_5_trinetx_cosmos_query.ipynb`

## What it does

1. **Search PubMed** for TriNetX and Epic Cosmos articles via E-utilities (`esearch` + `efetch` with history server).
2. **Parse each record** for PMID, title, year, journal, ISSN, DOI, abstract, MeSH/author keywords, author-affiliation countries, and PubMed correction flags (retractions, errata, comments, letters, expressions of concern).
3. **Deduplicate** on PMID, combining source labels so articles matching both queries are tagged `cosmos+trinetx`.
4. **Enrich via OpenAlex** — journal 2-year mean citedness, h-index, and works count (batched by ISSN); per-article citation counts (batched by PMID). Both paths retry on HTTP 429 with escalating backoff.
5. **Categorize topics** by keyword matching across title, abstract, and keywords into 12 clinical/methodologic categories. Articles can match multiple categories; unmatched articles fall to `Other/Unclassified`.
6. **Generate figures** and write the final dataset to CSV and Parquet.

## Search strategy

| Source | Query |
| --- | --- |
| TriNetX | `TriNetX[tiab] OR TriNetX[ot]` |
| Epic Cosmos | `"Epic Cosmos"[tiab] OR "Epic Cosmos"[ot]`, plus a fallback requiring `Cosmos` AND `Epic` in tiab together with an EHR/real-world-data context term |

The Cosmos fallback exists because many papers reference the platform without using the exact phrase; the context terms (EHR, electronic health record, real-world data/evidence, patient data, claims data, clinical data) suppress astronomy and unrelated "Epic" hits. Both queries are defined in the `QUERIES` dict in the configuration cell.

## Requirements

```
requests
lxml
pandas
matplotlib
seaborn
pyarrow        # for .to_parquet()
```

Python 3.9+. The notebook runs top to bottom in Colab or Jupyter; `main()` executes in the final cell.

## Configuration

Set these in the configuration cell before running:

| Variable | Default | Purpose |
| --- | --- | --- |
| `EMAIL` | `"<>"` | **Required.** Your email, passed to NCBI (polite pool) and OpenAlex (`mailto`). Set this before a full run. |
| `TEST_MODE` | `False` | When `True`, truncates the deduplicated frame for fast iteration. |
| `TEST_LIMIT` | `100` | Article cap under `TEST_MODE`. |

Rate limiting is handled with a 0.11s sleep between PubMed batches and 0.15s between OpenAlex batches. NCBI batches are 200 records; OpenAlex batches are 50.

## Running

```bash
jupyter notebook ver_5_trinetx_cosmos_query.ipynb
```

Run all cells. Outputs are written to the working directory.

## Outputs

**Checkpointed data** (written after each stage, so a failure downstream doesn't cost a re-fetch):

- `01_raw_articles.parquet` — deduplicated PubMed records
- `02_enriched_articles.parquet` — with OpenAlex journal metrics and citation counts
- `03_categorized_articles.parquet` — with topic categories
- `final_analysis.csv` / `final_analysis.parquet` — final dataset

**Figures** (PNG, 150 dpi):

- `trinetx_volume_and_if.png` — annual volume (bars) against mean journal citedness (line)
- `cosmos_volume_and_if.png` — same, for Epic Cosmos
- `combined_comparison.png` — publication volume, both platforms
- `criticisms_by_year.png` — flagged articles by year and correction type
- `geographic_distribution.png` — top 15 countries
- `topic_distribution.png` — articles per topic category

## Data dictionary

| Column | Description |
| --- | --- |
| `pmid`, `title`, `year`, `journal`, `issn`, `doi`, `abstract` | Core PubMed metadata |
| `pubmed_status` | `Clean`, or comma-joined flags: `RETRACTED`, `ERRATUM`, `COMMENT_IN`, `LETTER_IN`, `EXPRESSION_OF_CONCERN` |
| `countries` / `primary_country` | Countries parsed from affiliation strings; `primary_country` is the first match |
| `keywords` | MeSH descriptors and author keywords, `; `-joined |
| `source_query` | `trinetx`, `cosmos`, or `cosmos+trinetx` |
| `oa_display_name` | OpenAlex venue name for the ISSN |
| `citedness_2yr`, `h_index`, `works_count` | OpenAlex journal-level summary statistics |
| `cited_by_count` | OpenAlex citation count for the article |
| `topic_category` | One or more topic labels, `; `-joined |

## Known limitations

- **Country extraction** matches a fixed list of 16 country patterns against affiliation text and takes the first match per affiliation. Countries outside the list are silently dropped, and `primary_country` reflects affiliation order in the XML rather than authorship position.
- **Topic categorization** is keyword-based, not validated against a reference standard. Categories are non-exclusive and the terms are broad — "drug" and "prediction model" in particular will over-capture.
- **`citedness_2yr` is a journal-level statistic**, not an impact factor and not an article-level measure. The volume/citedness figures are labeled accordingly.
- **Platform attribution rests on text mentions**, so studies that used TriNetX or Cosmos without naming it in the title, abstract, or keywords are missed, and papers merely *discussing* a platform are counted alongside papers that used it.
- **PubMed correction flags** depend on `CommentsCorrections` being indexed, which lags the actual retraction or erratum.
- `generate_summary_report()` is currently a stub printing only the record count.

## Reproducibility notes

PubMed and OpenAlex are both moving targets: counts, citation totals, and correction flags will differ between runs. Record the run date alongside any reported figures, and keep the stage checkpoints — `01_raw_articles.parquet` is the snapshot that makes a given set of results reconstructable.
