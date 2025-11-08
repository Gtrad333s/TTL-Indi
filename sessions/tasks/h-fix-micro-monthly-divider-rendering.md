---
name: h-fix-micro-monthly-divider-rendering
branch: fix/h-fix-micro-monthly-divider-rendering
status: pending
created: 2025-01-08
submodules: [TTL_v7_Rebuild]
---

# Fix Micro and Monthly Cycle Divider Rendering

## Problem/Goal

After implementing the two-stage cycle detection pattern fixes in task `h-fix-cycle-boundary-detection-bugs`, visual testing revealed two critical rendering bugs:

### Micro Cycle Divider Bug
**Current Behavior:** Micro cycle dividers only show Q1 labels at new session quarter transitions (every 90 minutes). The micro cycle is not calculating or rendering Q2/Q3/Q4 quarters within each session quarter.

**Expected Behavior:** Within each 90-minute session quarter, micro cycle should calculate and render 4 quarters:
- Micro Q1: 0-22.5 minutes
- Micro Q2: 22.5-45 minutes
- Micro Q3: 45-67.5 minutes
- Micro Q4: 67.5-90 minutes

**Root Cause:** The inline micro quarter transition logic (lines 573-603 in TTL_v7_01_Cycles.pine) is logging dividers correctly, but the rendering may not be showing them, OR the quarter calculation itself is incorrect and always returning Q1.

### Monthly Cycle Divider Bug
**Current Behavior:** Monthly cycle dividers have rendering issues (specifics to be investigated from v6 reference implementation).

**Expected Behavior:** Monthly dividers should render at weekly cycle boundaries (Sunday 18:00) with appropriate Q1/Q2/Q3/Q4/Qx labels based on week-of-month calculation.

### Investigation Approach
1. **Reference v6 implementation** at `C:\Users\garic\Downloads\TTL_Fractal_v6_Modular.pine` to understand how micro and monthly divider rendering worked correctly
2. **Use code-review agent** to analyze differences between v6 and v7 rendering logic
3. **Apply targeted fixes** to v7 divider calculation and rendering
4. **Verify with code-review agent** before visual testing

## Success Criteria
- [ ] Micro cycle renders 4 distinct quarters (Q1/Q2/Q3/Q4) within each 90-minute session quarter
- [ ] Micro dividers appear at correct intervals: 0min, 22.5min, 45min, 67.5min within each session quarter
- [ ] Monthly cycle dividers render correctly at weekly boundaries with accurate Q1/Q2/Q3/Q4/Qx labels
- [ ] Visual validation on M1 chart shows micro quarters cycling 1→2→3→4→1 throughout each session quarter
- [ ] Visual validation on H4 chart shows monthly dividers at Sunday 18:00 with correct labels
- [ ] Debug table confirms micro quarter values stay within 1-4 and cycle correctly
- [ ] Code review confirms v6 rendering patterns correctly applied to v7

## Context Manifest

### How Micro Cycle Quarters Should Work: Time-Based Calculation Within Session Quarters

#### The Architecture: Micro Cycles as Time-Based Fractals

Micro cycles are the finest granularity fractals in the TTL system. Understanding their structure is critical to fixing the rendering bug:

**Hierarchical Structure:**
- Each **daily quarter** = 6 hours = 360 minutes
- Each daily quarter contains 4 **session quarters** = 90 minutes each
- Each session quarter contains 4 **micro quarters** = 22.5 minutes each

**Critical Timing Math:**
- Micro Q1: 0-22.5 minutes elapsed in session quarter
- Micro Q2: 22.5-45 minutes elapsed in session quarter
- Micro Q3: 45-67.5 minutes elapsed in session quarter
- Micro Q4: 67.5-90 minutes elapsed in session quarter

**Micro Cycle Reset Behavior:**
Micro cycles RESET at every session quarter boundary (every 90 minutes). This means:
- When session transitions Q1→Q2 at 19:30 (90min after 18:00), micro resets to Q1
- When session transitions Q2→Q3 at 21:00 (90min after 19:30), micro resets to Q1
- When session transitions Q3→Q4 at 22:30 (90min after 21:00), micro resets to Q1
- When session transitions Q4→Q1 at 00:00 (new daily quarter), micro resets to Q1

#### How V6 Calculates Micro Quarters: Two-Stage Pattern

**Stage 1: Micro Cycle Reset at Session Quarter Boundaries**

V6 detects session quarter transitions to reset micro cycles (v6 lines 2560-2640):

```pinescript
// Detect when a new 90-minute session quarter starts
bool is_new_session_quarter = session_cycle.current_quarter != old_session_q or is_new_session

// Reset micro quarters when NEW 90-MINUTE SESSION QUARTER starts
if is_new_session_quarter
    // Reset ALL micro quarter tracking for new session quarter
    micro_cycle.current_quarter := 1
    micro_cycle.parent_start_bar := bar_index
    micro_cycle.parent_timestamp := time  // CRITICAL: Anchor point for time-based calculation
    micro_cycle.TO_price := na
    micro_cycle.TO_bar := na
    micro_cycle.q1_start_bar := bar_index

    // Reset previous quarter tracker so transitions work properly
    prev_micro_q := 0  // Set to 0 so first transition (0→1) doesn't match any transition checks

    // Reset all quarter start/end bars
    micro_cycle.q2_start_bar := na
    micro_cycle.q3_start_bar := na
    micro_cycle.q4_start_bar := na

    // Log Q1 divider at session quarter start
    array.push(hist_micro_q1_bars, bar_index)
    if array.size(hist_micro_q1_bars) > 50
        array.shift(hist_micro_q1_bars)
```

**Stage 2: Time-Based Quarter Calculation Within Session Quarter**

Once a session quarter starts and micro resets, V6 calculates which micro quarter based on time elapsed since `micro_cycle.parent_timestamp` (v6 lines 2642-2653):

