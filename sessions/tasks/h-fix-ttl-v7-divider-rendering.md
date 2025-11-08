---
name: h-fix-ttl-v7-divider-rendering
branch: fix/h-fix-ttl-v7-divider-rendering
status: pending
created: 2025-01-07
---

# Fix TTL v7 Quarter Divider Rendering Bug

## Problem/Goal
The TTL v7 Phase 1 (Core Cycles) indicator has a critical bug with quarter divider rendering. Multiple dividers are stacking on wrong timeframes, and dividers are not being properly labeled compared to the working v6 implementation. The v6 modular version properly labels quarter dividers, but the v7 rebuild does not replicate this behavior correctly.

## Success Criteria
- [ ] Quarter dividers are properly labeled (Q1, Q2, Q3, Q4) matching v6 behavior
- [ ] No stacking/multiple dividers appearing on wrong timeframes
- [ ] Each timeframe shows only its appropriate cycle dividers (Monthly on H4, Weekly on H1, Daily on M15, Session on M5, Micro on M1)
- [ ] Visual validation passes on all 5 timeframes (M1, M5, M15, H1, H4)
- [ ] Divider rendering logic matches the working v6 modular implementation

## Context Manifest
<!-- Added by context-gathering agent -->

### How the TTL v7 Divider System Currently Works (and Where It's Broken)

The TTL indicator is a fractal cycle analysis tool that visualizes 5 nested time cycles: Monthly (viewed on H4), Weekly (H1), Daily (M15), Session (M5), and Micro (M1). Each cycle is divided into 4 quarters (Q1, Q2, Q3, Q4) that represent phases of market expansion. The system must render vertical divider lines at quarter boundaries, labeled appropriately, on the correct timeframe without visual duplication.

**The Two-Tier Divider Architecture (v6 Proven Pattern)**

The working v6 implementation uses a two-tier separation of concerns that prevents divider stacking and ensures clean rendering:

**TIER 1 - Current Cycle Dividers**: These track the actively-running cycle's quarter positions. In v6, these are stored as simple var line/label pairs (e.g., daily_q1_line, daily_q1_label, daily_q2_line, daily_q2_label, etc.). On every barstate.islast (last bar update), the old line/label objects are deleted via line.delete() and label.delete(), then recreated at the current quarter bar positions stored in the cycle object (e.g., daily_cycle.q1_start_bar, daily_cycle.q2_start_bar). This ensures the current cycle always shows exactly 1-4 dividers depending on how far into the cycle we are, with no duplication.

**TIER 2 - Historical Dividers**: These track completed cycles' quarter positions. As quarters transition (Q1 to Q2, Q2 to Q3, etc.), the bar_index position is pushed into historical bar position arrays (e.g., hist_daily_q1, hist_daily_q2). On barstate.islast, all previous historical divider visual objects (lines and labels) are batch-deleted by popping from their storage arrays, then the system iterates through the historical bar position arrays and redraws dividers for bars within the divider_lookback window (default 500 bars), skipping positions that match the current cycle's positions to avoid duplication.

**Why This Separation Matters**: The current cycle dividers update their positions as the cycle progresses (moving forward with the chart), while historical dividers are static markers of completed cycles. Without this separation, you get stacking bugs where the same position is drawn multiple times by different rendering passes.

**v7's Current Implementation Status (Buggy)**

