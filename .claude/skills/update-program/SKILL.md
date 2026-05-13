---
name: update-program
description: Update the TSWIM 2026 website program schedule from the Excel spreadsheet. Use this skill whenever the user says to update the program, refresh the schedule, sync the program from Excel, or mentions changes to the program spreadsheet. Also trigger when the user says "update-program" or asks to regenerate the program HTML from the Excel data.
---

# Update Program Schedule from Excel

Reads the `_data/TSWIM_Program_Schedules.xlsx` file (the "2026" tab) and regenerates the program schedule HTML in `index.html`.

## Prerequisites

A Python venv with `pandas` and `openpyxl` is needed. Create it if missing:

```bash
python3 -m venv /tmp/xlsx_env && /tmp/xlsx_env/bin/pip install pandas openpyxl
```

If `/tmp/xlsx_env` already exists, verify it works:
```bash
/tmp/xlsx_env/bin/python3 -c "import pandas, openpyxl; print('ok')"
```

If that fails, recreate the venv.

## Step 1: Identify the correct sheet

The Excel file is at `_data/TSWIM_Program_Schedules.xlsx` (relative to the repo root). It may be a symlink. List all sheet names:

```python
import pandas as pd
path = '<repo-root>/_data/TSWIM_Program_Schedules.xlsx'
sheets = pd.read_excel(path, sheet_name=None, header=None)
for name in sheets:
    print(f'Sheet: "{name}" — shape: {sheets[name].shape}')
```

Use the sheet named exactly **"2026"** — not "2026-old", "2026-Feb", "2026 plan", or any variant. Those are archives. If you don't find a sheet named "2026", ask the user which sheet to use.

## Step 2: Read cell data and formatting

Use `openpyxl` to read values, background colors, and bold status:

```python
from openpyxl import load_workbook
wb = load_workbook(path)
ws = wb['2026']
for row in ws.iter_rows(min_row=1, max_row=ws.max_row, max_col=ws.max_column):
    for cell in row:
        if cell.value is not None:
            fill = str(cell.fill.start_color.rgb) if cell.fill and cell.fill.start_color and cell.fill.start_color.rgb else ''
            bold = cell.font.bold if cell.font else False
            print(f'{cell.coordinate}: val={repr(cell.value)[:100]} | fill={fill} | bold={bold}')
```

Also note merged cell ranges:
```python
print(list(ws.merged_cells.ranges))
```

## Step 3: Extract rich text formatting (critical!)

Openpyxl CANNOT read rich text (partial bold within a single cell). The Excel file uses rich text for ceremony/remarks cells where the first line is bold and the parenthetical second line is not. You MUST extract this from the raw XML.

```bash
cd /tmp && cp '<path-to-xlsx>' tswim_check.xlsx && mkdir -p tswim_check_unzip && cd tswim_check_unzip && unzip -o ../tswim_check.xlsx > /dev/null 2>&1
```

Then find the sheet number for "2026" from `xl/workbook.xml`, and check `xl/sharedStrings.xml` for rich text runs:

```python
import xml.etree.ElementTree as ET
tree = ET.parse('/tmp/tswim_check_unzip/xl/sharedStrings.xml')
ns = {'s': 'http://schemas.openxmlformats.org/spreadsheetml/2006/main'}
sis = tree.findall('.//s:si', ns)
for i, si in enumerate(sis):
    runs = si.findall('s:r', ns)
    if len(runs) > 1:
        full = ''
        for r in runs:
            t = r.find('s:t', ns)
            full += (t.text or '')
        print(f'String index {i}: {repr(full)[:120]}')
        for r in runs:
            rpr = r.find('s:rPr', ns)
            t = r.find('s:t', ns)
            text = t.text if t is not None else ''
            bold = rpr.find('s:b', ns) is not None if rpr is not None else False
            print(f'  bold={bold} text={repr(text)[:80]}')
```

Any cell with mixed bold/non-bold runs needs `<strong>` wrapping on the bold portions in HTML.

## Step 4: Map Excel formatting to CSS classes

The program schedule HTML lives in `index.html` inside `<section id="program">`. The CSS is in `assets/css/schedule.css`.

### Background color mapping

| Excel Fill Color | CSS Class | Usage |
|---|---|---|
| `#E2EFDA` (green) | `.sess-a` | Session A / parallel track A |
| `#FCE4D6` (orange) | `.sess-b` | Session B / parallel track B |
| `#DDEBF7` (blue) | `.sess-c` | Session C / parallel track C |
| `#FFF2CC` (yellow) | `.keynote` | Keynote sessions (always bold) |
| `#EDEDED` (grey) | `.break` | Coffee breaks, lunch |
| `#B4C6E7` (light blue) | `.day-header` | Day header rows |
| `#4472C4` (dark blue) | `.col-header` | Column header rows (white text) |
| `#FFFFFF` or no fill | `.ceremony` | Remarks, ceremonies, opening events |

### Key rules

- **All text must be centered.** The `.sess-a/b/c` classes set `text-align: center` in CSS. Do not use `text-align: left`.
- **Merged cells** (B:D merged) become `colspan="3"` in HTML.
- **Multiline cells** (containing `\n`) use the `.multiline` CSS class (enables `white-space: pre-line`). The newline appears literally in the HTML source between lines.
- **Bold cells**: If the entire cell is bold per openpyxl, add `style="font-weight:bold;"` to the `<td>`. If the cell has rich text with partial bold (from Step 3), wrap the bold portion in `<strong>`.
- **Keynote cells** are always bold via the `.keynote` CSS class — no extra styling needed.

