# Soko Fresh: Multi-File Data Processing Pipeline

This project demonstrates a batch data pipeline built for Soko Fresh Supermarkets, which runs branches
in Nairobi, Mombasa, Kisumu and Nakuru. Every night each branch's till system drops a CSV of the day's
sales into a shared folder. The pipeline discovers every daily file, validates and cleans each row,
consolidates the good rows into one trusted dataset, and keeps a separate, explainable record of every
row it refused so finance can chase the branches instead of chasing you.

## Core Pipeline Features

* **Automatic File Discovery:** Uses `pathlib.Path.glob()` to pick up every `transactions_*.csv` in
  `data/`, sorted by the date in the file name. Adding a new day's file needs no code changes.
* **Traceable Reading:** Every row remembers its `source_file` and `row_number` (its line in the file),
  so each rejection points to an exact line.
* **Column Validation with Sets:** Compares each header with the required columns using a set
  difference, maps the known `qty` alias to `quantity`, and rejects any file still missing a column.
* **Row Validation:** Flags blank required fields, quantities or prices that are not positive whole
  numbers, and transaction IDs already accepted in this run (in the same file or an earlier one). A
  row can carry several reasons at once.
* **Pure-Python Cleaning:** Strips whitespace, converts numbers to `int` and adds a computed
  `line_total`, using comprehensions and sets rather than pandas.
* **Reconciled Outputs:** Writes `consolidated_transactions.csv`, `rejected_transactions.csv` and
  `summary.txt`. The run stops with an error if accepted + rejected does not equal rows read.

## Requirements

* Python 3.12 or higher.
* `csv`, `pathlib` and `logging` from the standard library for reading, discovery and warnings.
* `pandas`, used only at the end to check that the outputs reconcile.

Install everything (including Jupyter) with:

```bash
pip install -r requirements.txt
```

## Project Structure

```text
.
├── pipeline.ipynb
├── data/
│   ├── transactions_2026-10-01.csv
│   ├── transactions_2026-10-02.csv
│   └── transactions_2026-10-03.csv
├── output/
│   ├── consolidated_transactions.csv
│   ├── rejected_transactions.csv
│   └── summary.txt
├── requirements.txt
└── README.md
```

## Usage

To run the pipeline, execute the cells sequentially within the `pipeline.ipynb` Jupyter Notebook. The
final cell contains a `main()` function that discovers the files, processes them, writes both CSVs, and
prints and saves the summary.

To run the whole notebook from the command line instead:

```bash
jupyter nbconvert --to notebook --execute --inplace pipeline.ipynb
```

The input and output folders are set by `INPUT_DIR` and `OUTPUT_DIR` in the setup cell. Re-running
replaces the three output files rather than appending to them.

## Results

```text
=== Processing summary ===
Files discovered:        3
Files processed:         3
Files with column issues:1  (transactions_2026-10-03.csv: missing quantity (mapped from 'qty'))
Rows read:               43
Rows accepted:           34
Rows rejected:           9
  - missing values:      4
  - invalid numbers:     2
  - duplicate IDs:       3
Total revenue (KES):     22,775
```

| Rejected row | File, line | Reason |
|---|---|---|
| TXN-1018 | 2026-10-02, line 4 | missing value: customer_id |
| TXN-1019 | 2026-10-02, line 5 | missing value: unit_price |
| TXN-1012 | 2026-10-02, line 6 | duplicate ID (original is in the 2026-10-01 file) |
| TXN-1021 | 2026-10-02, line 8 | missing value: quantity |
| TXN-1023 | 2026-10-02, line 11 | duplicate ID (exact repeat of line 10) |
| TXN-1025 | 2026-10-02, line 13 | missing value: branch |
| TXN-1028 | 2026-10-02, line 16 | invalid number: quantity 'two' |
| TXN-1029 | 2026-10-03, line 7 | duplicate ID (original is in the 2026-10-02 file) |
| TXN-1035 | 2026-10-03, line 8 | invalid number: quantity -4 is not positive |

The notebook re-reads both CSVs and checks that the row counts, unique IDs, `line_total` values and
total revenue all match the summary. The revenue was also recalculated separately from the raw
files and came to the same KES 22,775.

## Assumptions

* **The file with the unexpected header is mapped, not rejected.** `transactions_2026-10-03.csv` came
  after a POS upgrade: `quantity` is now called `qty`, and a new `cashier` column was added. The data
  is the same and only the name changed, so rejecting the file would throw away a full day of valid
  sales. The rule is:
  * only renames listed in `COLUMN_ALIASES` (`qty` to `quantity`) are mapped, and only when the real
    column is missing. Unknown names are never guessed;
  * every mapping is reported in the summary under "Files with column issues";
  * if a required column is still missing after mapping, the **whole file is rejected**, and each of its
    rows goes to the rejected file with the reason, so nothing disappears;
  * extra columns such as `cashier` are left out of the consolidated file, which keeps the agreed
    layout. They are kept in the rejected file.
* **The first copy of a duplicate is kept.** Files are processed in date order and rows in line order,
  so the first copy is the original day's record. Later copies are re-sends, and their `date` still
  shows the original day. Only *accepted* IDs count as seen, so if a first copy was rejected (for
  example for a blank field), a later valid copy is not thrown away as a duplicate.
* **`payment_ref` is optional.** Cash sales have no payment reference, so a blank one is not an error.
* **Quantities and prices must be positive whole numbers.** `"two"`, `-4`, `0` and decimals are
  rejected, which matches the `<int>` columns in the expected output.
* **A row is counted once per category.** A row with two missing fields counts as one "missing values"
  row. A row with reasons in two different categories counts in both, so the category lines can add up
  to more than "Rows rejected" (with this data they add up exactly).
* **Row numbers are file line numbers**, with the header as line 1, so they match what you see when
  you open the file in a text editor.

## Testing Error Handling

* **A fourth day's file is discovered automatically:** a copy of `data/` with an extra
  `transactions_2026-10-04.csv` and an unrelated `notes.txt` returns the four transaction files in date
  order and ignores the text file.
* **A file missing a required column with no alias:** a test `transactions_2026-10-04.csv` with no
  `payment_method` column is rejected as a whole file. Both of its rows go to the rejected file with
  `file column issue: file is missing payment_method`, the summary lists it, and the counts still
  reconcile (45 read = 34 accepted + 11 rejected).
* **A missing input folder** gives one clear message:
  `ERROR: Input folder not found: ...\no_such_folder`.
* **An unreadable file:** a test file that is not valid UTF-8 text is logged
  (`ERROR: Could not read transactions_2026-10-06.csv: ...`), listed under "Files with column issues"
  and skipped, while the other file in the folder is still processed.
* **A row with more values than the header:** it is rejected as `malformed row`, and the surplus value
  is kept in an `extra_values` column instead of being lost.

These tests run in temporary folders, so `data/` and `output/` are not touched.
