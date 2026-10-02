# Invoice Processing Bot (UiPath)

An RPA bot built in **UiPath Studio** that reads PDF invoices from a folder, extracts key fields, validates them, and separates clean records from exceptions.

## Problem Statement

Finance teams receive invoices as PDFs. Someone must open each one, copy the invoice number, date, vendor and amount into a spreadsheet, and check the values. This is slow, repetitive and error-prone.

## What the Bot Does

1. Reads every PDF in the `Input/` folder
2. Extracts **Invoice No, Date, Vendor Name and Amount** using regex
3. Validates each record against four rules
4. Writes valid records to `Output/ValidInvoices.csv`
5. Writes rejected records, with the reason, to `Output/ExceptionsInvoices.csv`
6. Moves each PDF to `Processed/` or `Failed/`

## Why a Rule-Based Approach First

Invoices from one source share a consistent layout, so basic PDF activities plus regex are deterministic, fast, free to run and easy to debug. AI/OCR is the natural next step for scanned or inconsistent invoices.

## Workflow Overview

```
Build 2 DataTables -> For Each File in Folder
   -> Try: Read PDF Text   (Catch: log error + move to Failed)
   -> Extract fields with Regex
   -> Validate (4 rules, first error wins)
   -> Valid?   Yes -> dtValid + move to Processed
               No  -> dtExceptions + move to Failed
   -> Log summary -> Write both tables to CSV
```

![Workflow part 1](docs/workflow-1.png)
![Workflow part 2](docs/workflow-2.png)

### Stage Breakdown

| Stage | Activities | Purpose |
|---|---|---|
| Setup | Build Data Table (x2) | Create `dtValid` and `dtExceptions` in memory; write to file once at the end |
| Loop | For Each File in Folder | Process any number of invoices with the same logic |
| Read | Try Catch, Read PDF Text, Log Message, Move File | Convert PDF to text; log and quarantine unreadable files so one bad PDF never stops the batch |
| Extract | Assign x4 with `Regex.Match` | Turn unstructured text into separate fields |
| Validate | Assign + If chain | Apply four rules, recording the first failure in `reason` |
| Route | If `reason = ""` | Send record to the valid or exceptions table and move the file |
| Output | Log Message, Write CSV x2 | Summary count and final deliverables |

## Regex Patterns

| Pattern | Role |
|---|---|
| `Invoice No:[ \t]*(.*)` | Extract invoice number |
| `Date:[ \t]*(.*)` | Extract date |
| `Vendor Name:[ \t]*(.*)` | Extract vendor |
| `Total Amount:[ \t]*Rs\.[ \t]*(.*)` | Extract amount |
| `^(0[1-9]\|[12][0-9]\|3[01])-(0[1-9]\|1[0-2])-\d{4}$` | Validate DD-MM-YYYY date |

## Validation Rules

| # | Rule | Reason logged on failure |
|---|---|---|
| 1 | Invoice number is not empty | Missing invoice number |
| 2 | Date matches DD-MM-YYYY | Invalid date |
| 3 | Amount is numeric | Invalid Amount |
| 4 | Invoice number not already in valid records | Duplicate invoice number |

Each check runs only if `reason = ""`, so the first problem found is the one reported.

## Problems Solved

- **`\s*` captured the next line** when a field was empty, because `\s` matches line breaks. Fixed by using `[ \t]*`, which matches spaces and tabs only.
- **`IsDate()` rejected valid dates** because it depends on the machine's regional settings (DD-MM vs MM-DD). Fixed with an explicit regex that checks the literal digits.

## Tech Stack

UiPath Studio, VB.NET expressions, Regular Expressions, DataTables, CSV

## How to Run

1. Clone this repo
2. Open the project (`project.json`) in UiPath Studio
3. Place sample PDF invoices in the `Input/` folder
4. Run `Main.xaml`
5. Check `Output/`, `Processed/` and `Failed/`

## Future Improvements

- Write to `.xlsx` with separate Valid and Exceptions sheets
- Add **Continue** in the Catch block so a failed read skips the rest of the loop iteration
- OCR for scanned invoices
- AI/LLM-based extraction for varied layouts
- Email summary of exceptions