```pinescript
// Calculate micro quarter based on progress within current 90-minute session quarter
if not na(micro_cycle.parent_timestamp) and not is_new_session_quarter
    int time_elapsed_ms = time - micro_cycle.parent_timestamp
    float minutes_elapsed = time_elapsed_ms / 60000.0  // Convert milliseconds to minutes

    // Determine which micro cycle quarter based on minutes elapsed
    // Each 90min session quarter has 4 independent micro cycle quarters of 22.5 minutes each
    int calculated_quarter = minutes_elapsed < 22.5 ? 1 : minutes_elapsed < 45.0 ? 2 : minutes_elapsed < 67.5 ? 3 : 4

    // Update the quarter
    micro_cycle.current_quarter := calculated_quarter
```

**The Calculation Logic:**
- `time` = current bar timestamp
- `micro_cycle.parent_timestamp` = timestamp when session quarter started (reset anchor point)
- `time_elapsed_ms` = milliseconds since session quarter started
- `minutes_elapsed` = elapsed time in minutes (0-90 range within session quarter)
- `calculated_quarter` = which micro quarter (1-4) based on 22.5-minute thresholds

**Stage 3: Micro Quarter Transition Detection**

After calculating the quarter, V6 detects Q1→Q2, Q2→Q3, Q3→Q4 transitions using `prev_micro_q` comparison (v6 lines 2787-2807):

```pinescript
// Track Micro quarter transitions and capture True Open at Q2 start
if micro_cycle.current_quarter == 2 and prev_micro_q == 1
    micro_cycle.q2_start_bar := bar_index
    micro_cycle.q2_end_bar := na
    micro_cycle.TO_price := open  // TRUE OPEN captured at micro Q2 start
    micro_cycle.TO_bar := bar_index

    // Log Micro Q2 historical divider
    array.push(hist_micro_q2_bars, bar_index)
    if array.size(hist_micro_q2_bars) > 50
        array.shift(hist_micro_q2_bars)

else if micro_cycle.current_quarter == 3 and prev_micro_q == 2
    micro_cycle.q2_end_bar := bar_index - 1
    micro_cycle.q3_start_bar := bar_index

    // Log Micro Q3 historical divider
    array.push(hist_micro_q3_bars, bar_index)
    if array.size(hist_micro_q3_bars) > 50
        array.shift(hist_micro_q3_bars)

else if micro_cycle.current_quarter == 4 and prev_micro_q == 3
    micro_cycle.q3_end_bar := bar_index - 1
    micro_cycle.q4_start_bar := bar_index

    // Log Micro Q4 historical divider
    array.push(hist_micro_q4_bars, bar_index)
    if array.size(hist_micro_q4_bars) > 50
        array.shift(hist_micro_q4_bars)

// Update prev_micro_q tracker
prev_micro_q := micro_cycle.current_quarter
```

**Why This Pattern Works:**
1. Session quarter transition → Reset micro to Q1, set anchor timestamp, reset `prev_micro_q := 0`
2. Every bar → Calculate which quarter (1-4) based on elapsed time since anchor
3. When `calculated_quarter` changes → Log divider, update quarter start bars
4. After 90 minutes → Session quarter transitions, micro resets again

#### How V7 Currently Handles Micro Cycles: The Bug

**V7's Micro Reset Logic (v7 lines 543-570):**

V7 correctly implements Stage 1 (micro reset at session quarter boundaries):

```pinescript
// Micro Cycle Reset at Session Quarter Boundaries
var int old_session_q = 0
bool is_new_session_quarter = (session_q != old_session_q) or is_new_session
if is_new_session_quarter and not is_in_pause
    // Log Micro Q1 divider at micro cycle reset
    array.push(hist_micro_q1, bar_index)
    if array.size(hist_micro_q1) > 100
        array.shift(hist_micro_q1)

    // Reset micro cycle
    micro_cycle.current_quarter := 1
    micro_cycle.q1_start_bar := bar_index
    micro_cycle.parent_start_bar := bar_index
    micro_cycle.parent_timestamp := time  // ✓ Anchor set correctly

    // Reset all quarter start bars (fresh cycle)
    micro_cycle.q2_start_bar := na
    micro_cycle.q3_start_bar := na
    micro_cycle.q4_start_bar := na

    // Reset tracker to 0 so Q1→Q2 transition detects properly in Stage 2
    prev_micro_q := 0  // ✓ Reset correctly
else
    micro_cycle.current_quarter := micro_q  // ✓ Updates from calculation
```

**V7's Micro Quarter Calculation (v7 lines 403):**

V7 uses the CycleEngine library function (same as v6):

```pinescript
int micro_q = CycleEngine.f_micro_q_index(daily_q, current_hour, current_minute)
```

**THE BUG:** V7 uses `f_micro_q_index(daily_q, current_hour, current_minute)` which calculates micro quarters **within the entire daily quarter (1-16 range)**, NOT within the 90-minute session quarter (1-4 range).

Looking at CycleEngine function (CycleEngine.pine lines 332-356):

```pinescript
export f_micro_q_index(int daily_q, int h, int m) =>
    int micro_q = 0
    float quarter_size = MICRO_QUARTER_MINUTES  // 22.5

    // Calculate minutes into the daily quarter
    int minutes_in_daily = 0

    if daily_q == 1  // Daily Q1: 18:00-23:59 (6 hours)
        if h >= 18 and h <= 23
            minutes_in_daily := (h - 18) * 60 + m
    else if daily_q == 2  // Daily Q2: 00:00-05:59 (6 hours)
        // ... similar logic for Q2/Q3/Q4

    // Determine micro quarter (1-16 within daily quarter)
    micro_q := int(math.floor(minutes_in_daily / quarter_size)) + 1
    micro_q := math.max(1, math.min(16, micro_q))  // Clamp to 1-16

    micro_q
```

