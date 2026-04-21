# SBT Production Reporting System — Handoff Document

## Overview
This is a set of standalone HTML reporting tools built for SBT (powersports parts/repair business). All files are single HTML files that employees open locally in Chrome and bookmark. They submit daily reports via EmailJS and log data to Google Sheets via Google Apps Script webhooks.

---

## Credentials & Services

### EmailJS
- **Public Key:** `LDktS4WNzBxQU0l_Y`
- **Service ID:** `service_yc71pjs` (Outlook account — Thomas@sbt.com)
- **Template ID:** `template_2glbdbx`
- **Template variables used:** `{{to_email}}`, `{{subject}}`, `{{{html_body}}}` (triple braces for HTML rendering)
- **Known issue:** Outlook OAuth token expires periodically. When it does, all sends hang or return 412 error. Fix: emailjs.com → Email Services → reconnect Outlook. A 15-second timeout wrapper (`sendEmail()`) is implemented in machine_shop.html to prevent infinite hangs.

### Google Sheets — Inventory & Warranty Master Sheet
- **Sheet ID:** `1NUUElaHFrJpfVBp3D4IAHCrctiWdj_mTQENo7S0yVzw`
- **Sheet URL:** https://docs.google.com/spreadsheets/d/1NUUElaHFrJpfVBp3D4IAHCrctiWdj_mTQENo7S0yVzw
- **Webhook URL:** `https://script.google.com/macros/s/AKfycby43laZkSiiHfgu1xeZBp5zIBffYTA4xCA9kIlDY3iodGTLRCG0iHVEmDXORkxkP8yw/exec`
- **Apps Script file:** `warranty_only_script.gs` (despite the name, handles all sources)
- **Tabs in sheet:** Warranties, Machine Shop Log, Maintenance Log, CNC Hours Log, Defects

### Google Sheets — Inventory (Paint + 4-Stroke)
- **Sheet ID:** `1sHCRc3WUGZN2UglGoCM6Nt6thJ6nmMDhN2v0abEtGiw`
- **Webhook URL:** `https://script.google.com/macros/s/AKfycbx-sT_8gLb_Ri8kvTKbARVaphoPWTgAPNNLZ6ZNzzuRVo6ZpTAjgbrb360IPNzhfmC9Dg/exec`
- **Apps Script file:** `inventory_script_v2.gs`
- **Tabs in sheet:** Inventory, WIP, Log

---

## Files

### 1. `painter_report.html` — Daily Paint Report
- **Who uses it:** Amir (painter)
- **Purpose:** Log parts painted each day
- **Fields:** Part number, Quantity (multiple rows)
- **Email subject:** `Painted Today - YYYY-MM-DD`
- **Email to:** thomas@sbt.com + group distribution (not yet set up via Gmail Groups)
- **Google Sheets:** Parts painted → ADD to Inventory tab (source: `paint`)
- **Local storage key:** `paint_history`

