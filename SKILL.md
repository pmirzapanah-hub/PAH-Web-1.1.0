---
name: cabinet-estimator
description: >
  AI-powered cabinet estimation engine. Reads and analyses Mozaik cut-list plans (PDF),
  identifies all parts by type (carcass, doors, drawer fronts, finished panels, kickboards),
  assigns the correct board material and thickness per part, bins parts into 2400x1200mm
  sheets with realistic nesting efficiency, generates a full material and hardware take-off
  report, and calculates labour time for CNC cutting and assembly. Use when a user uploads
  a cut-list PDF and says "estimate", "quote", "take-off", "material list", "how many
  sheets", "hardware count", or "labour hours".
version: 1.1.0
tags: [cabinets, estimation, cut-list, materials, hardware, joinery, take-off, labour]
allowed-tools: Read, Write, Bash, Python, WebFetch
---

# Cabinet Estimator — AI Estimation Engine

## Purpose

Parse a Mozaik (or similar) cabinet cut-list plan, classify every part, assign the
correct board material, nest parts into standard 2400x1200mm sheets, and produce a
structured estimate covering:

1. **Sheet material take-off** — number of sheets per material type (carcass, MDF, kickboard — each separate)
2. **Hardware take-off** — hinges, drawer runners, handles, adjustable legs, kickboard, etc.
3. **Labour estimate** — CNC cutting time + assembly time broken down by cabinet and task
4. **Summary** — all totals consolidated for pricing

---

## Cabinet Fundamentals (Domain Knowledge)

### Carcase (Carcass)
The structural box of a cabinet. Excludes doors, drawer fronts, and decorative panels.
Components:
- **Base (BOT)** — floor of the cabinet box
- **Top (TOP)** — ceiling panel (upper cabinets or tall cabinets)
- **End panels / Sides (UEL, UER)** — left and right walls
- **Upper back (UB)** — back panel of upper/wall cabinet
- **Front rail (FrS)** — horizontal rail at the front of a base cabinet
- **Drawer base (Dba, Dbm)** — bottom/middle components inside a drawer cavity
- **Adjustable shelf (AdjSh)** — shelf that sits on shelf pins inside the carcass

**Carcass material:** White HMR (High Moisture Resistant) chipboard/particle board, typically **16mm**.

### Doors & Drawer Fronts
The visible decorative panels mounted on the front of carcasses.
- **Door(L), Door(R)** — left-hinge and right-hinge doors
- **Dwr (drawer front)** — decorative face of a drawer

**Door/drawer front material:** MDF board. Thickness varies by project:
- Standard: **18mm MDF**
- Heavier/premium: **21mm MDF**
- Lightweight/small: **16mm MDF**
Default to **18mm** unless the plan specifies otherwise.

### Finished / Applied Panels
Visible side or end panels that match the door finish.
- **AdjSh with E3 edging** on visible face — may be a finished panel
- Any panel labelled as a finished end or applied panel

**Finished panel material:** Same MDF as doors (typically **18mm**).

### Kickboard / Toe Kick
A strip of board running along the base of a run of cabinets to cover the adjustable legs.
- **Material:** White HMR 16mm (same as carcass) — cut and estimated on a SEPARATE sheet pool
- **Height:** Typically 100mm (confirm from plan or job notes)
- **Length:** Sum of all base cabinet widths in a continuous run
- Kickboard is NOT nested with carcass parts — it requires its own sheet count because
  it is typically sourced as a separate rip or offcut sheet

### Part Code Reference (Mozaik notation)

| Code suffix | Part type            | Material            | Sheet pool        |
|-------------|----------------------|---------------------|-------------------|
| UB          | Upper back           | White HMR 16mm      | HMR Carcass       |
| UEL         | Upper end left       | White HMR 16mm      | HMR Carcass       |
| UER         | Upper end right      | White HMR 16mm      | HMR Carcass       |
| BOT         | Base/bottom panel    | White HMR 16mm      | HMR Carcass       |
| TOP         | Top panel            | White HMR 16mm      | HMR Carcass       |
| FrS         | Front rail           | White HMR 16mm      | HMR Carcass       |
| Dba         | Drawer base          | White HMR 16mm      | HMR Carcass       |
| Dbm         | Drawer mid panel     | White HMR 16mm      | HMR Carcass       |
| AdjSh       | Adjustable shelf     | White HMR 16mm      | HMR Carcass       |
| Kickboard   | Toe kick strip       | White HMR 16mm      | HMR Kickboard (*) |
| Door(L)     | Left-hand door       | MDF (18mm default)  | MDF Doors         |
| Door(R)     | Right-hand door      | MDF (18mm default)  | MDF Doors         |
| Dwr         | Drawer front         | MDF (18mm default)  | MDF Doors         |