**The Problem:**
- CycleEngine.f_micro_q_index returns 1-16 (micro quarters within 6-hour daily quarter)
- V7 sets `micro_cycle.current_quarter := micro_q` which gets values 1-16
- V7's inline transition detection (lines 573-608) compares against 1-4 only
- Transitions like "Q2 start" check `micro_cycle.current_quarter == 2 and prev_micro_q == 1`
- But `micro_cycle.current_quarter` could be 5, 6, 7, 8... (beyond Q1-Q4)
- So Q2/Q3/Q4 transitions NEVER TRIGGER after the first session quarter

**V7's Inline Transition Logic (v7 lines 578-608):**

```pinescript
if not is_in_pause
    // Micro Q1→Q2 transition
    if micro_cycle.current_quarter == 2 and prev_micro_q == 1  // FAILS when quarter > 4!
        micro_cycle.q2_start_bar := bar_index
        // ... log Q2 divider

    // Micro Q2→Q3 transition
    else if micro_cycle.current_quarter == 3 and prev_micro_q == 2  // FAILS when quarter > 4!
        micro_cycle.q3_start_bar := bar_index
        // ... log Q3 divider

    // Micro Q3→Q4 transition
    else if micro_cycle.current_quarter == 4 and prev_micro_q == 3  // FAILS when quarter > 4!
        micro_cycle.q4_start_bar := bar_index
        // ... log Q4 divider

    // Update prev_micro_q tracker
    prev_micro_q := micro_cycle.current_quarter  // Stores 1-16 range!
```

**Visual Example of the Bug:**

Session Q1 starts at 18:00:
- Bar at 18:00: `micro_q = 1` → Reset micro, `prev_micro_q := 0`
- Bar at 18:05: `micro_q = 1` → `micro_cycle.current_quarter = 1`
- Bar at 18:25: `micro_q = 2` → `micro_cycle.current_quarter = 2`, detects Q1→Q2 ✓, logs Q2 divider
- Bar at 18:50: `micro_q = 3` → `micro_cycle.current_quarter = 3`, detects Q2→Q3 ✓, logs Q3 divider
- Bar at 19:15: `micro_q = 4` → `micro_cycle.current_quarter = 4`, detects Q3→Q4 ✓, logs Q4 divider

Session Q2 starts at 19:30 (90 minutes after 18:00):
- Bar at 19:30: Reset micro cycle, `micro_cycle.parent_timestamp := time`, `prev_micro_q := 0`
- Bar at 19:30: `micro_q = 5` (5th micro quarter in daily Q1) → `micro_cycle.current_quarter = 5`
- Bar at 19:55: `micro_q = 6` → `micro_cycle.current_quarter = 6`
- Bar at 20:20: `micro_q = 7` → `micro_cycle.current_quarter = 7`
- Bar at 20:45: `micro_q = 8` → `micro_cycle.current_quarter = 8`