### 2. `4stroke_production.html` — 4-Stroke Production Report
- **Who uses it:** 4-stroke production team
- **Purpose:** Log daily shipping and WIP
- **Sections:**
  - **Shipping** — parts shipped (part # + qty)
  - **WIP** — work in progress components (part # + qty)
  - **Heads** — OE Head or PBW Head with qty (does NOT affect inventory)
- **Email subject:** `4-Stroke Production - YYYY-MM-DD`
- **Email to:** thomas@sbt.com + group (not yet set up)
- **Google Sheets logic:**
  - WIP rows (24-XXX, 29-XXX-22k parts) → ADD to WIP tab (source: `wip`)
  - Shipping rows → ADD to Inventory tab (source: `shipping`)
  - 29-XXX shipped → also REMOVES its two components from WIP tab
  - 40-XXX shipped → ADD to Inventory only, no WIP deduction
- **Assembly component map (hardcoded in Apps Script):**
  - 29-410 → [24-410, 29-410-22k]
  - 29-411 → [24-411, 29-411-22k]
  - 29-112 → [24-112, 29-112-22k]
  - 29-112A → [24-112A, 20-112-22k]
  - 29-113 → [24-113, 29-112-22K]
  - 29-113A → [24-113A, 29-112-22k]
  - 29-415 → [24-415, 29-415-22k]
  - 29-417 → [24-417, 29-417-22k]
  - 29-418 → [24-418, 29-418-22k]
- **Local storage key:** `4stroke_history`

### 3. `parts_washing.html` — Parts Washing Report
- **Who uses it:** Wash station employee
- **Purpose:** Log parts washed each day
- **Fields:** Part number (3 digits only), Description, Quantity (multiple rows)
- **Email subject:** `Washed Today - YYYY-MM-DD`
- **Email to:** thomas@sbt.com
- **Google Sheets:** None — email only
- **Local storage key:** `wash_history`

### 4. `2stroke_warranty.html` — 2-Stroke Warranty Tracker
- **Who uses it:** Dante (teardown), Tyler, others in teardown dept
- **Purpose:** Log warranty claims with full details and visualize trends
- **Nav tabs:** Submit, Chart, Filter, Raw Data
- **Fields:** Submitted By, Date (auto), Part #, RG #, Customer Name, Issue Description
- **Email subject:** `2-Stroke Warranties Today - YYYY-MM-DD`
- **Email to:** thomas@sbt.com
- **Google Sheets:** Logs to Warranties tab (source: `warranty`)
- **Chart:** Bar chart by part number, month/year selector. Click bar → jumps to Filter page pre-filled for that part + month
- **Data source:** Fetches live from Google Sheets via `?action=getWarranties`
- **Known date issue (fixed):** Google Sheets returns dates as Date objects; Apps Script uses explicit year/month/day formatting to ensure YYYY-MM-DD output

### 5. `machine_shop.html` — Machine Shop Dashboard
- **Who uses it:** Machine shop workers (currently 2 people, submit independently each day)
- **Purpose:** Log daily machining work, maintenance events, CNC hours, and defects
- **Nav tabs:** Log Work, Log Maintenance, CNC Hours, Chart, Filter, Log Defect

#### Tab: Log Work
- **Fields:** Submitted By, Date (auto), then per-entry rows:
  - Part Number (free text, e.g. "203")
  - Part Type (dropdown: Case, Cylinder, Rotary Cover, Web, Head, Valve, Pump Cover, Other)
  - Cylinder Sub-Type (appears only when Cylinder selected: Std, Reg, 4-Stroke) — saved as separate field but NOT shown in filter/chart grouping
  - Operation (dropdown: Deck, Bore, Deck and Bore, Deck (Degree Cut), Mill (Threaded Insert), Mill (Deck), Hog, Hone, Other)
  - Actual Time (hours + minutes)
  - Qty
- **All fields mandatory** — validates before submit
- **Email subject:** `Machine Shop Log - [Name] - YYYY-MM-DD`
- **Email to:** thomas@sbt.com
- **Google Sheets:** Logs to Machine Shop Log tab (source: `machine_shop`)
- **Sheet columns:** Date, Name, Part Number, Part Type, Cylinder Sub-Type, Operation, Qty, Actual Min

#### Tab: Log Maintenance
- **Fields:** Performed By, Date, CNC Machine (CNC #1/2/3), PM Type, Time taken (hrs + mins), Notes
- **PM Types:** 100hr (trough clean), 500hr (full clean), 1000hr (service), 2000hr (major service), Other
- **On submit:** Resets PM countdown for that machine/interval in localStorage
- **Email subject:** `CNC Maintenance - [CNC#] - YYYY-MM-DD`
- **Google Sheets:** Logs to Maintenance Log tab (source: `maintenance`)

#### Tab: CNC Hours
- **Fields:** Single text input per machine in HH:MM:SS format
- **Starting odometer values (hardcoded):**
  - CNC #1: 4251:45:42
  - CNC #2: 3769:09:51
  - CNC #3: 3792:11:46
- **On input:** Live delta calculation shown (today's hours + utilization %)
- **On submit:** 
  - Sends daily email report with delta + individual utilization % per machine
  - Shows **total utilization** = sum of all 3 machine hours ÷ 24hrs at bottom of email
  - Saves latest reading to localStorage
  - Sends separate PM alert emails if any machine is overdue
  - Logs to CNC Hours Log tab (source: `cnc_hours`)
- **PM alert email subject:** `IMPORTANT service on CNC #X IS NEEDED`
- **PM thresholds:**
  - 100hrs → Clean bottom troughs/chip pans
  - 500hrs → Coolant tank, chip conveyor, spindle taper, guideways
  - 1000hrs → Full service + bearings + heat exchanger + lubrication
  - 2000hrs → DMG Mori major service
- **PM state stored in:** localStorage key `cnc_hours`

#### Tab: Chart
- **Shows:** Performance table sorted longest → shortest average time per unit
- **Grouping:** Part # + Part Type + Operation as unique key (Cylinder sub-type intentionally excluded from grouping)
- **Metric:** Actual minutes ÷ qty = per-unit time, then averaged across all runs
- **Month/year selector** to filter by period
- **Data source:** Fetches live from Google Sheets via `?action=getMachineShop`

#### Tab: Filter
- **Filter by:** Name, Part #, Operation, date range
- **Shows:** Date, By, Part #, Part Type, Operation, Qty, Actual Min

#### Tab: Log Defect
- **Purpose:** Report a part that was ruined/messed up
- **Fields:** Reported By, Date (auto), Part #, Part Type, Cylinder Sub-Type (if Cylinder), Operation, Description
- **All fields mandatory**
- **Email subject:** `Machine Shop Defect Report - [Part#] - YYYY-MM-DD`
- **Email to:** thomas@sbt.com
- **Google Sheets:** Logs to Defects tab (source: `defect`)
- **Defect chart:** Count of defects by Part # + Part Type + Operation, most → least, same month/year selector

---

## Apps Scripts

### `warranty_only_script.gs` — Master Script (deploy to warranty/machine shop sheet)
Handles sources: `warranty`, `machine_shop`, `maintenance`, `cnc_hours`, `defect`
GET actions: `getWarranties`, `getMachineShop`, `getDefects`

### `inventory_script_v2.gs` — Inventory Script (deploy to inventory sheet)
Handles sources: `paint`, `wip`, `shipping`
Contains assembly component map for 29-XXX auto-deduction logic

---

## Known Issues & Notes

1. **EmailJS Outlook token expiry** — Reconnect periodically at emailjs.com. 15-second timeout implemented in machine_shop.html to prevent hangs.
2. **Google Sheets date format** — Sheets returns dates as Date objects. Apps Script formats explicitly to YYYY-MM-DD using getFullYear/getMonth/getDate.
3. **Machine Shop Log column shift** — Early test submissions (rows 2-10) were logged before Cylinder Sub-Type column was added, causing column misalignment. Delete those rows manually. New submissions have all 8 columns correct.
4. **Parts washing** — No Google Sheets integration, email only.
5. **Gmail Groups** — Paint report and 4-stroke report were intended to send to Gmail group distribution lists but this was not completed. Currently sends to thomas@sbt.com only.
6. **Inventory sheet** — Only tracks inflows (painted parts, WIP components, shipped finished goods). AccountMate handles outbound shipping so inventory numbers only go up. Cross-reference with AccountMate for true stock levels.
7. **CNC odometer state** — Stored in localStorage on the submitting computer. If the file moves to a new computer, starting values reset to the hardcoded baseline.

---

## Tech Stack
- Pure HTML/CSS/JS (no frameworks, no build step)
- EmailJS v4 for email sending
- Google Apps Script (deployed as Web App) for Google Sheets read/write
- Chart.js 4.4.0 for warranty bar chart
- SheetJS (xlsx) for Excel export on paint/4-stroke forms
- localStorage for local history and CNC odometer state
- All files are single-file — no external CSS or JS files
