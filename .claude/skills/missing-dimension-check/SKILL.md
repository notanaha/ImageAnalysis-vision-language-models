---
name: missing-dimension-check
description: A skill that checks a drawing for missing dimensions and annotates the drawing to be checked with red circles. This skill helps detect omitted dimension entries in engineering drawings.
---

# Drawing Missing-Dimension Check — Work Procedure

A procedure for finding omitted dimension entries (missing dimensions) in an engineering drawing by comparing a baseline drawing (complete drawing) with the drawing to be checked,
and for outputting an image that annotates the drawing to be checked with red circles.
* This document is a "reference note" (it is not registered as a skill).

---

## 1. Purpose / When to use

- **Purpose**: Inspect whether "the dimensions that should be entered are actually entered" in the drawing to be checked, by cross-referencing it against a complete baseline drawing.
- **Example triggers**: "Check this drawing for missing dimensions," "Compared to the baseline drawing, list the dimensions that are missing,"
  "Mark the omitted dimension entries on the drawing to be checked."
- **When NOT to use**: Design judgments about whether dimensions are good or bad (tolerance validity), evaluation of strength/function, or modification of the CAD model.
  This procedure is limited to "verifying the presence or absence of entries."

## 2. Inputs

| Input | Required | Notes |
|---|---|---|
| Baseline drawing (drawing with all dimensions fully entered) | ◎ | The source of the correct list of "what should exist" |
| Drawing to be checked (the check target) | ◎ | The target in which to look for omissions |
| OCR of the baseline drawing (JSON) | △ | Having it makes finalizing the dimension list and coordinate conversion faster and more accurate |

- Check `input/` first for the files (the user often does not specify the location).

## 3. Division of roles (important: lessons learned this time)

- **OCR (JSON) is useful for only two things**
  1. Listing the baseline drawing's dimensions **without misreading** (turning them into a checklist).
  2. **Coordinate conversion for the red circles** using each dimension's `source` coordinates (inches).
- **OCR alone cannot make the judgment**
  - OCR usually **only covers the baseline drawing** → "what is missing from the drawing to be checked" **requires visual analysis of the drawing to be checked**.
  - OCR contains misreads (e.g. `Ra`↔`Ro`, `+0.025`↔`+8.025`, and `2×1.40` has low confidence). Do not take it at face value.
- Conclusion: **Combining "OCR (if available) + visual inspection of both drawings"** is the fastest and most reliable. The final judgment must always be made visually.

## 4. Procedure

### Step 1 — Load and confirm identity
- Open both the baseline drawing and the drawing to be checked with `Read`.
- Confirm the part shape (outline / hole layout) is **identical**. If they are different parts, the comparison is not valid.

### Step 2 — Take inventory of the baseline drawing's dimensions
- If OCR JSON is available, enumerate all dimension strings from its `lines`/`words` to build a **complete list**.
- If not, zoom into each region of the image and read them.
- Dimension types to capture (cover them by type since they are easy to miss):
  - Length dimensions (`12.34±0.05` etc.) / diameter `Ø` / radius `R`
  - Geometric tolerance frames (position, perpendicularity, flatness) and datums (A/B/C)
  - Surface roughness (`Ra1.6` etc.) / reference dimensions (boxed) / with quantity (`2×`, `3×`) / fits (`H9(+0.025)`)

### Step 3 — Split the drawing to be checked into regions and cross-check visually
- Because the whole view is too small to read, **crop regions with PIL + enlarge** and check one region at a time:
  ```python
  from PIL import Image
  img = Image.open('input/check.png'); w,h = img.size
  crop = img.crop((int(w*0.55), int(h*0.34), int(w*0.80), int(h*0.62)))
  crop = crop.resize((crop.width*3, crop.height*3), Image.LANCZOS)
  crop.save('working/check_region.png')   # confirm this with Read
  ```
- Splitting into regions such as "top", "left vertical dimension group", "center (around holes)", "right side view", "section view" makes omissions less likely.
- **Focus point**: dimension lines (extension lines / arrows) are drawn but the **value is blank** → this is the typical "missing dimension".

### Step 4 — Determine differences and list them
- Present in the baseline drawing but absent in the drawing to be checked → **"missing"**.
- Same position as the baseline drawing but the **value differs** → report as a **separate category** called "**value mismatch (needs confirmation)**" (keep it separate from missing).
- Summarize per region, and also add the total count (e.g. ◯ missing out of 48 baseline items).