**None of the transitions detect because:**
- Q1→Q2 check: `micro_cycle.current_quarter == 2` fails (it's 5, 6, 7, 8)
- Q2→Q3 check: `micro_cycle.current_quarter == 3` fails
- Q3→Q4 check: `micro_cycle.current_quarter == 4` fails

**Result:** Only Q1 dividers render (at session quarter boundaries). Q2/Q3/Q4 dividers never render because transitions never detect.

#### The Fix: Use Time-Based Calculation Like V6

V7 needs to calculate micro quarters **within the 90-minute session quarter**, not within the 6-hour daily quarter. V6 does this by:

1. Storing `micro_cycle.parent_timestamp` when session quarter starts
2. Calculating elapsed time: `time - micro_cycle.parent_timestamp`
3. Dividing by 22.5-minute intervals to get quarter (1-4)

**Fix Implementation (replace lines 566-567 in v7):**

```pinescript
// Update old_session_q tracker for next bar
old_session_q := session_q

// Calculate micro quarter based on time elapsed within session quarter (1-4 range)
if not is_new_session_quarter and not na(micro_cycle.parent_timestamp)
    int time_elapsed_ms = time - micro_cycle.parent_timestamp
    float minutes_elapsed = time_elapsed_ms / 60000.0

    // Calculate which micro quarter (1-4) within 90-minute session quarter
    int calculated_micro_q = minutes_elapsed < 22.5 ? 1 : minutes_elapsed < 45.0 ? 2 : minutes_elapsed < 67.5 ? 3 : 4
    micro_cycle.current_quarter := calculated_micro_q
else
    micro_cycle.current_quarter := micro_q  // Fallback to library function (should be 1 after reset)
```

**Why This Fixes The Bug:**
- After session quarter reset, calculate quarter based on elapsed time **within that 90-minute window**
- `calculated_micro_q` always returns 1-4 (never exceeds 4)
- Transition detection logic works: `current_quarter == 2 and prev_micro_q == 1` succeeds
- All Q2/Q3/Q4 dividers render correctly

---

### How Monthly Cycle Dividers Should Work: Week-Aligned Quarters

#### The Architecture: Monthly Quarters Aligned to Weekly Boundaries

Monthly cycles in TTL have a unique architectural requirement: **monthly quarters must align with weekly cycle boundaries** (Sunday 18:00 ET). This is critical for proper H4 timeframe visualization and DFR calculations.

**Monthly Quarter Structure:**
- Month starts: First Sunday 18:00 after the 1st of the month (or exactly on 1st if it's Sunday 18:00)
- Month quarters: Each aligns to a Sunday 18:00 weekly cycle start
  - MQ1 = Week 1 of month (first full week)
  - MQ2 = Week 2 of month
  - MQ3 = Week 3 of month
  - MQ4 = Week 4+ of month (may span multiple weeks)
- Partial weeks: Labeled "Qx" when weekly cycle crosses month boundary mid-week

**Critical Concept: Weekly Tracking, Not Time-Based**

Unlike micro/session/daily cycles which use time-based calculations, monthly quarters are **event-based** - they align to weekly cycle starts. V6 tracks EVERY Sunday 18:00 and labels it with the appropriate monthly quarter (Q1/Q2/Q3/Q4/Qx).

#### How V6 Tracks Monthly Dividers: Weekly Boundary Alignment

**V6's Monthly Weekly Tracking (v6 lines 1618-1646):**

V6 tracks monthly dividers at EVERY weekly cycle start (Sunday 18:00):

```pinescript
if is_new_weekly
    // Track monthly weekly dividers with Qx labels at every weekly cycle start (Sunday 18:00 ET only)
    if current_dow == dayofweek.sunday and current_hour == 18
        // Additional check: don't add duplicate if this bar_index is already tracked
        bool already_tracked = false
        if array.size(monthly_week_divider_bars) > 0
            int last_tracked = array.get(monthly_week_divider_bars, array.size(monthly_week_divider_bars) - 1)
            already_tracked := last_tracked == bar_index

        if not already_tracked
            current_monthly_week_label := f_get_monthly_quarter_label()  // Returns Q1/Q2/Q3/Q4/Qx
            array.push(monthly_week_divider_bars, bar_index)
            array.push(monthly_week_divider_labels, current_monthly_week_label)

            // CAPTURE MONTHLY QUARTER STARTS AND TRUE OPEN AT SUNDAY 18:00 ET
            // This ensures all monthly quarter transitions happen at exact weekly cycle starts
            if current_monthly_week_label == "Q2" and na(monthly_cycle.TO_bar)
                monthly_cycle.q2_start_bar := bar_index
                monthly_cycle.TO_price := open  // TRUE OPEN = OPEN price at Q2 start (Sunday 18:00)
                monthly_cycle.TO_bar := bar_index
            else if current_monthly_week_label == "Q3" and na(monthly_cycle.q3_start_bar)
                monthly_cycle.q3_start_bar := bar_index
            else if current_monthly_week_label == "Q4" and na(monthly_cycle.q4_start_bar)
                monthly_cycle.q4_start_bar := bar_index

            // Maintain maximum size
            while array.size(monthly_week_divider_bars) > max_monthly_week_dividers
                array.shift(monthly_week_divider_bars)
                array.shift(monthly_week_divider_labels)
```

**Key Storage Arrays (v6 lines 816-822):**

```pinescript
// Monthly weekly dividers with Qx support
var array<int> monthly_week_divider_bars = array.new<int>()      // Bar positions
var array<string> monthly_week_divider_labels = array.new<string>()  // Q1/Q2/Q3/Q4/Qx
var array<line> monthly_week_divider_lines = array.new<line>()   // Visual objects
var array<label> monthly_week_divider_label_objects = array.new<label>()
var string current_monthly_week_label = na
var int max_monthly_week_dividers = 20
```

**Monthly Quarter Label Calculation (v6 lines 1364-1386):**

V6 uses forward-looking logic to determine if the upcoming weekly cycle crosses a month boundary:

```pinescript
f_get_monthly_quarter_label() =>
    string label_text = "Q4"

    int curr_year = year(time, CycleEngine.TIMEZONE_ET)
    int curr_month = month(time, CycleEngine.TIMEZONE_ET)
    int curr_dom = dayofmonth(time, CycleEngine.TIMEZONE_ET)
    int curr_dow = dayofweek(time, CycleEngine.TIMEZONE_ET)
    int curr_hour = hour(time, CycleEngine.TIMEZONE_ET)

    // This function is called at weekly cycle boundaries (Sunday 18:00)
    if curr_dow == dayofweek.sunday and curr_hour == 18
        // Check if this is the 1st of the month (month starts exactly on weekly cycle boundary)
        if curr_dom == 1
            label_text := "Q1"  // Month starts exactly on Sunday 18:00 - this is Q1
        else
            // Calculate next Sunday to see if upcoming weekly cycle crosses month boundary
            int next_sunday_ts = time + (7 * 24 * 60 * 60 * 1000)  // Add 7 days in ms
            int next_sunday_month = month(next_sunday_ts, CycleEngine.TIMEZONE_ET)

            // Check if this weekly cycle will cross into a new month mid-week
            if next_sunday_month != curr_month
                label_text := "Qx"  // This weekly cycle transitions to new month mid-week
            else
                // Both this Sunday and next Sunday are in the same month
                // Calculate normal quarterly position based on week-of-month
                int first_full_week_ts = f_get_first_full_week_start(curr_year, curr_month)
                int curr_time_et = timestamp(CycleEngine.TIMEZONE_ET, curr_year, curr_month, curr_dom, curr_hour, 0)
                int weeks_since = int(math.floor((curr_time_et - first_full_week_ts) / (7 * 24 * 60 * 60 * 1000)))

                if weeks_since == 0
                    label_text := "Q1"  // First full week of month
                else if weeks_since == 1
                    label_text := "Q2"  // Second week
                else if weeks_since == 2
                    label_text := "Q3"  // Third week
                else if weeks_since == 3
                    label_text := "Q4"  // Fourth week
                else
                    label_text := "Qx"  // Partial week(s) after Q4

    label_text
```

**Forward-Looking Logic Explanation:**

The key insight is: when standing at Sunday 18:00, we need to know if the UPCOMING week (the weekly cycle that's STARTING now) will cross into a new month before it ends. If yes, label it "Qx" because it's a partial week.

Example:
- Current time: January 28, Sunday 18:00 (weekly cycle starting)
- Next Sunday: February 4, Sunday 18:00 (weekly cycle ending)
- Months differ → Label "Qx" (this week spans January 28-31 + February 1-4)

#### How V7 Currently Handles Monthly Dividers: The Implementation

**V7's Monthly Weekly Tracking (v7 lines 428-487):**

V7 correctly implements the tracking logic in the `is_new_weekly` block:

```pinescript
// Track monthly weekly dividers with Qx labels (at every Sunday 18:00)
if current_dow == dayofweek.sunday and current_hour == 18
    bool already_tracked = false
    if array.size(monthly_week_divider_bars) > 0
        int last_tracked = array.get(monthly_week_divider_bars, array.size(monthly_week_divider_bars) - 1)
        already_tracked := last_tracked == bar_index

    if not already_tracked
        string monthly_label = f_get_monthly_quarter_label()
        array.push(monthly_week_divider_bars, bar_index)
        array.push(monthly_week_divider_labels, monthly_label)

        // Maintain array size limit
        if array.size(monthly_week_divider_bars) > 100
            array.shift(monthly_week_divider_bars)
            array.shift(monthly_week_divider_labels)

        // Capture monthly quarter starts at Sunday 18:00 (align with weekly cycle starts)
        if monthly_label == "Q1"
            monthly_cycle.current_quarter := 1
            monthly_cycle.q1_start_bar := bar_index
            monthly_cycle.parent_start_bar := bar_index
            monthly_cycle.parent_timestamp := time
            prev_monthly_q := 0  // Reset tracker for Q1→Q2 transition
            // Clear Q2/Q3/Q4 for new cycle
            monthly_cycle.q2_start_bar := na
            monthly_cycle.q3_start_bar := na
            monthly_cycle.q4_start_bar := na

            // Log monthly Q1 divider
            array.push(hist_monthly_q1, bar_index)
            if array.size(hist_monthly_q1) > 100
                array.shift(hist_monthly_q1)

        else if monthly_label == "Q2"
            monthly_cycle.current_quarter := 2
            monthly_cycle.q2_start_bar := bar_index

            // Log monthly Q2 divider
            array.push(hist_monthly_q2, bar_index)
            if array.size(hist_monthly_q2) > 100
                array.shift(hist_monthly_q2)

        // ... Q3, Q4 similar logic
```

**V7's Monthly Label Calculation (v7 lines 106-173):**

V7 implements the forward-looking logic with guard check:

```pinescript
f_get_monthly_quarter_label() =>
    string label_text = "Q4"

    int curr_year = year(time, CycleEngine.TIMEZONE_ET)
    int curr_month = month(time, CycleEngine.TIMEZONE_ET)
    int curr_dom = dayofmonth(time, CycleEngine.TIMEZONE_ET)
    int curr_dow = dayofweek(time, CycleEngine.TIMEZONE_ET)
    int curr_hour = hour(time, CycleEngine.TIMEZONE_ET)

    // Guard: Verify function is called at weekly cycle boundary only
    if not (curr_dow == dayofweek.sunday and curr_hour == 18)
        runtime.error("f_get_monthly_quarter_label() called outside Sunday 18:00 ET boundary")

    // This function is called at weekly cycle boundaries (Sunday 18:00)
    // Determine if THIS weekly cycle (starting now) belongs to Qx

    if curr_dow == dayofweek.sunday and curr_hour == 18
        // Check if this is the 1st of the month (month starts exactly on weekly cycle boundary)
        if curr_dom == 1
            label_text := "Q1"
        else
            // Calculate next Sunday to see if upcoming weekly cycle crosses month boundary
            int next_sunday_ts = time + (7 * 24 * 60 * 60 * 1000)
            int next_sunday_month = month(next_sunday_ts, CycleEngine.TIMEZONE_ET)

            // Check if this weekly cycle will cross into a new month mid-week
            if next_sunday_month != curr_month
                label_text := "Qx"
            else
                // Both this Sunday and next Sunday are in the same month
                // Calculate normal quarterly position
                // ... week-of-month calculation logic

    label_text
```

**V7's Rendering Function (v7 lines 779-800):**

V7 implements a dedicated renderer for monthly weekly dividers:

```pinescript
// Render monthly weekly dividers with Qx labels (special handling for monthly cycle)
f_render_monthly_weekly_dividers() =>
    if show_quarter_dividers and f_should_show_cycle("Monthly") and barstate.islast
        color div_color = color.new(divider_color_universal, 70)

        // Delete old monthly weekly dividers
        while array.size(monthly_week_divider_lines) > 0
            line.delete(array.pop(monthly_week_divider_lines))
        while array.size(monthly_week_divider_labels_vis) > 0
            label.delete(array.pop(monthly_week_divider_labels_vis))

        // Redraw monthly weekly dividers
        if array.size(monthly_week_divider_bars) > 0
            for i = 0 to array.size(monthly_week_divider_bars) - 1
                int bar_pos = array.get(monthly_week_divider_bars, i)
                if (bar_index - bar_pos) <= divider_lookback
                    string label_text = array.get(monthly_week_divider_labels, i)
                    line new_line = line.new(bar_pos, line_bottom, bar_pos, line_top, color=div_color, width=1, style=line.style_dotted, extend=extend.both)
                    array.push(monthly_week_divider_lines, new_line)
                    if show_quarter_labels
                        label new_label = label.new(bar_pos + 1, line_bottom, label_text, style=label.style_label_left, color=color.new(color.white, 100), textcolor=div_color, size=f_get_label_size(label_size), yloc=yloc.price, textalign=text.align_left)
                        array.push(monthly_week_divider_labels_vis, new_label)
```

**THE BUG (SUSPECTED):**

Based on the task description, monthly dividers have "rendering issues at weekly boundaries". Potential root causes:

1. **Label Calculation Issues:** The forward-looking logic may have edge cases causing incorrect Qx labels
2. **Duplicate Prevention Logic:** The `already_tracked` check might prevent some dividers from being logged
3. **Rendering Range Issues:** The `divider_lookback` filter might exclude some monthly dividers
4. **Q1 Detection Issues:** Month start detection might fail when 1st doesn't fall on Sunday

**Most Likely Issue:** The forward-looking logic in `f_get_monthly_quarter_label()` needs to handle edge cases:
- Month starts on Sunday 18:00 (1st = Sunday) → Should be Q1, not Qx
- Last week of month ending on Sunday before 1st → Should be Qx (crosses boundary)
- First week of month starting mid-week → Should be Qx (partial week before first full week)

#### Key Differences Between V6 and V7 Monthly Handling

**V6 Pattern:**
- Calls `f_get_monthly_quarter_label()` at EVERY Sunday 18:00 (v6 line 1628)
- Stores label for each weekly divider (v6 lines 1629-1630)
- Captures Q2/Q3/Q4 quarter starts based on label (v6 lines 1634-1641)
- Renders ALL weekly dividers within range with appropriate labels (v6 rendering function)

**V7 Pattern:**
- Calls `f_get_monthly_quarter_label()` at EVERY Sunday 18:00 (v7 line 436)
- Stores label for each weekly divider (v7 lines 437-438)
- Captures Q1/Q2/Q3/Q4 quarter starts based on label (v7 lines 446-487)
- Renders ALL weekly dividers within range with appropriate labels (v7 line 779-800)

**Architecture is nearly identical!** The bug is likely in label calculation edge cases or rendering logic.

---

### File Locations and Line Ranges

**V7 Implementation (Primary Work File):**
- Path: `D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\01_Core_Cycles\TTL_v7_01_Cycles.pine`
- **Micro Cycle Fix Required:**
  - Lines 566-570: Replace `old_session_q` tracker update with time-based micro quarter calculation
  - Add logic: Calculate elapsed time since `micro_cycle.parent_timestamp`, derive quarter (1-4)
- **Monthly Cycle Investigation Required:**
  - Lines 106-173: `f_get_monthly_quarter_label()` - review edge case handling
  - Lines 428-487: Monthly weekly tracking in `is_new_weekly` block
  - Lines 779-800: `f_render_monthly_weekly_dividers()` rendering function
  - Check for: Label calculation bugs, duplicate prevention issues, rendering range problems

**V6 Reference (Gold Standard):**
- Path: `C:\Users\garic\Downloads\TTL_Fractal_v6_Modular.pine`
- **Micro Cycle Reference:**
  - Lines 2560-2640: Micro reset at session quarter boundaries
  - Lines 2642-2653: Time-based micro quarter calculation (THE PATTERN TO COPY)
  - Lines 2787-2807: Micro Q2/Q3/Q4 transition detection
- **Monthly Cycle Reference:**
  - Lines 1364-1386: `f_get_monthly_quarter_label()` - forward-looking logic
  - Lines 1333-1354: `f_get_first_full_week_start()` helper function
  - Lines 1618-1646: Monthly weekly tracking at Sunday 18:00
  - Lines 4960-4990: Monthly weekly divider rendering (search for "monthly_week_divider")

**CycleEngine Library:**
- Path: `C:\Users\garic\Downloads\TTL_CycleEngine.pine`
- **Relevant Functions:**
  - Lines 332-356: `f_micro_q_index()` - calculates 1-16 range (NOT suitable for reset-based micro cycles)
  - Lines 193-199: `f_is_new_weekly_cycle()` - weekly cycle start detection
  - Lines 302-314: `f_is_new_session_cycle()` - session cycle start detection

---

### Implementation Strategy

#### Micro Cycle Fix (High Confidence)

**Root Cause:** V7 uses `f_micro_q_index()` which returns 1-16 (daily quarter scope), but micro cycles reset every 90 minutes (session quarter scope) and need 1-4 range.

**Solution:** Implement V6's time-based calculation pattern (v6 lines 2642-2653):

```pinescript
// After micro reset block (after line 570), replace generic quarter update:

// Calculate micro quarter based on time elapsed within session quarter (1-4 range)
if not is_new_session_quarter and not na(micro_cycle.parent_timestamp)
    int time_elapsed_ms = time - micro_cycle.parent_timestamp
    float minutes_elapsed = time_elapsed_ms / 60000.0

    // Calculate which micro quarter (1-4) within 90-minute session quarter
    // Each session quarter = 90 minutes = 4 micro quarters of 22.5 minutes each
    int calculated_micro_q = minutes_elapsed < 22.5 ? 1 : minutes_elapsed < 45.0 ? 2 : minutes_elapsed < 67.5 ? 3 : 4
    micro_cycle.current_quarter := calculated_micro_q
else if not is_new_session_quarter
    // Fallback: Use library function (should rarely execute after reset)
    micro_cycle.current_quarter := micro_q
// If is_new_session_quarter, quarter already set to 1 in reset block above
```

**Why This Works:**
- `parent_timestamp` is anchor point set when session quarter starts
- Calculate elapsed minutes since anchor
- Use threshold comparison to get quarter (1-4)
- Inline transition detection (lines 578-608) now works because `current_quarter` stays in 1-4 range
- Q2/Q3/Q4 dividers render correctly

#### Monthly Cycle Fix (Investigation Required)

**Root Cause:** Unknown - need to compare V6 vs V7 label calculation and rendering logic.

**Investigation Steps:**
1. Add debug logging to `f_get_monthly_quarter_label()` - print calculated label at each Sunday 18:00
2. Check edge cases:
   - Month starts on Sunday 18:00 (curr_dom == 1) → Should return "Q1"
   - First week of month with 1st mid-week → Should return "Qx" until first full Sunday
   - Last week of month crossing boundary → Should return "Qx"
3. Verify rendering: Check if `divider_lookback` filter excludes some monthly dividers
4. Compare V6 helper functions: Does V7 correctly implement `f_get_first_full_week_start()`?

**Suspected Fix Locations:**
- Line 106-173: Review `f_get_monthly_quarter_label()` edge case handling
- Line 82-94: Review `f_get_first_full_week_start()` implementation
- Line 428-487: Verify monthly label capture and quarter start assignments
- Line 779-800: Check rendering loop and lookback filter

---

### Testing Validation

**Micro Cycle Testing (M1 Chart):**
1. Find a session quarter boundary (e.g., 19:30 after 18:00 daily start)
2. Verify micro Q1 divider appears at 19:30
3. Verify micro Q2 divider appears at 19:52.5 (22.5 minutes later)
4. Verify micro Q3 divider appears at 20:15 (45 minutes later)
5. Verify micro Q4 divider appears at 20:37.5 (67.5 minutes later)
6. Check debug table: Micro quarter should cycle 1→2→3→4→1 every 90 minutes

**Monthly Cycle Testing (H4 Chart):**
1. Find a month boundary (e.g., January 31 → February 1)
2. Verify Qx dividers appear at Sunday 18:00 before/after month boundary
3. Verify Q1 divider appears at first full Sunday 18:00 of new month
4. Verify Q2/Q3/Q4 dividers appear at subsequent Sunday 18:00 times
5. Check labels match expected week-of-month position

---

### Architectural Context: TTL Cycle Independence Principle

**Critical Reminder:** Each of the 5 fractal cycles (Monthly/Weekly/Daily/Session/Micro) must process in complete isolation. Micro cycle bug demonstrates what happens when this principle is violated:

- Micro used `f_micro_q_index(daily_q, ...)` which couples micro calculation to daily quarter scope
- Should use session quarter scope: `time - micro_cycle.parent_timestamp`
- Breaking independence causes state corruption: micro quarters exceed 1-4 range

**Two-Stage Cycle Detection Pattern:**
- Stage 1: Explicit cycle start blocks handle Q1 detection and initialization
- Stage 2: Quarter transition logic handles Q2/Q3/Q4 detection
- Previous task `h-fix-cycle-boundary-detection-bugs` fixed this pattern for Weekly/Daily/Session
- Micro cycle partially implemented pattern but calculation bug remained
- Monthly cycle uses weekly event-based detection instead of time-based calculation

## User Notes
- User specifically requested: "seek for the best coding solutions while tackling this bug"
- Use code agent for fixes, then verify with code-review agent
- Reference v6 implementation as gold standard for correct behavior
- Focus on micro cycle first (more critical visual bug), then monthly cycle

## Work Log

### 2025-01-08

#### Session Summary
**Status**: INCOMPLETE - Micro cycle Q3/Q4 dividers still not rendering despite multiple fix attempts

#### Fixes Attempted

**1. Time-Based Micro Quarter Calculation (Initial Fix)**
- **Lines changed**: 566-584 (TTL_v7_01_Cycles.pine)
- **What**: Replaced CycleEngine.f_micro_q_index() with V6's time-based calculation pattern
- **Implementation**: Calculate elapsed time since `micro_cycle.parent_timestamp`, use 22.5-minute thresholds to determine quarter (1-4)
- **Expected result**: Micro quarters stay in 1-4 range instead of exceeding to 5-16
- **Actual result**: Q3/Q4 dividers still missing

**2. Monthly Qx Label Calculation (Fixed)**
- **Lines changed**: 106-173 (TTL_v7_01_Cycles.pine)
- **Issues found**:
  - Backward-looking logic using f_is_in_partial_week()
  - First Monday calculation had timezone bug
- **Fixes applied**:
  - Implemented V6's forward-looking logic (check if next Sunday crosses month boundary)
  - Fixed first Monday calculation to use ET timezone consistently
  - Added runtime guard to enforce Sunday 18:00 call restriction
- **Result**: ✅ Monthly Qx labels now calculate correctly

**3. Pause Guard Fix #1 - Add to Calculation (Applied, Then Reverted)**
- **Line changed**: 541
- **What**: Added `and not is_in_pause` to micro quarter calculation
- **Reasoning**: Prevent calculation during transitional pause (17:00-18:00 ET)
- **Result**: Made problem WORSE - prevented Q3/Q4 detection in Session Q4

**4. Tracker Synchronization Fix (Applied)**
- **Line changed**: 588
- **What**: Moved `prev_micro_q` update outside `if not is_in_pause` block
- **Reasoning**: Tracker must update every bar to stay synchronized with `micro_cycle.current_quarter`
- **Result**: Improved synchronization but Q3/Q4 still missing

**5. Pause Guard Fix #2 - Remove from Calculation (Final Attempt)**
- **Line changed**: 542 (reverted fix #3)
- **What**: Removed `and not is_in_pause` from micro quarter calculation
- **Reasoning**: Code review revealed calculation must run continuously, only transition LOGGING should skip pause
- **Result**: INCOMPLETE - User reports Q3/Q4 dividers still missing

#### Issues Discovered

**Micro Cycle Root Causes Identified (Multiple Theories)**
1. **Original bug**: CycleEngine.f_micro_q_index() returns 1-16 range (daily quarter scope) instead of 1-4 (session quarter scope)
2. **Pause timing issue**: Session Q4 (16:30-18:00) overlaps transitional pause (17:00-18:00), causing Q3/Q4 to fall within pause period
3. **Tracker desync**: `prev_micro_q` updates guarded by pause, causing permanent desynchronization
4. **Calculation skip**: Pause guard on calculation line prevents quarter tracking during pause

**Monthly Cycle Issues (Resolved)**
- Backward-looking partial week detection
- Timezone inconsistency in first Monday calculation
- Missing runtime guard for function call restriction

#### Current Status

**Micro Cycle**: ❌ NOT WORKING
- Only Q1 and Q2 dividers visible across entire chart
- Q3 and Q4 dividers completely missing
- Three separate fixes applied, none successful
- Root cause remains elusive despite multiple code reviews

**Monthly Cycle**: ✅ WORKING
- Qx labels calculate correctly with forward-looking logic
- First Monday calculation uses correct ET timezone
- Runtime guard prevents misuse

#### Remaining Work

**Critical Investigation Needed:**
1. Add extensive debug logging to micro cycle calculation
   - Log `minutes_elapsed` values at each bar
   - Log `calculated_micro_q` output
   - Log `is_new_session_quarter` boolean
   - Log `is_in_pause` boolean
   - Log when transitions fire (or fail to fire)

2. Verify session quarter stability
   - Confirm `session_q` changes every 90 minutes (not more frequently)
   - Verify `old_session_q` updates correctly after reset
   - Check if `is_new_session` triggers incorrectly

3. Test ternary operator precedence
   - Refactor line 549 to explicit if/else blocks
   - Verify threshold comparisons work for 45.0 and 67.5 values

4. Check rendering vs calculation
   - Verify if Q3/Q4 values appear in `hist_micro_q3` and `hist_micro_q4` arrays
   - Check if rendering function iterates all 4 quarter arrays
   - Verify no filters preventing Q3/Q4 display

5. Compare working sessions to broken sessions
   - Identify which session quarters successfully show Q3/Q4 (if any)
   - Determine pattern: Is it ALL sessions or just Session Q4?
   - If only Session Q4 broken, pause is root cause
   - If ALL sessions broken, calculation logic has deeper issue

#### Next Steps
1. User to provide specific chart range where Q3/Q4 are missing
2. Add comprehensive debug table showing all micro cycle variables
3. Test on multiple timeframes (M1, M5) covering different session quarters
4. If Session Q1-Q3 work but Q4 doesn't, pause is confirmed root cause
5. If no sessions work, calculation logic needs complete review against V6 implementation

---

### Discovered During Implementation
[Date: 2025-01-08]

During today's implementation session, we discovered critical timing details about the transitional pause and its interaction with micro cycles that weren't fully documented in the original context:

#### Pause Timing: 17:00-17:59:59 ET (Full Hour, Not 30-Minute Window)

The transitional pause is **the entire 17:00 hour** (17:00:00-17:59:59 ET), not a 30-minute window as might be assumed. This was confirmed by examining the CycleEngine pause detection logic:

```pinescript
// Line 85 in CycleEngine
h == 17  // Pause is entire 17:00 hour
```

**Why This Matters**: This timing affects Session Q4 micro quarter calculations because Session Q4 runs from 16:30-18:00, meaning half of Session Q4 (17:00-18:00) overlaps with the pause period.

#### Session Q4 Unique Overlap with Pause

Session Q4 (16:30-18:00) is the only session quarter that overlaps with the transitional pause. This creates a unique challenge for micro cycle quarter detection:

**Micro Quarter Timing in Session Q4:**
- Micro Q1: 16:30-16:52:30 ✓ (before pause begins)
- Micro Q2: 16:52:30-17:15:00 ⚠️ (starts before pause, ends during pause)
- Micro Q3: 17:15:00-17:37:30 ✗ (entirely within 17:00-18:00 pause)
- Micro Q4: 17:37:30-18:00:00 ✗ (entirely within 17:00-18:00 pause)

**Impact**: If calculation or transition detection is guarded by `is_in_pause`, micro cycles will stall at Q2 during Session Q4 and never reach Q3/Q4 before the next session quarter reset at 18:00.

#### Architectural Principle: Calculation vs Logging During Pause

A critical separation of concerns was discovered: **quarter calculation must run continuously to track state**, while only **transition logging should be skipped during pause**.

**Why Both Are Needed:**
- **Calculation** = Determines which quarter (1-4) based on elapsed time → Runs every bar to maintain state tracking
- **Transition Logging** = Records divider positions in historical arrays → Skips during pause to avoid invalid entries

**Code Pattern:**
```pinescript
// Quarter Calculation: NO pause guard (runs continuously)
if not is_new_session_quarter and not na(micro_cycle.parent_timestamp)
    int time_elapsed_ms = time - micro_cycle.parent_timestamp
    float minutes_elapsed = time_elapsed_ms / 60000.0
    int calculated_micro_q = minutes_elapsed < 22.5 ? 1 : ... : 4
    micro_cycle.current_quarter := calculated_micro_q

// Transition Logging: HAS pause guard (skips during pause)
if not is_in_pause
    if micro_cycle.current_quarter == 2 and prev_micro_q == 1
        array.push(hist_micro_q2, bar_index)  // Log divider
```

**What We Learned**: Early fix attempts added `and not is_in_pause` to the calculation line, which prevented state tracking during pause. This caused micro cycles to freeze at Q2 during Session Q4. The correct pattern is to guard only the logging, not the calculation.

#### Unresolved Mystery: Q3/Q4 Still Missing Despite Fixes

Despite applying three separate fixes that correctly match V6's architectural patterns:
1. Time-based calculation using `parent_timestamp` anchor
2. Tracker synchronization (moved `prev_micro_q` update outside pause guard)
3. Continuous calculation (removed pause guard from calculation logic)

...the micro Q3/Q4 dividers still don't render. This suggests there's a deeper issue not yet discovered:

**Theories Under Investigation:**
- Session quarter stability: Is `session_q` changing more frequently than expected?
- Ternary operator precedence: Does the threshold chain `minutes_elapsed < 22.5 ? 1 : ... : 4` have evaluation issues?
- Rendering vs calculation: Are Q3/Q4 values calculated correctly but not rendering?
- Timeframe-specific issue: Does the bug only affect certain timeframes or session quarters?

**Critical Debug Points for Next Session:**
1. Add comprehensive debug table showing `minutes_elapsed`, `calculated_micro_q`, `session_q`, `old_session_q`, `is_new_session_quarter`, and all boolean flags
2. Test across multiple session quarters (Q1/Q2/Q3/Q4) to determine if pause overlap is the only affected period
3. Verify if historical arrays (`hist_micro_q3`, `hist_micro_q4`) contain entries that aren't rendering
4. Refactor ternary operator to explicit if/else blocks to rule out precedence issues
