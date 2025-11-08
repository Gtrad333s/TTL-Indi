---
name: h-fix-cycle-boundary-detection-bugs
branch: fix/h-fix-cycle-boundary-detection-bugs
status: pending
created: 2025-01-07
submodules: [TTL_v7_Rebuild]
---

# Fix Cycle Boundary Detection Bugs

## Problem/Goal
The TTL v7 cycle detection logic has several critical bugs where quarter dividers are not being drawn at the correct cycle boundaries:

1. **Monthly cycle partial week (Qx)** - Missing divider for partial weeks at month start/end
2. **Weekly cycle Thursday 6pm (Qx)** - Missing divider for partial period (Thu 6pm - Sun 6pm)
3. **Weekly Q1 not drawing at Sunday 6pm** - Cycle start detection failing to trigger Q1 divider
4. **Monthly quarters not starting at Sunday 6pm** - Monthly quarter boundaries should align with weekly starts
5. **Session Q1 not drawing at daily quarter transitions** - Session cycle start not detected when daily quarter changes
6. **Micro cycle stopping prematurely** - Micro cycles not continuing beyond first quarter

## Success Criteria

**Visual Verification:**
- [ ] Monthly Qx dividers appear at partial weeks (start/end of month)
- [ ] Weekly Qx dividers appear at Thursday 6pm (partial period before weekly start)
- [ ] Weekly Q1 dividers render at Sunday 6pm (weekly cycle start)
- [ ] Monthly quarter dividers align with Sunday 6pm weekly starts
- [ ] Session Q1 dividers render at daily quarter transitions (18:00, 00:00, 06:00, 12:00)
- [ ] Micro cycle dividers continue through all quarters and reset properly at session quarter transitions (VERIFIED WORKING)

**Code Verification:**
- [ ] Cycle start detection logic properly triggers for all 5 cycles
- [ ] Quarter transition detection doesn't skip Q1 starts
- [ ] Qx label logic implemented for monthly and weekly partial periods
- [ ] Historical divider tracking includes Qx positions

**Testing:**
- [ ] Visual validation on all 5 timeframes (M1, M5, M15, H1, H4)
- [ ] Verification at actual boundary times (Sunday 6pm, Thursday 6pm, daily quarter starts)

## Context Manifest

### CRITICAL ARCHITECTURAL UNDERSTANDING: Two-Stage Cycle Detection Pattern

V6 uses a sophisticated two-stage pattern that V7 has incorrectly collapsed into one stage. Understanding this distinction is KEY to fixing all bugs:

**STAGE 1: Cycle Start Detection (is_new_* functions)**
These functions detect when a NEW CYCLE STARTS - the beginning of Q1. They are SEPARATE from quarter transition logic:
- `is_new_weekly` - Triggers at Sunday 18:00 ET (weekly Q1 start)
- `is_new_daily` - Triggers at 18:00 ET any day (daily Q1 start)
- `is_new_session` - Triggers at session start times (18:00, 00:00, 06:00, 12:00)
- `is_new_micro` - Triggers when micro_q changes value

**STAGE 2: Quarter Transition Detection (current_q vs prev_q comparison)**
This happens EVERY BAR and detects when we move between quarters within a cycle (Q1→Q2, Q2→Q3, Q3→Q4, Q4→Q1).

