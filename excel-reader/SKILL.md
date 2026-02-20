---
name: excel-reader
description: "Read and inspect Excel workbooks (.xlsx). List sheets with dimensions, extract headers, read specific rows or row ranges, extract columns by name or index. Handles large files (50k+ rows, 100MB+) via streaming. Use when the user wants to explore, preview, or extract data from spreadsheets, when building import or ETL scripts from Excel sources, or when analyzing spreadsheet structure and content."
metadata:
  version: "1.0.0"
  author: totophe
allowed-tools: Bash(python3:*)
---

# Excel Reader — Claude Skill

Read Excel workbooks via `scripts/excel.py`. Requires Python 3. Auto-installs `openpyxl` on first run.

## Commands

| Command | Purpose | Example |
|---------|---------|---------|
| `sheets` | List all sheets with row/column counts | `python3 scripts/excel.py sheets data.xlsx` |
| `headers` | Show first N rows (default 1) | `python3 scripts/excel.py headers data.xlsx --rows 3` |
| `rows` | Extract row range | `python3 scripts/excel.py rows data.xlsx 10-20` |
| `column` | Extract a column by name or index | `python3 scripts/excel.py column data.xlsx "Email"` |

## Common Arguments

| Argument | Applies to | Default | Description |
|----------|-----------|---------|-------------|
| `--sheet NAME` | all except `sheets` | first sheet | Target sheet (case-insensitive) |
| `--limit N` | `rows`, `column` | 50 | Max rows in output |
| `--offset N` | `rows`, `column` | 0 | Skip first N data rows |
| `--header-row N` | `rows`, `column` | 1 | Row number containing headers |
| `--json` | all | off | Output as JSON instead of markdown table |

## Typical Workflow

1. List sheets: `python3 scripts/excel.py sheets file.xlsx`
2. Preview headers: `python3 scripts/excel.py headers file.xlsx --sheet Sales --rows 2`
3. Sample data: `python3 scripts/excel.py rows file.xlsx --sheet Sales --limit 10`
4. Inspect a column: `python3 scripts/excel.py column file.xlsx "Status" --sheet Sales`
5. Use the extracted structure and sample data to generate an import script

## Examples

```bash
# List sheets
python3 scripts/excel.py sheets /path/to/workbook.xlsx

# Preview headers and first 5 rows
python3 scripts/excel.py headers /path/to/workbook.xlsx --rows 5

# Extract rows 100-200 from "Orders" sheet
python3 scripts/excel.py rows /path/to/workbook.xlsx 100-200 --sheet Orders

# Extract "Email" column (first 50 values)
python3 scripts/excel.py column /path/to/workbook.xlsx "Email" --sheet Users

# Extract column by 0-based index
python3 scripts/excel.py column /path/to/workbook.xlsx 3 --sheet Users

# JSON output for structured processing
python3 scripts/excel.py rows /path/to/workbook.xlsx 1-100 --json
```

## Large Files

All commands stream via openpyxl `read_only` mode — constant memory regardless of file size. Default `--limit 50` prevents context overflow. Increase with `--limit 500` if needed.

## Limitations

- `.xlsx` only. Legacy `.xls` files must be converted first.
- `data_only=True`: returns computed values, not formulas. Files never opened in Excel may show empty formula cells.
- Merged cells show value only in the top-left cell; other cells in the merge appear empty.