### Row height guidelines

| Row Type | Height |
|---|---|
| Day header | `19px` |
| Column header | `19.5px` |
| Registration | `24px` |
| Paper sessions (parallel, 100 min) | `80px` |
| Coffee break (20 min) | `20px` |
| Ceremony/Remarks (10 min) | `30px` |
| Keynote (50 min) | `40px` |
| Lunch (60-70 min) | `48px`-`56px` |
| Panel (50 min) | `40px` |
| Talk + Panel (60 min) | `48px` |
| Research Incubator (100 min) | `80px` |
| Banquet (120 min) | `96px` |
| Spacer | `15px` |
| Title row | `15px` |

## Step 5: Generate the HTML

The HTML structure follows this pattern. Build the full table content between the `<table>` tags:

```html
<!-- Title -->
<tr class="title-row" style="height:15px;">
  <td colspan="5" style="font-style:italic; font-size:11pt;">Venue: College of Technology Management, NTHU</td>
</tr>
<tr class="spacer" style="height:15px;"><td colspan="5"></td></tr>

<!-- Day header -->
<tr class="day-header" style="height:19px;">
  <td colspan="5">Day N: ...</td>
</tr>

<!-- Column header (Day 1 has Activity colspan=3; Days 2-3 have Session A/B/C) -->
<tr class="col-header" style="height:19.5px;">
  <td>Time</td>
  <td>Session A (...)</td>
  <td>Session B (...)</td>
  <td>Session C (...)</td>
  <td>Location</td>
</tr>

<!-- Parallel sessions -->
<tr style="height:80px;">
  <td class="time-cell">HH:MM-HH:MM</td>
  <td class="sess-a" style="font-weight:bold;">Paper Session 1A</td>
  <td class="sess-b" style="font-weight:bold;">Paper Session 1B</td>
  <td class="sess-c" style="font-weight:bold;">Paper Session 1C</td>
  <td class="loc-cell">Parallel</td>
</tr>

<!-- Break -->
<tr style="height:20px;">
  <td class="time-cell">HH:MM-HH:MM</td>
  <td colspan="3" class="break">Coffee Break</td>
  <td class="loc-cell">Location</td>
</tr>

<!-- Ceremony/Remarks (rich text: first line bold) -->
<tr style="height:30px;">
  <td class="time-cell">HH:MM-HH:MM</td>
  <td colspan="3" class="ceremony multiline"><strong>Opening Ceremony</strong>
(Name details)</td>
  <td class="loc-cell">1F Auditorium</td>
</tr>

<!-- Keynote -->
<tr style="height:40px;">
  <td class="time-cell">HH:MM-HH:MM</td>
  <td colspan="3" class="keynote">TSWIM Keynote: Speaker Name</td>
  <td class="loc-cell">1F Auditorium</td>
</tr>

<!-- Research Incubator (rich text: first line bold, uses sess-a color) -->
<tr style="height:80px;">
  <td class="time-cell">HH:MM-HH:MM</td>
  <td colspan="3" class="sess-a" style="text-align:center;"><strong>Research Incubator Sessions</strong><br>(Up to 4 Parallel Sessions)</td>
  <td class="loc-cell">Location</td>
</tr>
```

### Colgroup (always present at the top of the table)

```html
<colgroup>
  <col class="col-time">
  <col class="col-sessA">
  <col class="col-sessB">
  <col class="col-sessC">
  <col class="col-location">
</colgroup>
```

## Step 6: Replace the HTML

Replace everything inside `<section id="program">` between `<table>` and `</table>` (keeping the colgroup and wrapping divs). The surrounding structure is:

```html
<section id="program" class="program section light-background">
  <div class="container">
    <div class="tos-header text-center">
      <h2>Program</h2>
    </div>
    <div class="program-schedule-wrap">
    <div class="program-schedule">
      <table>
        <colgroup>...</colgroup>
        <!-- REPLACE TABLE ROWS HERE -->
      </table>
    </div><!-- /.program-schedule -->
    </div><!-- /.program-schedule-wrap -->
  </div>
</section>
```

## Step 7: Verify in browser

After updating the HTML, preview the site locally (use the `preview` skill or navigate Chrome to `http://localhost:8080/#program`) and take screenshots to verify:

1. All text is centered in cells
2. Colors match the Excel (green/orange/blue for parallel sessions, yellow for keynotes, grey for breaks, white for ceremonies)
3. Bold text matches the Excel (including rich text partial bold)
4. Event order matches the Excel row order
5. All times, locations, and names are correct

## Common pitfalls to avoid

1. **Never assume the sheet hasn't changed.** Always re-read the Excel file fresh — don't rely on cached data from a previous read.
2. **The "2026" sheet may have the same name as an old sheet.** Always verify by listing all sheets and confirming the shape/content.
3. **Openpyxl cannot read rich text.** You MUST check the shared strings XML for cells with multiple `<r>` (run) elements to detect partial bold formatting.
4. **Session cells must be centered.** The CSS classes `.sess-a/b/c` must have `text-align: center` — never `left`.
5. **Multiline cells** need literal newlines in the HTML source (not `<br>`), except for "Research Incubator" which uses `<br>` because it spans all 3 session columns with `colspan="3"`.
6. **The Excel file may be a symlink.** Use the path as-is — don't try to resolve it.
7. **Python environment.** System Python may not have pandas/openpyxl and `pip install` may fail due to PEP 668. Always use a venv at `/tmp/xlsx_env`.