**THE CRITICAL BUG IN V7**:
The `f_process_cycle_quarters()` function ONLY checks `current_q != prev_q` transitions. On the VERY FIRST BAR of a new cycle, `prev_q` is already set to the previous cycle's Q4 value. So when Q1 starts:
- `current_q = 1` (from the quarter index function)
- `prev_q = 1` (initialized to 1, or left at 1 from previous cycle's Q1 end)
- The condition `current_q == 1 and prev_q != 1` FAILS!
- No Q1 divider is drawn, no Q1 start is recorded

**V6's Solution**:
V6 uses the `is_new_*` cycle detection functions to EXPLICITLY handle Q1 starts OUTSIDE of the quarter transition logic. Look at lines 1617-1718 in v6:

```pinescript
if is_new_weekly
    // Log Weekly Q1 historical divider at new weekly cycle start
    if not CycleEngine.f_is_in_transitional_pause()
        array.push(hist_weekly_q1_bars, bar_index)
        // ...

    weekly_cycle.current_quarter := 1
    weekly_cycle.parent_start_bar := bar_index
    weekly_cycle.q1_start_bar := bar_index
    // ... clear Q1 arrays, reset state
```

Then SEPARATELY, V6 handles Q2/Q3/Q4 transitions using the `prev_q` comparison pattern (lines 1820-1989).

---

### How V6 Handles Cycle Starts and Q1 Dividers

#### Weekly Cycle Start (Sunday 18:00 ET)

**Detection Logic (CycleEngine line 193-199)**:
```pinescript
export f_is_new_weekly_cycle() =>
    int dow_now  = dayofweek(time, TIMEZONE_ET)
    int dow_prev = dayofweek(time[1], TIMEZONE_ET)
    int h_now    = f_get_hour_et()
    int h_prev   = hour(time[1], TIMEZONE_ET)
    (dow_now == dayofweek.sunday and h_now == 18) and not (dow_prev == dayofweek.sunday and h_prev == 18)
```

This function returns `true` ONLY on the first bar where we enter Sunday 18:00. It checks both current AND previous bar to ensure we don't trigger multiple times.

**V6 Usage (v6 lines 1617-1718)**:
```pinescript
if is_new_weekly
    // Explicitly log Q1 divider at cycle start
    if not CycleEngine.f_is_in_transitional_pause()
        array.push(hist_weekly_q1_bars, bar_index)
        if array.size(hist_weekly_q1_bars) > 10
            array.shift(hist_weekly_q1_bars)

    // Set cycle properties
    weekly_cycle.current_quarter := 1
    weekly_cycle.parent_start_bar := bar_index
    weekly_cycle.parent_timestamp := time
    weekly_cycle.TO_price := na  // TO captured at Q2 start, not Q1
    weekly_cycle.TO_bar := na

    // Initialize Q1 data collection
    array.clear(weekly_cycle.q1_opens)
    array.clear(weekly_cycle.q1_highs)
    array.clear(weekly_cycle.q1_lows)
    array.clear(weekly_cycle.q1_closes)
    weekly_cycle.q1_complete := false
    weekly_cycle.q1_start_bar := bar_index
else
    // If not new cycle, just update current quarter from calculation
    weekly_cycle.current_quarter := weekly_q
```

**Key Insight**: V6 uses the `is_new_weekly` flag to handle Q1 start explicitly. It doesn't rely on `prev_weekly_q` comparison for Q1 detection. The Q1 divider is added DIRECTLY in the `is_new_weekly` block, not in the quarter transition logic.

#### Daily Cycle Start (18:00 ET any day)

**Detection Logic (CycleEngine lines 234-242)**:
```pinescript
export f_is_new_daily_cycle() =>
    h_now = f_get_hour_et()
    h_prev = hour(time[1], TIMEZONE_ET)
    if bar_index == 0
        h_now == 18
    else
        (h_now == 18 and h_prev != 18)
```

**V6 Usage (v6 lines 2018-2108)**:
```pinescript
if is_new_daily
    // ... archive previous cycle's TO

    daily_cycle.current_quarter := 1
    daily_cycle.parent_start_bar := bar_index
    daily_cycle.parent_timestamp := time
    daily_cycle.TO_price := na
    daily_cycle.TO_bar := na

    array.clear(daily_cycle.q1_opens)
    // ... clear Q1 arrays
    daily_cycle.q1_complete := false
    daily_cycle.q1_start_bar := bar_index
else
    daily_cycle.current_quarter := daily_q
```

**Note**: Daily Q1 dividers are logged at SESSION cycle starts (see next section), because daily quarters align with session starts (18:00, 00:00, 06:00, 12:00).

#### Session Cycle Start (Daily Quarter Boundaries: 18:00, 00:00, 06:00, 12:00)

**Detection Logic (CycleEngine lines 302-314)**:
```pinescript
export f_is_new_session_cycle(int daily_q) =>
    h = f_get_hour_et()
    m = f_get_minute_et()
    h_prev = hour(time[1], TIMEZONE_ET)
    m_prev = minute(time[1], TIMEZONE_ET)

    curr_session_q = f_session_q_index_within_daily_quarter(daily_q, h, m)
    prev_session_q = f_session_q_index_within_daily_quarter(daily_q, h_prev, m_prev)

    (curr_session_q == 1 and prev_session_q != 1)
```

This detects when we transition into session Q1 (which happens at daily quarter boundaries).

**V6 Usage (v6 lines 2369-2437)**:
```pinescript
if is_new_session
    // ... archive previous cycle's TO

    // CRITICAL: Log Daily Q1/Q2/Q3/Q4 dividers based on which daily quarter we're in
    if not CycleEngine.f_is_in_transitional_pause()
        if daily_cycle.current_quarter == 1
            array.push(hist_daily_q1_bars, bar_index)  // Daily Q1 at 18:00
        else if daily_cycle.current_quarter == 2
            array.push(hist_daily_q2_bars, bar_index)  // Daily Q2 at 00:00
        else if daily_cycle.current_quarter == 3
            array.push(hist_daily_q3_bars, bar_index)  // Daily Q3 at 06:00
        else if daily_cycle.current_quarter == 4
            array.push(hist_daily_q4_bars, bar_index)  // Daily Q4 at 12:00

    // Log Session Q1 divider at session start
    if not CycleEngine.f_is_in_transitional_pause()
        array.push(hist_session_q1_bars, bar_index)

    current_session_number := daily_q  // Track which session (1-4)

    session_cycle.current_quarter := 1
    session_cycle.parent_start_bar := bar_index
    session_cycle.parent_timestamp := time
    session_cycle.TO_price := na
    session_cycle.TO_bar := na

    array.clear(session_cycle.q1_opens)
    // ... clear arrays
    session_cycle.q1_complete := false
    session_cycle.q1_start_bar := bar_index
else
    session_cycle.current_quarter := CycleEngine.f_session_q_index_within_daily_quarter(daily_q, current_hour, current_minute)
```

**Key Insight**: Session cycle starts trigger BOTH Session Q1 dividers AND Daily Q1/Q2/Q3/Q4 dividers (because daily quarters align with session starts). This is why V6 doesn't need separate daily quarter transition tracking - it leverages the fractal hierarchy.

#### Micro Cycle Reset at Session Quarter Boundaries

**V6 Usage (v6 lines 2565-2619)**:
```pinescript
// Detect when a new 90-minute session quarter starts
bool is_new_session_quarter = session_cycle.current_quarter != old_session_q or is_new_session

// Reset micro quarters when NEW 90-MINUTE SESSION QUARTER starts
if is_new_session_quarter
    // ... archive previous micro TO

    // Reset ALL micro quarter tracking for new session quarter
    micro_cycle.current_quarter := 1
    micro_cycle.parent_start_bar := bar_index
    micro_cycle.q1_start_bar := bar_index

    // CRITICAL: Reset prev_micro_q to 0 so first transition works
    prev_micro_q := 0

    // Reset all quarter start/end bars
    micro_cycle.q2_start_bar := na
    micro_cycle.q3_start_bar := na
    micro_cycle.q4_start_bar := na
    // ... reset DFR values, etc.
```

**Key Insight**: Micro cycles reset every 90 minutes (at session quarter boundaries). The `prev_micro_q := 0` initialization is CRITICAL - it ensures the first Q1→Q2 transition will be detected properly.

---

### How V6 Handles Quarter Transitions (Q2, Q3, Q4)

Once Q1 is established via cycle start detection, V6 uses the `prev_q` comparison pattern for Q2/Q3/Q4 transitions.

#### Weekly Q1→Q2 Transition (Monday 18:00 ET)

**V6 Logic (v6 lines 1818-1831)**:
```pinescript
var int prev_weekly_q = 1
if weekly_cycle.current_quarter == 2 and prev_weekly_q == 1
    weekly_cycle.q2_start_bar := bar_index
    weekly_cycle.q2_end_bar := na

    // TRUE OPEN: Captured PRECISELY at Q2 start transition (Monday 18:00 ET)
    weekly_cycle.TO_price := open
    weekly_cycle.TO_bar := bar_index

    // Log Weekly Q2 historical divider at transition
    if not CycleEngine.f_is_in_transitional_pause()
        array.push(hist_weekly_q2_bars, bar_index)
        if array.size(hist_weekly_q2_bars) > 10
            array.shift(hist_weekly_q2_bars)
```

**At end of transition processing**:
```pinescript
prev_weekly_q := weekly_cycle.current_quarter  // Update tracker for next bar
```

This pattern repeats for Q2→Q3 (lines 1888-1896) and Q3→Q4 (lines 1951-1959).

---

### How V6 Handles Qx (Partial Period) Dividers

Qx dividers mark "partial" periods that don't fit the standard Q1-Q4 pattern:
- **Monthly Qx**: Partial weeks at month start/end (before first full Sunday or after last full Wednesday)
- **Weekly Qx**: Thursday 18:00 - Sunday 18:00 (partial period before weekly cycle starts)

#### Monthly Qx Detection

**V6 Logic (v6 lines 1330-1386)**:
```pinescript
// Get the first Sunday 18:00 ET of the month (first full week start)
f_get_first_full_week_start(int year_val, int month_val) =>
    // Find first Monday of month
    int first_monday_dom = CycleEngine.f_first_monday_dom(year_val, month_val)

    // First full week starts on Sunday before first Monday
    int sunday_dom = first_monday_dom - 1
    if sunday_dom < 1
        // First Monday is day 1, so Sunday is in previous month
        // ... calculate previous month's last Sunday
    else
        timestamp(CycleEngine.TIMEZONE_ET, year_val, month_val, sunday_dom, 18, 0)

// Check if currently in a partial week (before first full week of month)
f_is_in_partial_week() =>
    int curr_year = year(time, CycleEngine.TIMEZONE_ET)
    int curr_month = month(time, CycleEngine.TIMEZONE_ET)
    int first_full_week_ts = f_get_first_full_week_start(curr_year, curr_month)

    // If current time is before first full week start, we're in partial week
    time < first_full_week_ts

// Get monthly quarter label including Qx for partial weeks
f_get_monthly_quarter_label() =>
    string label_text = "Q4"

    if f_is_in_partial_week()
        label_text := "Qx"  // Partial week at start of month
    else
        // Calculate which week we're in since first full week
        int curr_year = year(time, CycleEngine.TIMEZONE_ET)
        int curr_month = month(time, CycleEngine.TIMEZONE_ET)
        int first_full_week_ts = f_get_first_full_week_start(curr_year, curr_month)
        int weeks_since = int(math.floor((time - first_full_week_ts) / CycleEngine.MS_WEEK))

        switch weeks_since
            0 => label_text := "Q1"  // Week 0 (first full week)
            1 => label_text := "Q2"  // Week 1
            2 => label_text := "Q3"  // Week 2
            3 => label_text := "Q4"  // Week 3
            => label_text := "Qx"    // Partial week(s) after Q4 at end of month

    label_text
```

**V6 Monthly Qx Tracking (v6 lines 1618-1646)**:
```pinescript
if is_new_weekly
    // Track monthly weekly dividers with Qx labels at every weekly cycle start (Sunday 18:00 ET only)
    if current_dow == dayofweek.sunday and current_hour == 18
        bool already_tracked = false
        if array.size(monthly_week_divider_bars) > 0
            int last_tracked = array.get(monthly_week_divider_bars, array.size(monthly_week_divider_bars) - 1)
            already_tracked := last_tracked == bar_index

        if not already_tracked
            current_monthly_week_label := f_get_monthly_quarter_label()
            array.push(monthly_week_divider_bars, bar_index)
            array.push(monthly_week_divider_labels, current_monthly_week_label)

            // CAPTURE MONTHLY QUARTER STARTS AT SUNDAY 18:00 ET
            if current_monthly_week_label == "Q2" and na(monthly_cycle.TO_bar)
                monthly_cycle.q2_start_bar := bar_index
                monthly_cycle.TO_price := open
                monthly_cycle.TO_bar := bar_index
            else if current_monthly_week_label == "Q3" and na(monthly_cycle.q3_start_bar)
                monthly_cycle.q3_start_bar := bar_index
            else if current_monthly_week_label == "Q4" and na(monthly_cycle.q4_start_bar)
                monthly_cycle.q4_start_bar := bar_index
```

**Key Insight**: V6 tracks monthly weekly dividers at EVERY Sunday 18:00, labeling them with Q1/Q2/Q3/Q4/Qx based on which week of the month it is. Monthly quarter boundaries are aligned to weekly cycle starts.

#### Weekly Qx Detection (Thursday 18:00)

**V6 Logic (v6 lines 1389-1408)**:
```pinescript
// Get weekly quarter label including Qx for partial period (Thursday-Sunday)
f_get_weekly_quarter_label(int dow, int h) =>
    string label_text = "Q4"

    if (dow == dayofweek.sunday and h >= 18) or (dow == dayofweek.monday and h < 18)
        label_text := "Q1"  // Sunday 18:00 - Monday 18:00
    else if (dow == dayofweek.monday and h >= 18) or (dow == dayofweek.tuesday and h < 18)
        label_text := "Q2"  // Monday 18:00 - Tuesday 18:00
    else if (dow == dayofweek.tuesday and h >= 18) or (dow == dayofweek.wednesday and h < 18)
        label_text := "Q3"  // Tuesday 18:00 - Wednesday 18:00
    else if (dow == dayofweek.wednesday and h >= 18) or (dow == dayofweek.thursday and h < 18)
        label_text := "Q4"  // Wednesday 18:00 - Thursday 18:00
    else if dow >= dayofweek.thursday and (dow != dayofweek.sunday or h < 18)
        label_text := "Qx"  // Thursday 18:00 - Sunday 18:00 (partial period)

    label_text
```

**V6 Weekly Qx Tracking (v6 lines 1991-2010)**:
```pinescript
// Detect Qx period start (Thursday 18:00) for historical tracking
if current_dow == dayofweek.thursday and current_hour == 18
    // Check if we just entered Thursday 18:00 (wasn't Thursday 18:00 last bar)
    int prev_dow = dayofweek(time[1], CycleEngine.TIMEZONE_ET)
    int prev_hour = hour(time[1], CycleEngine.TIMEZONE_ET)
    if not (prev_dow == dayofweek.thursday and prev_hour == 18)
        bool already_tracked_qx = false
        if array.size(hist_weekly_qx_bars) > 0
            int last_tracked_qx = array.get(hist_weekly_qx_bars, array.size(hist_weekly_qx_bars) - 1)
            already_tracked_qx := last_tracked_qx == bar_index

        // Log Weekly Qx historical divider at Thursday 18:00
        if not already_tracked_qx and not CycleEngine.f_is_in_transitional_pause()
            array.push(hist_weekly_qx_bars, bar_index)
            if array.size(hist_weekly_qx_bars) > 10
                array.shift(hist_weekly_qx_bars)
```

**Key Insight**: Weekly Qx is detected SEPARATELY from the Q1-Q4 cycle logic. It's a standalone check for Thursday 18:00, tracking this "partial period" before the weekly cycle restarts on Sunday 18:00.

---

### Root Cause Analysis of Each Bug

Now that we understand V6's two-stage pattern, let's diagnose each bug:

#### Bug 1: Monthly Cycle Partial Week (Qx) Dividers Missing

**Root Cause**: V7 doesn't implement monthly weekly tracking at all. V6 tracks monthly quarters by logging dividers at EVERY Sunday 18:00 and labeling them Q1/Q2/Q3/Q4/Qx based on week-of-month calculation.

**V7 Current Behavior**: V7 calculates `monthly_q` using `f_monthly_q_index_time()` but never tracks individual weekly dividers within the month. It only tracks Q1/Q2/Q3/Q4 quarterly boundaries, missing the Qx partial weeks.

**Fix Required**:
1. Implement `f_get_monthly_quarter_label()` logic (detect partial weeks at month start/end)
2. In `is_new_weekly` block, track monthly weekly dividers with Qx labels
3. Add storage arrays: `var array<int> monthly_week_divider_bars`, `var array<string> monthly_week_divider_labels`
4. Render these dividers separately from standard Q1-Q4 dividers

#### Bug 2: Weekly Cycle Thursday 6pm (Qx) Dividers Missing

**Root Cause**: V7 doesn't implement weekly Qx detection. V6 explicitly checks for Thursday 18:00 and logs Qx dividers.

**V7 Current Behavior**: V7 only tracks Q1-Q4 quarters (Sunday-Monday-Tuesday-Wednesday 18:00 boundaries). The Thursday-Sunday period is ignored.

**Fix Required**:
1. Add explicit Thursday 18:00 detection (similar to v6 lines 1991-2010)
2. Add storage arrays: `var array<int> hist_weekly_qx_bars`
3. Add rendering arrays: `var array<line> hist_weekly_qx_lines`, `var array<label> hist_weekly_qx_labels`
4. Log Qx divider when entering Thursday 18:00

#### Bug 3: Weekly Q1 Not Drawing at Sunday 6pm Cycle Start

**Root Cause**: V7's `f_process_cycle_quarters()` logic fails to detect Q1 starts because `prev_weekly_q` is initialized to 1. When `is_new_weekly` triggers (Sunday 18:00), the function checks `current_q == 1 and prev_q != 1`, which fails because `prev_q == 1`.

**V7 Current Behavior**:
- `is_new_weekly` detection exists in CycleEngine (line 193-199)
- V7 imports and uses it, but ONLY to update `weekly_cycle.current_quarter` generically
- V7 never explicitly handles Q1 start in the `is_new_weekly` block
- All divider logic is delegated to `f_process_cycle_quarters()`, which can't detect Q1

**Fix Required**:
1. Add explicit Q1 divider tracking in a dedicated cycle start block (before calling `f_process_cycle_quarters`)
2. When `is_new_weekly` triggers, log Q1 divider DIRECTLY:
   ```pinescript
   if is_new_weekly and not is_in_pause
       array.push(hist_weekly_q1, bar_index)
       weekly_cycle.q1_start_bar := bar_index
       weekly_cycle.parent_start_bar := bar_index
       weekly_cycle.parent_timestamp := time
       prev_weekly_q := 0  // Reset to ensure Q1→Q2 transition works
   ```
3. Keep `f_process_cycle_quarters()` for Q2/Q3/Q4 transitions only

#### Bug 4: Monthly Quarters Not Starting at Sunday 6pm

**Root Cause**: V7 calculates monthly quarters using time-based percentage (`f_monthly_q_index_time`), which doesn't align to weekly cycle starts. V6 explicitly captures monthly Q2/Q3/Q4 starts at Sunday 18:00 when the weekly cycle label matches.

**V7 Current Behavior**: Monthly quarter calculation is independent of weekly boundaries. It divides the month into 4 equal time periods, which means quarter transitions can happen mid-week.

**Fix Required**:
1. Change monthly quarter logic to align with weekly cycle starts (like v6)
2. In `is_new_weekly` block, check `f_get_monthly_quarter_label()` result
3. When label is "Q2", set `monthly_cycle.q2_start_bar` and capture TO
4. When label is "Q3", set `monthly_cycle.q3_start_bar`
5. When label is "Q4", set `monthly_cycle.q4_start_bar`
6. This ensures monthly quarters always start on Sunday 18:00

#### Bug 5: Session Q1 Not Drawing at Daily Quarter Transitions

**Root Cause**: Same as Bug 3. V7's `f_process_cycle_quarters()` can't detect Q1 starts because `prev_session_q` is initialized to 1.

**V7 Current Behavior**:
- `is_new_session` detection exists in CycleEngine (lines 302-314)
- V7 calls `f_process_cycle_quarters()` for sessions, but Q1 detection fails

**Fix Required**:
1. Add explicit handling for `is_new_session` detection (currently not called in V7!)
2. When `is_new_session` triggers, log Session Q1 divider DIRECTLY:
   ```pinescript
   if is_new_session and not is_in_pause
       array.push(hist_session_q1, bar_index)
       session_cycle.q1_start_bar := bar_index
       prev_session_q := 0  // Reset tracker
   ```
3. Also log Daily Q1/Q2/Q3/Q4 dividers in this block (based on `daily_q` value)
4. Keep `f_process_cycle_quarters()` for Session Q2/Q3/Q4 transitions

#### Bug 6: Micro Cycle Dividers Continuing Issues

**Root Cause**: According to task description, micro cycle resets at session quarter transitions ARE working. The issue is likely that Q1 dividers aren't rendering due to the same `prev_micro_q` initialization problem.

**V7 Current Behavior**: Micro cycles reset at session quarter boundaries (which is correct), but Q1 dividers may not render.

**Fix Required**:
1. Verify micro cycle reset logic matches v6 pattern (lines 2565-2619)
2. Ensure `prev_micro_q := 0` reset happens at session quarter boundaries
3. Add explicit Q1 divider tracking when micro cycle resets
4. Potentially add micro cycle start detection separate from quarter transition logic

---

### V7 Architecture Analysis

**Current V7 Implementation (TTL_v7_01_Cycles.pine)**:

**Cycle Detection (lines 260-289)**:
```pinescript
// Get current time components (ET timezone)
current_hour = CycleEngine.f_get_hour_et()
current_dow = CycleEngine.f_get_dow_et()
current_minute = CycleEngine.f_get_minute_et()
is_in_pause = CycleEngine.f_is_in_transitional_pause()

// Calculate all quarters (every bar)
int monthly_q = CycleEngine.f_monthly_q_index_time(time, anchor_curr, anchor_next)
int weekly_q = CycleEngine.f_weekly_q_index(current_dow, current_hour)
int daily_q = CycleEngine.f_daily_q_index(current_hour)
int session_q = CycleEngine.f_session_q_index_within_daily_quarter(daily_q, current_hour, current_minute)
int micro_q = CycleEngine.f_micro_q_index(daily_q, current_hour, current_minute)
```

**Problem**: V7 calculates quarters but never calls the cycle start detection functions (`is_new_weekly`, `is_new_daily`, `is_new_session`).

**Quarter Transition Processing (lines 292-308)**:
```pinescript
if not is_in_pause
    // Update cycle current quarters
    monthly_cycle.current_quarter := monthly_q
    weekly_cycle.current_quarter := weekly_q
    daily_cycle.current_quarter := daily_q
    session_cycle.current_quarter := session_q
    micro_cycle.current_quarter := micro_q

    // Process transitions for all cycles
    prev_monthly_q := f_process_cycle_quarters(monthly_cycle, monthly_q, prev_monthly_q, ...)
    prev_weekly_q := f_process_cycle_quarters(weekly_cycle, weekly_q, prev_weekly_q, ...)
    prev_daily_q := f_process_cycle_quarters(daily_cycle, daily_q, prev_daily_q, ...)
    prev_session_q := f_process_cycle_quarters(session_cycle, session_q, prev_session_q, ...)
    prev_micro_q := f_process_cycle_quarters(micro_cycle, micro_q, prev_micro_q, ...)
```

**Problem**: This relies entirely on `f_process_cycle_quarters()`, which can't detect Q1 starts.

**f_process_cycle_quarters() Logic (lines 196-254)**:
```pinescript
f_process_cycle_quarters(Types.FractalCycle cycle, int current_q, int prev_q, ...) =>
    // Detect quarter transitions
    bool is_q1_start = current_q == 1 and prev_q != 1  // FAILS for new cycles!
    bool is_q2_start = current_q == 2 and prev_q == 1
    bool is_q3_start = current_q == 3 and prev_q == 2
    bool is_q4_start = current_q == 4 and prev_q == 3

    // Q1 Start (new cycle)
    if is_q1_start
        cycle.q1_start_bar := bar_index
        cycle.parent_start_bar := bar_index
        // ... archive previous Q1

    // ... Q2/Q3/Q4 logic

    // Return updated previous quarter tracker
    current_q
```

**Problem**: The condition `current_q == 1 and prev_q != 1` can never be true on the first bar of a new cycle because:
1. First time through: `prev_q` is initialized to 1 (line 162: `var int prev_weekly_q = 1`)
2. Returning from Q4: When Q4→Q1 transition happens, this should work, but only if we're transitioning within the same logical cycle tracking
3. Fresh cycle start: When `is_new_weekly` triggers, we need to FORCE Q1 recognition, not rely on `prev_q` comparison

---

### Implementation Strategy

To fix all bugs, V7 needs to adopt V6's two-stage pattern:

**Stage 1: Add Explicit Cycle Start Detection Blocks**

Before the current quarter processing logic, add:

```pinescript
// ────────────────────────────────────────────────────────────────────────────
// EXPLICIT CYCLE START DETECTION (Stage 1)
// ────────────────────────────────────────────────────────────────────────────

// Weekly Cycle Start (Sunday 18:00)
bool is_new_weekly = CycleEngine.f_is_new_weekly_cycle()
if is_new_weekly and not is_in_pause
    // Log Weekly Q1 divider
    array.push(hist_weekly_q1, bar_index)
    if array.size(hist_weekly_q1) > 100
        array.shift(hist_weekly_q1)

    // Initialize weekly cycle
    weekly_cycle.current_quarter := 1
    weekly_cycle.q1_start_bar := bar_index
    weekly_cycle.parent_start_bar := bar_index
    weekly_cycle.parent_timestamp := time
    prev_weekly_q := 0  // Reset to ensure Q1→Q2 transition works

    // Track monthly weekly dividers with Qx labels
    if current_dow == dayofweek.sunday and current_hour == 18
        string monthly_label = f_get_monthly_quarter_label()
        array.push(monthly_week_divider_bars, bar_index)
        array.push(monthly_week_divider_labels, monthly_label)

        // Capture monthly quarter starts at Sunday 18:00
        if monthly_label == "Q2" and na(monthly_cycle.TO_bar)
            monthly_cycle.q2_start_bar := bar_index
            monthly_cycle.TO_price := open
            monthly_cycle.TO_bar := bar_index
        // ... Q3, Q4 similar logic

// Daily Cycle Start (18:00 any day)
bool is_new_daily = CycleEngine.f_is_new_daily_cycle()
if is_new_daily and not is_in_pause
    daily_cycle.current_quarter := 1
    daily_cycle.q1_start_bar := bar_index
    daily_cycle.parent_start_bar := bar_index
    daily_cycle.parent_timestamp := time
    prev_daily_q := 0

// Session Cycle Start (18:00, 00:00, 06:00, 12:00)
bool is_new_session = CycleEngine.f_is_new_session_cycle(daily_q)
if is_new_session and not is_in_pause
    // Log Session Q1 divider
    array.push(hist_session_q1, bar_index)
    if array.size(hist_session_q1) > 100
        array.shift(hist_session_q1)

    // Log Daily Q1/Q2/Q3/Q4 divider (based on which daily quarter we're in)
    if daily_cycle.current_quarter == 1
        array.push(hist_daily_q1, bar_index)
    else if daily_cycle.current_quarter == 2
        array.push(hist_daily_q2, bar_index)
    else if daily_cycle.current_quarter == 3
        array.push(hist_daily_q3, bar_index)
    else if daily_cycle.current_quarter == 4
        array.push(hist_daily_q4, bar_index)

    session_cycle.current_quarter := 1
    session_cycle.q1_start_bar := bar_index
    session_cycle.parent_start_bar := bar_index
    session_cycle.parent_timestamp := time
    prev_session_q := 0

// Micro Cycle Reset at Session Quarter Boundaries
bool is_new_session_quarter = (session_q != prev_session_q and session_q == 1) or is_new_session
if is_new_session_quarter and not is_in_pause
    // Log Micro Q1 divider
    array.push(hist_micro_q1, bar_index)
    if array.size(hist_micro_q1) > 100
        array.shift(hist_micro_q1)

    micro_cycle.current_quarter := 1
    micro_cycle.q1_start_bar := bar_index
    micro_cycle.parent_start_bar := bar_index
    micro_cycle.parent_timestamp := time
    prev_micro_q := 0

// Weekly Qx Detection (Thursday 18:00)
if current_dow == dayofweek.thursday and current_hour == 18 and not is_in_pause
    int prev_dow = dayofweek(time[1], CycleEngine.TIMEZONE_ET)
    int prev_hour = hour(time[1], CycleEngine.TIMEZONE_ET)
    if not (prev_dow == dayofweek.thursday and prev_hour == 18)
        bool already_tracked = false
        if array.size(hist_weekly_qx_bars) > 0
            int last_tracked = array.get(hist_weekly_qx_bars, array.size(hist_weekly_qx_bars) - 1)
            already_tracked := last_tracked == bar_index

        if not already_tracked
            array.push(hist_weekly_qx_bars, bar_index)
            if array.size(hist_weekly_qx_bars) > 100
                array.shift(hist_weekly_qx_bars)
```

**Stage 2: Keep Existing Quarter Transition Logic**

The existing `f_process_cycle_quarters()` function can remain largely unchanged - it will handle Q2/Q3/Q4 transitions correctly. However, update the cycle quarter assignments to only happen when NOT in a cycle start block:

```pinescript
// ────────────────────────────────────────────────────────────────────────────
// QUARTER CALCULATIONS (Every Bar)
// ────────────────────────────────────────────────────────────────────────────

// Calculate current quarters (every bar)
int monthly_q = CycleEngine.f_monthly_q_index_time(time, anchor_curr, anchor_next)
int weekly_q = CycleEngine.f_weekly_q_index(current_dow, current_hour)
int daily_q = CycleEngine.f_daily_q_index(current_hour)
int session_q = CycleEngine.f_session_q_index_within_daily_quarter(daily_q, current_hour, current_minute)
int micro_q = CycleEngine.f_micro_q_index(daily_q, current_hour, current_minute)

// ────────────────────────────────────────────────────────────────────────────
// UPDATE CURRENT QUARTERS (Only if not already set by cycle start logic)
// ────────────────────────────────────────────────────────────────────────────

if not is_in_pause
    // Only update if not in cycle start
    if not is_new_weekly
        weekly_cycle.current_quarter := weekly_q
    if not is_new_daily
        daily_cycle.current_quarter := daily_q
    if not is_new_session
        session_cycle.current_quarter := session_q
    if not is_new_session_quarter
        micro_cycle.current_quarter := micro_q

    // Monthly always updates (no dedicated cycle start detection)
    monthly_cycle.current_quarter := monthly_q

    // Process Q2/Q3/Q4 transitions
    prev_monthly_q := f_process_cycle_quarters(monthly_cycle, monthly_q, prev_monthly_q, ...)
    prev_weekly_q := f_process_cycle_quarters(weekly_cycle, weekly_q, prev_weekly_q, ...)
    prev_daily_q := f_process_cycle_quarters(daily_cycle, daily_q, prev_daily_q, ...)
    prev_session_q := f_process_cycle_quarters(session_cycle, session_q, prev_session_q, ...)
    prev_micro_q := f_process_cycle_quarters(micro_cycle, micro_q, prev_micro_q, ...)
```

---

### Data Structures to Add

**Monthly Weekly Tracking**:
```pinescript
var array<int> monthly_week_divider_bars = array.new<int>()
var array<string> monthly_week_divider_labels = array.new<string>()
var array<line> monthly_week_divider_lines = array.new<line>()
var array<label> monthly_week_divider_labels_vis = array.new<label>()
```

**Weekly Qx Tracking**:
```pinescript
var array<int> hist_weekly_qx_bars = array.new<int>()
var array<line> hist_weekly_qx_lines = array.new<line>()
var array<label> hist_weekly_qx_labels = array.new<label>()
```

**Helper Functions to Add**:
```pinescript
// Get first full week start of month (first Sunday 18:00 after first Monday)
f_get_first_full_week_start(int year_val, int month_val) =>
    // ... (copy from v6 lines 1333-1354)

// Check if in partial week
f_is_in_partial_week() =>
    // ... (copy from v6 lines 1356-1362)

// Get monthly quarter label (Q1/Q2/Q3/Q4/Qx)
f_get_monthly_quarter_label() =>
    // ... (copy from v6 lines 1364-1386)

// Get weekly quarter label (Q1/Q2/Q3/Q4/Qx)
f_get_weekly_quarter_label(int dow, int h) =>
    // ... (copy from v6 lines 1392-1408)
```

---

### Rendering Updates Required

**Monthly Qx Dividers** (new renderer):
Add a function to render monthly weekly dividers with Qx labels, similar to v6's approach. This will render ALL weekly dividers within the visible range and label them appropriately.

**Weekly Qx Dividers** (update existing renderer):
Add Qx rendering to the weekly divider function. Currently v7 only renders Q1-Q4. Add Qx array iteration.

**Historical Divider Renderer** (update):
The `f_render_historical_dividers()` function needs to handle Qx arrays for weekly cycle.

---

### Testing Checklist

After implementation, verify:

1. **Monthly Qx**: View M15/H1 chart, find month boundaries, confirm Qx dividers at partial weeks
2. **Weekly Qx**: View M5/M15 chart, find Thursday 18:00, confirm Qx divider renders
3. **Weekly Q1**: View M5/M15 chart, find Sunday 18:00, confirm Q1 divider renders
4. **Monthly Q-alignment**: Verify monthly Q2/Q3/Q4 dividers appear at Sunday 18:00 (not mid-week)
5. **Session Q1**: View M1/M5 chart, find daily quarter transitions (18:00/00:00/06:00/12:00), confirm Session Q1 dividers
6. **Daily Q1-Q4**: Verify Daily quarter dividers appear at session starts (18:00/00:00/06:00/12:00)
7. **Micro cycles**: View M1 chart, verify micro Q1-Q4 dividers reset at 90-minute intervals (session quarter boundaries)

---

### Critical File Locations

**V7 Implementation**:
- File: `D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\01_Core_Cycles\TTL_v7_01_Cycles.pine`
- Lines to modify:
  - 162-166: Previous quarter trackers (keep, but reset to 0 in cycle start blocks)
  - 196-254: `f_process_cycle_quarters()` (keep as-is for Q2/Q3/Q4)
  - 260-308: Main execution (ADD cycle start detection blocks before this)
  - 321-459: Rendering functions (UPDATE to handle Qx arrays)

**V6 Reference**:
- File: `C:\Users\garic\Downloads\TTL_Fractal_v6_Modular.pine`
- Key sections:
  - 1330-1408: Monthly/Weekly Qx detection functions
  - 1617-1718: Weekly cycle start handling
  - 1818-2010: Weekly quarter transitions and Qx tracking
  - 2018-2108: Daily cycle start handling
  - 2369-2468: Session cycle start handling
  - 2540-2619: Micro cycle reset handling

**CycleEngine Library**:
- File: `C:\Users\garic\Downloads\TTL_CycleEngine.pine`
- Relevant functions (already working):
  - 193-199: `f_is_new_weekly_cycle()`
  - 234-242: `f_is_new_daily_cycle()`
  - 302-314: `f_is_new_session_cycle()`
  - 392-393: `f_is_new_micro_cycle()`

---

### Summary

The core issue across all bugs is V7's architectural mistake: collapsing V6's two-stage cycle detection into a single universal function. V6 separates:
1. **Cycle start detection** (is_new_* functions) - handles Q1 explicitly
2. **Quarter transition detection** (prev_q comparison) - handles Q2/Q3/Q4

V7 tried to handle everything in one function (`f_process_cycle_quarters`), which breaks Q1 detection due to `prev_q` initialization.

The fix is to reintroduce explicit cycle start blocks before quarter transition processing, directly logging Q1 dividers when cycle start functions trigger. This mirrors V6's proven pattern and ensures Q1 dividers render correctly for all 5 fractal levels.

Additionally, V7 needs to implement Qx tracking for monthly (partial weeks) and weekly (Thursday-Sunday period) cycles, which V6 handles with dedicated detection logic separate from the Q1-Q4 cycle tracking.

## User Notes
<!-- Any specific notes or requirements from the developer -->

## Work Log
<!-- Updated as work progresses -->
- [YYYY-MM-DD] Started task, initial research