### Step 5 — Annotate the drawing to be checked with red circles
- Create conversion factors from the page dimensions (OCR's `width`/`height`, e.g. 16.5278×11.6806 inch):
  ```python
  SX = W/16.5278   # px per inch (horizontal)
  SY = H/11.6806   # px per inch (vertical)
  cx, cy = inch_x*SX, inch_y*SY
  ```
- Draw a red circle + number at each "missing" position with `ImageDraw.ellipse(..., outline=(225,0,0), width=5)`.
- Save to `output/` (e.g. `check_missing_dimensions.png`). Always present a **legend mapping** numbers to dimensions.

### Step 6 — Verification (required)
- After drawing, **re-read** the annotated image with `Read` and confirm/fine-tune that the circles sit on the correct blank positions.
- Confirm the deliverables exist with `Glob output/**/*` before reporting "complete".

## 5. Output format
1. **List of missing dimensions** (per region, with dimension values)
2. **List of value mismatches** (separate category, needs confirmation)
3. **Red-circle annotated image** (`output/`, numbered) + **legend mapping numbers → dimensions**
4. Summary (number missing / total number)

## 6. Pitfalls / guardrails
- **Do not fabricate values.** If you cannot read it, honestly write "illegible".
- If there is **no baseline drawing**, extract only the spots where "a dimension line exists but the value is blank" from the drawing to be checked alone (state clearly that completeness cannot be guaranteed).
- **Beware of layout shifts**: even for the same part, the placement of projection/section views may differ between drawings
  (this time too, the position of SECTION A-A was shifted between the baseline drawing and the drawing to be checked). Do not reuse coordinates; re-verify on the drawing-to-be-checked side.
- **Do not confuse "missing" with "value mismatch"** (e.g. baseline Ø2.00 appearing as Ø1.00 in the drawing to be checked is a mismatch, not a "missing").
- OCR is an aid. **The final judgment must always be made visually.**
- If you want to be even more efficient, **also obtain OCR of the drawing to be checked** and machine-compare the text sets → limit the visual work to final confirmation.

## 7. Checkpoint operation for long-running processing (measures for message length / 408 timeout)

The caller may instruct you to "execute one chunk at a time." This is an operation for resuming
**without growing the conversation history**, by saving state to the container's file system. Treat
**the checkpoint file as the single source of state**, not the conversation history, and follow the rules below.

### 7.1 Hold state in a file (do not depend on conversation history)
- Always save progress to `working/checkpoint.json` (create `working/` if it does not exist).
- Update the checkpoint each time you finish one chunk (roughly per Step, or per 2–3 regions).
- **Always save the checkpoint before returning a response**, and at the end of the response
  clearly write `[CONTINUE]` if incomplete or `[DONE]` if all steps are complete.
- Do not paste images, long logs, or large intermediate data into the response body (keep messages short).

### 7.2 checkpoint.json schema (minimal)
```json
{
  "status": "in_progress",
  "completed_steps": ["step1_load", "step2_baseline_inventory"],
  "next_action": "Cross-check the center region (around the holes) of the drawing to be checked",
  "baseline_dimensions": ["…list of extracted baseline dimensions…"],
  "missing": [{"id": 1, "label": "Ø2.00", "region": "center", "inch_xy": [3.21, 4.05]}],
  "value_mismatch": [],
  "input_files": {"baseline": "sample01.png", "check": "check01.png"},
  "output_files": {"annotated": "output/check_missing_dimensions.png"},
  "notes": ""
}
```
- For large intermediate artifacts such as cropped images, **save them to files and record only the path** in the checkpoint.
- `status` is one of `in_progress` / `done`.

### 7.3 Resume procedure (always perform at the start of each request)
1. First check whether `working/checkpoint.json` exists.
2. If it does, read it and decide where to continue from `completed_steps` and `next_action`.
   **Even without any conversation history**, it must be possible to resume using only the checkpoint and the input files (which remain in the container).
3. Execute **only the single chunk** in `next_action`, then update the checkpoint and
   return `[CONTINUE]` / `[DONE]` as described in 7.1.
4. If `status` is `done`, present the path of the final deliverable (the red-circle annotated image) and
   the number → dimension legend, and finish with `[DONE]`.

### 7.4 Notes
- **Do not try to complete all steps in a single response.** Chunking keeps message length and execution time down.
- Place input images, deliverables, and the checkpoint all at fixed paths inside the container, and reference them by path every time.
- **Do not return a response without updating the checkpoint** (otherwise resuming becomes impossible).