(*) Kickboard pool is always kept separate even though it shares the same material as the carcass.

> If a part code is not in this table, infer from context: parts with E1/E4 edging
> codes are typically carcass (HMR); parts with E3 edging are typically MDF/door panels.

---

## Workflow

### Step 1: Parse the Cut-List

Extract all parts from the uploaded PDF plan. For each part capture:
- **Part label** (e.g. R17C1 Door(L) #1)
- **Width x Length in mm** (e.g. 727 x 399)
- **Material** — determined by part type (see table above)
- **Thickness** — 16mm HMR for carcass; 16/18/21mm MDF for doors/fronts/panels
- **Cabinet reference** (e.g. R17C1 = Row 17, Cabinet 1)

Group parts into three distinct sheet pools:
1. **HMR Carcass** — all structural carcass parts
2. **MDF Doors/Fronts/Panels** — all door, drawer front, and finished panel parts
3. **HMR Kickboard** — kickboard strips only (estimated separately)

### Step 2: Classify Cabinets

From part labels, identify distinct cabinet units (e.g. R17C1, R17C2, R17C3, R17C4).
For each cabinet, infer its type:
- Has Door parts only → door cabinet
- Has Dwr parts only → drawer bank
- Has both Door + Dwr → mixed cabinet
- Has UB (upper back) → wall/upper cabinet
- Has BOT (base) → base cabinet
- Tall cabinet → end-panel height typically >900mm
- Drawer bank → cabinet with 2 or more Dwr parts and no Door parts

### Step 3: Nest Parts into Sheets

Standard sheet size: **2400mm x 1200mm**

Nesting rules:
- Apply a **kerf** (saw blade width) of **13mm** between parts
- Apply a **trim allowance** of **5mm** on all four sheet edges
- Usable area per sheet = (2400 - 10) x (1200 - 10) = **2390 x 1190mm**
- Use a guillotine cut nesting model: sort parts largest first, fill strips
- Calculate yield % = (total part area / sheet area) x 100
- Add a **10% waste buffer** and round up
- Nest each of the three sheet pools completely independently

**Sheet area** = 2400 x 1200 = **2,880,000 mm2**

```
sheets_required = ceil( (sum_of_part_areas / (sheet_area x efficiency)) x 1.10 )
```

Where efficiency = 0.80 (conservative default).
If the Mozaik plan header states an actual yield %, use that figure instead of 0.80.

#### Kickboard Sheet Estimation (separate pool)

Kickboard strips are long and narrow (height typically 100mm x cabinet width length).

```
kickboard_total_length_mm  = sum of all base cabinet widths (mm)
kickboard_height_mm        = 100mm (default — confirm from job notes)
kickboard_strips_per_sheet = floor(1190 / kickboard_height_mm) = floor(1190 / 100) = 11 strips
max_strip_length_mm        = 2390mm

sheets_needed = ceil( (kickboard_total_length_mm / (max_strip_length_mm x kickboard_strips_per_sheet)) x 1.10 )
```

Report kickboard as both:
- **Linear metres** (for installation and ordering reference)
- **Sheets required** (for purchasing, kept separate from carcass sheets)

### Step 4: Count Hardware

#### Hinges
- Each **Door** requires **2 hinges** (standard)
- Add 1 extra hinge for any door taller than **900mm** → 3 hinges per tall door
- Type: soft-close concealed cup hinges, 35mm cup, standard overlay
- Corner/blind cabinets: treat as 2 doors

#### Drawer Runners / Slides
- Each **Dwr (drawer front)** = 1 drawer box = **1 pair of drawer runners**
- Default: under-mount soft-close runners
- Inner drawers (behind a door): count separately

#### Handles / Knobs
- 1 handle per door
- 1 handle per drawer front
- Total handles = total doors + total drawer fronts

#### Adjustable Shelf Pins
- Each **AdjSh** panel = **4 shelf pins**
- Total = count of AdjSh panels x 4

#### Adjustable Legs / Feet
- Each base cabinet (BOT panel present) = **4 adjustable legs** (plastic, 100-150mm)
- Corner/blind base cabinets = **6 legs**

#### Kickboard
- Report total linear metres (calculated in Step 3)
- Report sheets to order (calculated in Step 3)

#### Cam Locks / Confirmat Screws (carcass assembly fasteners)
- Small cabinet (width <=600mm): **8 fasteners**
- Large/tall cabinet (width >600mm): **12 fasteners**

#### Edge Banding
- E1 = white ABS/PVC (carcass visible edges)
- E2 = matching/coloured edge band
- E3 = MDF/door colour match
- E4 = internal/raw (often not banded)
- Calculate total lineal metres per edge type by summing all edged edges per part

---

### Step 5: Calculate Labour

Labour is split into three categories. Calculate each separately, then sum for the total.

---

#### CATEGORY A — CNC Cutting Labour

Allow **10 minutes per sheet** across ALL sheet pools combined.

```
cnc_total_sheets  = HMR_carcass_sheets + MDF_door_sheets + HMR_kickboard_sheets
cnc_minutes       = cnc_total_sheets x 10
cnc_hours         = cnc_minutes / 60
```

This covers machine setup, loading, cutting, and unloading per sheet.

---

#### CATEGORY B — Cabinet Carcass Assembly Labour

Allow **15 minutes per cabinet carcass unit** regardless of cabinet type.

A "cabinet unit" = one discrete carcass box identified by its unique cabinet reference
code (e.g. R17C1, R17C2, R17C3, R17C4 = 4 units).

```
carcass_assembly_minutes = number_of_cabinet_units x 15
```

---

#### CATEGORY C — Drawer Assembly Labour

Drawer cabinets require additional time on top of the carcass allowance above.

- The **drawer carcass box itself** is already covered in Category B (15 min) — do not
  add it again here.
- Allow **15 minutes per individual drawer** for drawer assembly and fitting.

```
drawer_assembly_minutes = total_number_of_individual_drawers x 15
```

**Drawer bank worked example:**
A 3-drawer bank (e.g. R17C2 with Dwr #5, #6, #7):
- Carcass assembly: 15 min (Category B)
- Drawer #1: 15 min
- Drawer #2: 15 min
- Drawer #3: 15 min
- **Total for this cabinet: 60 minutes (1 hour)**

---

#### TOTAL LABOUR

```
total_labour_minutes = cnc_minutes + carcass_assembly_minutes + drawer_assembly_minutes
total_labour_hours   = total_labour_minutes / 60   (round to 2 decimal places)
```

---

## Output Format

Generate the following four sections in order:

---

### SECTION 1 — MATERIAL TAKE-OFF

#### 1A. White HMR Chipboard — 16mm (Carcass)

| Part # | Cabinet | Part Name | Width (mm) | Length (mm) | Area (mm2) |
|--------|---------|-----------|------------|-------------|------------|
| ...    | ...     | ...       | ...        | ...         | ...        |

**Total HMR carcass parts:** [N]
**Total part area:** [X] mm2
**Raw sheets at 80% efficiency:** [N]
**With 10% waste buffer (x1.1, rounded up): ORDER [N] sheets**

---

#### 1B. MDF Board — 18mm (Doors, Drawer Fronts, Finished Panels)

| Part # | Cabinet | Part Name | Width (mm) | Length (mm) | Area (mm2) |
|--------|---------|-----------|------------|-------------|------------|
| ...    | ...     | ...       | ...        | ...         | ...        |

**Total MDF parts:** [N]
**Total part area:** [X] mm2
**Raw sheets at 80% efficiency:** [N]
**With 10% waste buffer (x1.1, rounded up): ORDER [N] sheets**

*(Repeat block for each additional MDF thickness if applicable)*

---

#### 1C. White HMR Chipboard — 16mm (Kickboard — Separate Pool)

| Cabinet Run | Base Cabinet Widths (mm) | Kickboard Height (mm) | Kickboard Length (mm) |
|-------------|-------------------------|-----------------------|----------------------|
| ...         | ...                     | 100                   | ...                  |

**Total kickboard length: [X] mm = [X] linear metres**
**Strips per sheet (1190 / 100mm height): 11 strips**
**Max strip length per sheet: 2390mm**
**Raw sheets required: [N]**
**With 10% waste buffer (x1.1, rounded up): ORDER [N] sheets**

---

### SECTION 2 — HARDWARE TAKE-OFF

#### 2A. Hinges (Soft-Close Concealed Cup, 35mm)
| Cabinet | Door Count | Door Height >900mm? | Hinges Required |
|---------|-----------|---------------------|-----------------|
| ...     | ...       | Yes / No            | ...             |
**TOTAL HINGES: [N]**

---

#### 2B. Drawer Runners (Pairs, Under-Mount Soft-Close)
| Cabinet | Drawer Count | Runner Pairs |
|---------|-------------|--------------|
| ...     | ...         | ...          |
**TOTAL RUNNER PAIRS: [N]**

---

#### 2C. Handles
| Type             | Count |
|------------------|-------|
| Door handles     | [N]   |
| Drawer handles   | [N]   |
**TOTAL HANDLES: [N]**

---

#### 2D. Adjustable Shelf Pins
| Cabinet | Adj. Shelf Count | Pins (x4) |
|---------|-----------------|-----------|
| ...     | ...             | ...       |
**TOTAL SHELF PINS: [N]**

---

#### 2E. Adjustable Cabinet Legs / Feet
| Cabinet | Type         | Legs |
|---------|--------------|------|
| ...     | Base         | 4    |
| ...     | Corner/Blind | 6    |
**TOTAL LEGS: [N]**

---

#### 2F. Kickboard
**Total linear metres: [X] m**
*(Sheets to order: see Section 1C)*

---

#### 2G. Edge Banding (Lineal Metres)
| Edge Code | Description              | Total (m) |
|-----------|--------------------------|-----------|
| E1        | White ABS — carcass      | [X] m     |
| E2        | Colour match             | [X] m     |
| E3        | MDF / door match         | [X] m     |
**TOTAL EDGE BANDING: [X] m**

---

#### 2H. Carcass Fasteners (Cam Locks / Confirmat)
| Cabinet | Width Category  | Fasteners |
|---------|-----------------|-----------|
| ...     | Small / Large   | 8 or 12   |
**TOTAL FASTENERS: [N]**

---

### SECTION 3 — LABOUR ESTIMATE

#### 3A. CNC Cutting Labour (10 min / sheet)

| Sheet Pool               | Sheets | Minutes  |
|--------------------------|--------|----------|
| HMR Carcass 16mm         | [N]    | [N x 10] |
| MDF Doors 18mm           | [N]    | [N x 10] |
| HMR Kickboard 16mm       | [N]    | [N x 10] |
| **TOTAL CNC**            | **[N]**| **[X] min = [X] hrs** |

---

#### 3B. Carcass Assembly Labour (15 min / cabinet unit)

| Cabinet | Type              | Assembly (min) |
|---------|-------------------|----------------|
| ...     | Base / Upper / Tall | 15           |
**TOTAL CARCASS ASSEMBLY: [N] cabinets x 15 min = [X] min = [X] hrs**

---

#### 3C. Drawer Assembly Labour (15 min / drawer)

| Cabinet | Drawer Count | Drawer Labour (min) | Carcass Labour (min) | Cabinet Total (min) |
|---------|-------------|---------------------|----------------------|---------------------|
| R17C2   | 3           | 45                  | 15 (in 3B)           | 60                  |
| ...     | ...         | ...                 | ...                  | ...                 |

> The carcass column is shown for scheduling reference. It is already counted in 3B —
> do not add it again to the labour total.

**TOTAL DRAWER ASSEMBLY: [N] drawers x 15 min = [X] min = [X] hrs**

---

#### 3D. Labour Summary

| Category              | Minutes | Hours   |
|-----------------------|---------|---------|
| CNC Cutting           | [X]     | [X]     |
| Carcass Assembly      | [X]     | [X]     |
| Drawer Assembly       | [X]     | [X]     |
| **TOTAL LABOUR**      | **[X]** | **[X]** |

---

### SECTION 4 — JOB SUMMARY

| Item                          | Quantity | Unit    |
|-------------------------------|----------|---------|
| White HMR 16mm — Carcass      | [N]      | sheets  |
| MDF 18mm — Doors / Fronts     | [N]      | sheets  |
| White HMR 16mm — Kickboard    | [N]      | sheets  |
| Kickboard                     | [X]      | lin. m  |
| Soft-close hinges             | [N]      | each    |
| Drawer runner pairs           | [N]      | pairs   |
| Handles                       | [N]      | each    |
| Shelf pins                    | [N]      | each    |
| Adjustable legs               | [N]      | each    |
| Edge banding (total)          | [X]      | metres  |
| Carcass fasteners             | [N]      | each    |
| CNC Labour                    | [X]      | hours   |
| Assembly Labour (carcass)     | [X]      | hours   |
| Assembly Labour (drawers)     | [X]      | hours   |
| **TOTAL LABOUR**              | **[X]**  | **hrs** |

---

## Rules & Notes

1. **Three separate sheet pools always** — HMR Carcass, MDF Doors/Fronts, and HMR Kickboard are never mixed, even though HMR Carcass and HMR Kickboard share the same raw material.
2. **Kickboard is reported two ways** — linear metres (for installation) and sheets (for purchasing). Both must appear in the output.
3. **Default MDF thickness is 18mm** unless the plan header or part labels indicate otherwise.
4. **Sheet size is always 2400x1200mm** nominal. Usable area is 2390x1190mm after trim.
5. **Yield / efficiency default is 80%**. Override with the actual yield % from the Mozaik plan header when available.
6. **Hinge rule:** 2 per door; 3 per door if door height exceeds 900mm.
7. **Hardware separation:** Every hardware type must appear in its own clearly labelled sub-section so suppliers and trades can extract their list independently.
8. **Round up all sheet counts** — never order partial sheets.
9. **Waste buffer: multiply by 1.10 before rounding up** on all three sheet pools.
10. **CNC labour:** 10 minutes per sheet, all pools combined into one total.
11. **Carcass assembly labour:** 15 minutes per cabinet unit (every discrete carcass box).
12. **Drawer assembly labour:** 15 minutes per individual drawer. The drawer carcass itself is already covered by the 15-min carcass allowance — do not double-count.
13. **Drawer bank rule:** drawers x 15 min + 15 min (carcass) = total for that cabinet. Example: 3 drawers = 45 + 15 = 60 min.
14. **Labour is reported in both minutes and hours** — minutes for workshop scheduling, hours for client quoting.
15. **If MDF thickness is ambiguous**, flag it and present two scenarios (e.g. 16mm vs 18mm).

---

## Edge Code Key (Mozaik)

| Code | Meaning                                              |
|------|------------------------------------------------------|
| E1   | Single edge — white ABS/PVC (carcass visible edge)   |
| E2   | Single edge — coloured/matched (e.g. end panel edge) |
| E3   | Single edge — MDF/door colour match                  |
| E4   | Single edge — internal/raw (often not banded)        |

---

## Worked Example (Job: 3 Jade Place — from uploaded Part.pdf)

### Cabinets identified: R17C1, R17C2, R17C3, R17C4 (4 cabinet units)

**R17C1** — 2 doors + 1 AdjSh + BOT + UB = door cabinet (upper/wall)
**R17C2** — 3 drawer fronts (Dwr #5, #6, #7) + UB + BOT = 3-drawer bank
**R17C3** — 2 doors + 1 AdjSh + BOT + UB = door cabinet
**R17C4** — 2 doors + 1 AdjSh + panels = door cabinet

### Labour calculation:

**CNC (example — 9 sheets total across all pools):**
- 9 sheets x 10 min = 90 min = 1.5 hrs CNC

**Carcass assembly (4 cabinet units):**
- 4 x 15 min = 60 min = 1.0 hr

**Drawer assembly (R17C2 — 3 drawers):**
- 3 x 15 min = 45 min = 0.75 hr

**Total labour:**
- 90 + 60 + 45 = 195 min = **3.25 hours**

### R17C2 drawer bank breakdown (workshop reference):
```
Carcass assembly:     15 min   (in carcass total above)
Drawer #5 assembly:   15 min
Drawer #6 assembly:   15 min
Drawer #7 assembly:   15 min
                     ────────
Cabinet total:        60 min  (1 hour)
```