The v7 rebuild (file: D:\Time Traders indicator\TTL IndiProject files\TTL_v7_Rebuild\01_Core_Cycles\TTL_v7_01_Cycles.pine) attempted to implement the two-tier pattern with DRY (Don't Repeat Yourself) principles using universal functions. It partially succeeded but has critical bugs:

**What v7 Got Right:**
- Universal cycle processor f_process_cycle_quarters() that handles quarter transitions for all 5 cycles (lines 174-232)
- Historical bar position tracking arrays (hist_monthly_q1 through hist_micro_q4) declared correctly (lines 68-91)
- Historical visual storage arrays (hist_*_q*_lines and hist_*_q*_labels) for preventing stacking (lines 94-137)
- UDT (User-Defined Type) pattern for current dividers called QuarterDividers (lines 147-155) that groups related line/label pairs semantically
- Universal current divider renderer f_render_current_dividers() (lines 367-424)
- Universal historical divider renderer f_render_historical_dividers() (lines 300-364)

**What v7 Got Wrong (The Bugs):**

1. **Labels Not Being Created**: In f_render_current_dividers() (lines 367-424), the labels are conditionally created based on show_quarter_labels setting. However, the pattern doesn't match v6's delete-then-recreate approach exactly. In v6 (TTL_Fractal_v6_Modular.pine lines 3510-3590), EVERY quarter's label gets checked for deletion first (if not na(daily_q1_label) then label.delete(daily_q1_label)) before creating the new one. The v7 version (lines 374-383 for Q1 example) follows this pattern, but inspection shows labels may not be getting the proper text content or styling parameters compared to v6.

2. **Stacking on Wrong Timeframes**: The rendering calls happen for ALL cycles on EVERY chart timeframe because barstate.islast is always true on the last bar regardless of what timeframe you're viewing. V6 includes a filtering system (lines 3413: "f_should_show_dividers(cycle_name, active_cycle, divider_fractal_filter)") that determines if a specific cycle's dividers should render based on the chart timeframe and user settings. V7 has this check in the universal functions (line 301: f_should_show_cycle(cycle_name)) but it's only checking if the cycle toggle is enabled, NOT if it's appropriate for the current chart timeframe. This means if you're on M5 timeframe viewing Session cycles, you're also rendering Daily, Weekly, and Monthly dividers which creates visual noise and stacking.

3. **Historical Dividers Filtering Issue**: In f_render_historical_dividers() (lines 300-364), the function iterates through historical bar positions and checks "(bar_index - bar_pos) <= divider_lookback" to limit how far back to draw (line 326). However, there's a critical second check missing from v6: "if bar_pos != cycle.q1_start_bar" (v6 line 3796). This check prevents drawing a historical divider at the SAME bar position as the current cycle divider, which would cause duplication. V7 line 326 has this check for Q1 ("if bar_pos != cycle.q1_start_bar") but the logic needs verification for Q2/Q3/Q4 as well (lines 337, 348, 359 - these checks use cycle.q2_start_bar, cycle.q3_start_bar, cycle.q4_start_bar respectively, which is correct).

4. **Label Text Content**: Looking at v6 line 3466, the label text is simply "Q1" as a string literal. V7 line 330 also uses "Q1" as string literal. This should be working. The bug report says labels aren't appearing, which suggests either: (a) the label.new() call is failing silently due to parameter issues, (b) the labels are being created but immediately deleted by the next rendering pass, or (c) the label positioning is placing them off-screen.

5. **Label Positioning**: V6 uses "bar_pos + 1" for the x-coordinate (line 3466: "label.new(weekly_cycle.q1_start_bar + 1, line_bottom, ...)"). V7 also uses this pattern (line 330: "label.new(bar_pos + 1, line_bottom, ...)"). The y-coordinate uses line_bottom which is calculated as "lowest_visible - (price_range * 0.3)" (v7 line 296, v6 line 3409). If the visible price range calculation is off, labels could be rendering far below the visible chart area.

**The Cycle Detection Flow (Working Correctly in v7)**

Every bar, the system calculates which quarter each cycle is currently in:
- Monthly quarter: Uses time-based calculation via CycleEngine.f_monthly_q_index_time() with anchor timestamps (v7 lines 249-255)
- Weekly quarter: Based on day-of-week and hour via CycleEngine.f_weekly_q_index() (line 258)
- Daily quarter: Based on hour via CycleEngine.f_daily_q_index() (line 261)
- Session quarter: Based on daily quarter, hour, and minute via CycleEngine.f_session_q_index_within_daily_quarter() (line 264)
- Micro quarter: Based on daily quarter, hour, and minute via CycleEngine.f_micro_q_index() (line 267)

These calculations happen BEFORE the transitional pause check (lines 273-286), which is correct per the 2-phase pattern documented in coding_session_2025-01-07.md. The pause period (hour 17 ET, 5pm-5:59pm) is when the daily cycle transitions between Q4 and Q1, and no calculations should update cycle state during this window.

**Quarter Transition Detection**

The universal processor f_process_cycle_quarters() detects transitions by comparing current_q to prev_q (lines 177-180). When a transition occurs:
- Q1 Start (new cycle): Sets q1_start_bar, parent_start_bar, parent_timestamp; archives previous Q1 position if it exists (lines 183-192)
- Q2 Start: Sets q2_start_bar, marks Q1 as complete, archives Q2 position (lines 195-205)
- Q3 Start: Sets q3_start_bar, marks Q2 as complete, archives Q3 position (lines 208-217)
- Q4 Start: Sets q4_start_bar, marks Q3 as complete, archives Q4 position (lines 220-229)

The archiving pushes bar_index into the appropriate hist_q* array and maintains a max size of 100 historical positions per quarter (lines 191-192: "if array.size(hist_q1) > 100 then array.shift(hist_q1)").

**Visual Rendering Flow (Where Bugs Occur)**

On barstate.islast (last bar of chart), two rendering phases execute:

**Phase 1 - Current Dividers** (lines 427-431):
Calls f_render_current_dividers() for each of the 5 cycles. This function:
1. Checks if cycle should render via f_should_show_cycle() and barstate.islast and not na(cycle.q1_start_bar) (line 370)
2. Deletes old Q1 line/label if they exist (lines 374-377)
3. Creates new Q1 line at cycle.q1_start_bar position (line 380)
4. Creates new Q1 label if show_quarter_labels is true (lines 382-383)
5. Repeats for Q2/Q3/Q4 if their start bars are not na (lines 386-419)
6. Returns a new QuarterDividers UDT instance with all the line/label references (line 422)

**Phase 2 - Historical Dividers** (lines 433-437):
Calls f_render_historical_dividers() for each of the 5 cycles. This function:
1. Checks if cycle should render via f_should_show_cycle() and barstate.islast (line 301)
2. Batch-deletes all old historical divider visual objects by popping from storage arrays (lines 305-320)
3. For each quarter (Q1, Q2, Q3, Q4):
   - Iterates through the hist_q*_bars array (e.g., line 324: "for i = 0 to array.size(hist_q1_bars) - 1")
   - Gets the bar position (line 325: "int bar_pos = array.get(hist_q1_bars, i)")
   - Checks if it's not the current cycle's position AND within lookback window (line 326)
   - Creates line with extend.both and dotted style (line 327)
   - Creates label with "Q1"/"Q2"/"Q3"/"Q4" text if show_quarter_labels is true (lines 329-331)
   - Pushes the new line/label into the storage arrays (lines 328, 331)

**The Critical Difference: v6 Uses Separate Variables, v7 Uses Universal Function**

V6 has 5 separate rendering blocks (one for each cycle), each with direct variable names:
```pinescript
// V6 Daily rendering (lines 3505-3590)
if show_quarter_dividers and f_should_show_dividers("Daily", active_cycle, divider_fractal_filter) and barstate.islast
    if not na(daily_cycle.q1_start_bar)
        if not na(daily_q1_line)
            line.delete(daily_q1_line)
        daily_q1_line := line.new(daily_cycle.q1_start_bar, line_bottom, ...)
        if show_labels
            if not na(daily_q1_label)
                label.delete(daily_q1_label)
            daily_q1_label := label.new(daily_cycle.q1_start_bar + 1, line_bottom, "Q1", ...)
```

V7 abstracts this into a universal function that returns a UDT:
```pinescript
// V7 universal rendering (lines 367-424, called at line 429)
daily_dividers := f_render_current_dividers(daily_cycle, "Daily", daily_dividers)
```

The UDT pattern is more DRY but introduces a layer of indirection. If the UDT isn't being properly updated or if the old_dividers parameter isn't passing the correct state, dividers could fail to render.

### Technical Reference: Key Data Structures

**FractalCycle UDT (from TTL_Types/2 library)**
```pinescript
type FractalCycle
    int current_quarter        // 1-4
    int q1_start_bar          // Bar index where Q1 started
    int q2_start_bar          // Bar index where Q2 started
    int q3_start_bar          // Bar index where Q3 started
    int q4_start_bar          // Bar index where Q4 started
    int q1_end_bar            // Bar index where Q1 ended
    int q2_end_bar            // Bar index where Q2 ended
    int q3_end_bar            // Bar index where Q3 ended
    bool q1_complete          // True after Q2 starts
    int parent_start_bar      // Same as q1_start_bar (cycle start)
    int parent_timestamp      // Timestamp of cycle start
```

**QuarterDividers UDT (v7 implementation, lines 147-155)**
```pinescript
type QuarterDividers
    line q1_line
    label q1_label
    line q2_line
    label q2_label
    line q3_line
    label q3_label
    line q4_line
    label q4_label
```

**Historical Storage Arrays (v7 lines 68-137)**
- Bar position arrays: hist_monthly_q1 through hist_micro_q4 (20 arrays total: 5 cycles × 4 quarters)
- Visual object arrays: hist_monthly_q1_lines/labels through hist_micro_q4_lines/labels (40 arrays total: 5 cycles × 4 quarters × 2 types)

**Current Divider Storage (v7 lines 158-162)**
- 5 QuarterDividers instances: monthly_dividers, weekly_dividers, daily_dividers, session_dividers, micro_dividers

### PineScript v6 Constraints That Affect This Bug

1. **No Multi-Line Syntax**: All ternary operators, function signatures, and function calls must be single-line
2. **Immutable Function Parameters**: Cannot reassign function parameters with := operator; must return new values
3. **No Early Returns**: Functions must have a single return expression at the end
4. **barstate.islast Timing**: Rendering code only executes on the last bar of the chart; all drawing objects from previous bars persist unless explicitly deleted
5. **Line/Label Limits**: PineScript has max_lines_count=500 and max_labels_count=500 (set in indicator() call line 2); exceeding these limits causes silent failures

### The 5 Cycles and Their Timeframes

| Cycle | Best Timeframe | Quarter Duration | Quarter Start Times (ET) |
|-------|----------------|------------------|--------------------------|
| Monthly | H4 (4-hour) | ~1 week | Sunday 18:00 (weekly boundaries within month) |
| Weekly | H1 (1-hour) | ~1 day | Sun 18:00 (Q1), Mon 18:00 (Q2), Tue 18:00 (Q3), Wed 18:00 (Q4) |
| Daily | M15 (15-min) | 6 hours | 18:00 (Q1), 00:00 (Q2), 06:00 (Q3), 12:00 (Q4) |
| Session | M5 (5-min) | 90 minutes | Within daily quarters, 4 sessions per day |
| Micro | M1 (1-min) | 22.5 minutes | Within session quarters, 4 micros per session |

All cycles respect the Transitional Law pause period (17:00-17:59:59 ET / 5pm-5:59pm) where no state updates occur.

### File Locations

**Buggy v7 Implementation:**
- Main indicator: D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\01_Core_Cycles\TTL_v7_01_Cycles.pine
- Test script: D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\01_Core_Cycles\TTL_v7_01_Test.pine

**Working v6 Reference:**
- Modular implementation: C:\Users\garic\Downloads\TTL_Fractal_v6_Modular.pine
  - Current dividers: Lines 3456-3752 (separate blocks per cycle)
  - Historical dividers: Lines 3756-4100 (batch delete/redraw pattern)
  - Variable declarations: Lines 947-974 (current line vars), 982-1007 (current label vars), 811-899 (historical arrays)

**Library Dependencies:**
- CycleEngine: C:\Users\garic\Downloads\TTL_CycleEngine.pine (imported as GmoneyT/TTL_CycleEngine/3)
- Types: Published library GmoneyT/TTL_Types/2

**Project Documentation:**
- Build log: D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\docs\coding_session_2025-01-07.md

### Configuration Settings Affecting Rendering

**Input Controls (v7 lines 20-38):**
- show_monthly, show_weekly, show_daily, show_session, show_micro: Boolean toggles for each cycle
- divider_lookback: How many bars back to render historical dividers (default 500, line 28)
- show_quarter_labels: Whether to render Q1/Q2/Q3/Q4 text labels (default true, line 29)
- label_size: Text size for labels ("tiny"/"small"/"normal"/"large", default "small", line 30)
- divider_color_universal: Color for all dividers (default #4d4d4d grey, line 33)

**Helper Functions (v7 lines 44-54):**
- f_get_label_size(): Converts string to size constant
- f_get_cycle_color(): Returns cycle-specific color (though v7 uses universal grey for dividers)
- f_should_show_cycle(): Checks if cycle toggle is enabled

### What Needs to Be Fixed

Based on the bug report and v6 reference implementation, the fixes needed are:

1. **Add Timeframe Filtering Logic**: Implement v6's f_should_show_dividers() pattern that checks not just if the cycle is enabled, but if it's appropriate for the current chart timeframe. This prevents stacking by ensuring only the correct cycle renders on each timeframe.

2. **Verify Label Creation Logic**: Ensure labels are being created with correct parameters. Compare v7 lines 382-383 (Q1 label creation) against v6 lines 3466-3469 to verify all parameters match (style, color, textcolor, size, yloc, textalign).

3. **Debug UDT State Passing**: Verify that the QuarterDividers UDT instances are correctly passing state between render calls. The pattern "daily_dividers := f_render_current_dividers(daily_cycle, "Daily", daily_dividers)" (line 429) should delete old dividers from the incoming old_dividers parameter and return new ones, but if the state isn't persisting correctly, dividers could disappear.

4. **Verify Historical Deduplication**: Confirm that historical divider rendering is correctly skipping positions that match current cycle positions for ALL quarters (Q1/Q2/Q3/Q4), not just Q1.

5. **Check Visual Object Limits**: With 5 cycles × 4 quarters × 2 tiers × 500 lookback bars potentially drawing lines/labels, verify we're not hitting PineScript's 500-object limits. The max_lines_count=500 and max_labels_count=500 in the indicator() declaration (line 2) may need to be increased or the lookback reduced.

### Expected Correct Behavior (from v6)

When viewing the Daily cycle on M15 timeframe:
- Should see exactly 4 current dividers (Q1, Q2, Q3, Q4) if all quarters have started
- Should see historical dividers for previous daily cycles within the lookback window
- Each divider is a thin dotted vertical line extending through the full chart (extend.both)
- Each divider has a label positioned at bar_pos + 1 (one bar to the right), at line_bottom (30% below visible range), displaying "Q1"/"Q2"/"Q3"/"Q4" in grey text
- NO dividers from other cycles (Monthly, Weekly, Session, Micro) should appear on this timeframe
- On barstate.islast, all old dividers are deleted and redrawn to reflect current positions

When switching to H1 timeframe:
- Daily dividers should disappear
- Weekly dividers should appear following the same pattern
- This is the timeframe filtering that v7 is missing

## User Notes
<!-- Any specific notes or requirements from the developer -->

## Work Log
<!-- Updated as work progresses -->
- [YYYY-MM-DD] Started task, initial research
