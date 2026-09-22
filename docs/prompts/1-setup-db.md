# Task: Load the MTGJSON Cards and Sets Data

> **Reconstructed.** The original Phase 1 prompt was not saved. This is a
> short reconstruction of what Phase 1 did, written in 2026 so the prompt
> set is complete. Prompts 2–8 are kept exactly as originally run.

## Context
I'm testing whether an AI agent can build everything needed to go from raw
data to a plain-English analytics chatbot. The first step is getting the
raw data in place. The data is MTGJSON (Magic: The Gathering cards and
sets), which I know well enough to catch wrong answers quickly.

## Steps
1. Download <https://mtgjson.com/api/v5/AllPrintingsParquetFiles.zip> and
   unzip it.
2. Copy only `cards.parquet` and `sets.parquet` into a `data/` folder in
   the repo root. Add `data/` to `.gitignore`; the files must not be
   committed.
3. Confirm DuckDB can read both files and report the row counts:

   ```bash
   duckdb -c "SELECT COUNT(*) FROM 'data/cards.parquet';"
   duckdb -c "SELECT COUNT(*) FROM 'data/sets.parquet';"
   ```

4. Run `DESCRIBE` on each file and list the columns and types, so the next
   phase (the data catalog) has something to work from.

Do not transform the data. Later phases read the parquet files directly.
