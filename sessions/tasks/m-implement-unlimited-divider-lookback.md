---
name: m-implement-unlimited-divider-lookback
branch: feature/m-implement-unlimited-divider-lookback
status: pending
created: 2025-01-08
submodules: [TTL_v7_Rebuild]
---

# Implement Unlimited Historical Divider Lookback and Fixed Label Positioning

## Problem/Goal
The TTL v7 indicator currently has two limitations with quarter divider rendering:

1. **Limited Historical Lookback**: The `divider_lookback` setting (default 500 bars) restricts how far back historical cycle dividers are rendered. This prevents users from seeing the full cycle history across all bars when they want to analyze longer timeframes or need complete historical context.

2. **Drifting Label Positions**: Quarter labels (Q1, Q2, Q3, Q4) are currently positioned using `yloc.price` with a calculated `line_bottom` price level. This causes labels to drift out of view when users scroll/zoom the chart vertically, and they may accidentally cover price bars. Labels need to remain fixed at the bottom of divider lines, visible and properly positioned relative to the user's viewport regardless of scale or scroll position.

## Success Criteria

**Historical Lookback:**
- [ ] Users can set `divider_lookback` to 0 (unlimited) to render all historical dividers across entire chart history
- [ ] Historical divider calculation runs from bar 0 onwards, capturing all quarter transitions
- [ ] Performance remains acceptable with unlimited rendering on charts with extensive history (tested on multi-month/year datasets)
- [ ] Option to configure lookback limit remains available for users who prefer limited rendering

**Fixed Label Positioning:**
- [ ] Quarter labels (Q1, Q2, Q3, Q4) remain fixed at the bottom of divider lines regardless of Y-axis scroll/zoom
- [ ] Labels remain visible in viewport when divider is visible
- [ ] Labels do not accidentally cover price bars during chart scale changes
- [ ] Label positioning adapts automatically to chart viewport changes (zoom in/out, pan up/down)
- [ ] Visual validation passes on all 5 timeframes (M1, M5, M15, H1, H4) with various zoom/scroll scenarios

## Context Manifest

### How the TTL v7 Divider System Currently Works

The TTL v7 indicator implements a sophisticated two-tier divider rendering system for visualizing fractal cycle boundaries across 5 nested time cycles (Monthly, Weekly, Daily, Session, Micro). Understanding this architecture is critical to implementing both unlimited historical lookback and fixed label positioning.

#### The Two-Tier Divider Architecture (Proven v6 Pattern)

The system separates divider rendering into two distinct tiers to prevent visual duplication and maintain clean rendering:

**TIER 1 - Current Cycle Dividers:** These track the actively-running cycle's quarter positions. When a cycle starts (Q1), it progresses through quarters Q1 → Q2 → Q3 → Q4 before starting a new cycle. The current dividers show where each quarter of the CURRENT cycle began. For example, if you're currently in Q3 of a Daily cycle, you'll see exactly 3 dividers: one at the bar where Q1 started (cycle start), one at Q2 start, and one at Q3 start. These are stored in the QuarterDividers UDT (User-Defined Type) which holds line and label object references for all 4 possible quarters.

On every `barstate.islast` (the last bar update), the `f_render_current_dividers()` function (lines 389-446) executes a delete-then-recreate pattern: it deletes the old line/label objects from the previous rendering pass, then creates new ones at the current quarter bar positions stored in the FractalCycle object (e.g., `daily_cycle.q1_start_bar`, `daily_cycle.q2_start_bar`). This ensures the current cycle always shows exactly 1-4 dividers depending on cycle progress, with no duplication.

**TIER 2 - Historical Dividers:** These track completed cycles' quarter positions. As quarters transition (Q1→Q2, Q2→Q3, Q3→Q4, Q4→Q1), the `bar_index` position is pushed into historical bar position arrays (e.g., `hist_daily_q1`, `hist_daily_q2`, `hist_daily_q3`, `hist_daily_q4`) by the `f_process_cycle_quarters()` function (lines 196-254). These arrays are capped at 100 positions per quarter per cycle using `array.shift()` when the size exceeds 100 (lines 213-214, 226-227, 238-239, 250-251).

On `barstate.islast`, the `f_render_historical_dividers()` function (lines 322-386) performs a batch delete-then-redraw operation:

1. **Batch Delete Phase** (lines 327-342): Pops and deletes ALL old historical divider visual objects (lines and labels) from their storage arrays. This prevents stacking from previous render passes.

2. **Selective Redraw Phase** (lines 345-386): Iterates through the historical bar position arrays for each quarter (Q1, Q2, Q3, Q4). For each stored bar position, it checks two critical conditions:
   - `bar_pos != cycle.q*_start_bar`: Ensures we don't draw a historical divider at the same position as a current cycle divider (prevents duplication between tiers)
   - `(bar_index - bar_pos) <= divider_lookback`: Only renders dividers within the lookback window (default 500 bars)

If both conditions pass, it creates a new line and label, then pushes them into the visual storage arrays (`hist_*_q*_lines` and `hist_*_q*_labels`).

**Why This Separation Matters:** Without tier separation, the same bar position could be rendered multiple times (once as "current" and again as "historical"), creating visual stacking bugs. The tier system ensures each position is rendered exactly once with correct styling.

#### Current Lookback Limitation: The 500-Bar Restriction

The `divider_lookback` input setting (line 29) defaults to 500 bars with a hardcoded maximum of 2000 bars (`input.int(500, "Divider Lookback", minval=50, maxval=2000)`). This restriction exists for two reasons:

1. **Historical Purpose**: The original design assumed users would primarily focus on recent cycle history, not entire chart history. Rendering thousands of dividers was considered unnecessary.

2. **Performance Caution**: PineScript has object count limits (`max_lines_count=500`, `max_labels_count=500` declared in line 2). With 5 cycles × 4 quarters × 2 tiers potentially creating hundreds of objects, the lookback limit acts as a safety valve to prevent hitting these limits.

However, this restriction prevents users from analyzing long-term cycle patterns across months or years of data. The historical divider rendering loop (lines 346-386) skips any bar positions where `(bar_index - bar_pos) > divider_lookback`, which means older cycles simply disappear from view even though their positions are still stored in the arrays (remember: arrays are capped at 100 positions each, not by lookback).

**The Mathematical Reality**: The worst-case scenario for object counts occurs on very fast-cycling timeframes with unlimited lookback:
- Micro cycles on M1 charts: ~23 minutes per cycle = ~2.6 cycles per hour = ~62 cycles per day
- 4 quarters per cycle × 62 cycles × 2 objects (line + label) = 496 objects per day for one tier
- With both tiers and 5 cycles active, you could theoretically exceed limits very quickly

However, in practice:
- Only ONE cycle is active per timeframe (the `active_cycle` filtering system ensures this, per the previous bug fix in task h-fix-ttl-v7-divider-rendering.md)
- Most timeframes have slower cycles (Daily on M15 = 1 cycle per day, Weekly on H1 = 1 cycle per week)
- The 100-position array cap per quarter (lines 213-251) naturally limits historical storage

#### Current Label Positioning: The yloc.price Problem

Labels are currently positioned using the `yloc.price` coordinate system (lines 352, 363, 374, 385, 405, 417, 429, 441). This means the Y-coordinate value passed to `label.new()` is interpreted as a price level, not a pixel position or percentage.

**How It Works Now (Lines 315-319):**
```pinescript
float highest_visible = ta.highest(high, 100)
float lowest_visible = ta.lowest(low, 100)
float price_range = highest_visible - lowest_visible
float line_bottom = lowest_visible - (price_range * 0.3)
float line_top = highest_visible + (price_range * 0.3)
```

The `line_bottom` calculation finds the lowest price in the last 100 bars, calculates the visible price range, then subtracts 30% of that range to position divider lines and labels "below the chart." Labels are created at `line_bottom` with `yloc=yloc.price` (e.g., line 352: `label.new(bar_pos + 1, line_bottom, "Q1", ..., yloc=yloc.price, ...)`).

**The Drifting Problem:** When a user scrolls the chart vertically (zooms in/out on the Y-axis) or when price moves significantly, the `line_bottom` price level remains fixed at whatever price was calculated during the last `barstate.islast` rendering pass. This causes several issues:

1. **Labels Drift Out of View**: If price moves up significantly after rendering, the labels (anchored to the old `line_bottom` price) are now far below the visible viewport. Users must scroll down to see them, which is frustrating and breaks the visual connection to the divider lines.

2. **Labels Cover Price Bars**: Conversely, if price drops below the `line_bottom` level, labels can overlap with actual price bars, obscuring trading data.

3. **Scale Changes Break Positioning**: When users change the Y-axis scale (logarithmic vs linear, auto-scaling, etc.), the relationship between price levels and viewport positions changes, causing labels to appear in unexpected locations.

**Why yloc.price Was Used:** This was likely chosen because it's the simplest coordinate system in PineScript v6. The lines themselves use `extend=extend.both` which makes them span the entire chart vertically regardless of viewport, but labels don't have an "extend" option—they need a specific Y-coordinate.

**The Alternative: yloc.abovebar and yloc.belowbar:** PineScript v6 provides viewport-relative positioning modes for labels:
- `yloc=yloc.abovebar`: Positions label relative to the high of the bar, staying above the candlestick
- `yloc=yloc.belowbar`: Positions label relative to the low of the bar, staying below the candlestick

These modes make labels "stick" to the bar regardless of Y-axis scroll/zoom. However, the current design positions labels at divider bottoms (using `line_bottom` as the Y-coordinate), not at specific bar high/low values. The divider lines use `extend=extend.both` which means they extend infinitely in both directions—there is no "bottom" position in the visual sense, only in the price coordinate sense.

**The Core Challenge:** We need labels to remain at a consistent visual position relative to the divider lines (e.g., "at the bottom of the visible divider"), but divider lines extend infinitely using `extend=extend.both`. The solution requires either:
1. Anchoring labels to actual bar high/low values using `yloc.abovebar`/`yloc.belowbar` (but this means they won't be at divider "bottoms", they'll be at bar edges)
2. Removing `extend=extend.both` from lines and drawing them from `line_bottom` to `line_top` as fixed-length lines, then positioning labels at the `line_bottom` coordinate (but this loses the infinite extension visual)
3. Using a different label positioning strategy entirely (e.g., always at the low of the bar at the divider position using `yloc.belowbar`)

The trade-off is between "labels at consistent viewport positions" vs "labels at visually intuitive positions relative to dividers."

#### Quarter Detection and Transition Flow (Working Correctly)

Before we can render dividers, the system must detect when quarters transition. This happens in two phases on every bar:

**Phase 1: Calculate Current Quarter (Lines 261-290)** - This runs BEFORE the transitional pause check, ensuring calculations always reflect current time:
- Monthly quarter: Uses `CycleEngine.f_monthly_q_index_time()` with anchor timestamps (lines 271-277)
- Weekly quarter: Uses `CycleEngine.f_weekly_q_index()` based on day-of-week and hour (line 280)
- Daily quarter: Uses `CycleEngine.f_daily_q_index()` based on hour (line 283)
- Session quarter: Uses `CycleEngine.f_session_q_index_within_daily_quarter()` based on daily quarter, hour, minute (line 286)
- Micro quarter: Uses `CycleEngine.f_micro_q_index()` based on daily quarter, hour, minute (line 289)

**Phase 2: Process Quarter Transitions (Lines 295-308)** - This runs ONLY if NOT in transitional pause (hour 17 ET / 5pm-5:59pm):
- Updates `cycle.current_quarter` for all 5 cycles (lines 297-301)
- Calls `f_process_cycle_quarters()` for each cycle (lines 304-308)

The universal processor `f_process_cycle_quarters()` (lines 196-254) detects transitions by comparing `current_q` to `prev_q`:

**Q1 Start (New Cycle, lines 199-214):**
- Triggered when `current_q == 1` and `prev_q != 1`
- Sets `cycle.q1_start_bar := bar_index`
- Sets `cycle.parent_start_bar := bar_index` (cycle start)
- Sets `cycle.parent_timestamp := time`
- Archives previous Q1 position: If `cycle.q1_start_bar` already existed and is different from current `bar_index`, it pushes the old position into `hist_q1` array
- Maintains 100-position cap: `if array.size(hist_q1) > 100 then array.shift(hist_q1)`

**Q2 Start (lines 217-227):**
- Triggered when `current_q == 2` and `prev_q == 1`
- Sets `cycle.q2_start_bar := bar_index`
- Marks Q1 as complete: `cycle.q1_end_bar := bar_index - 1` and `cycle.q1_complete := true`
- Archives Q2 position into `hist_q2` array (no previous Q2 check needed—Q2 always starts fresh each cycle)

**Q3 Start (lines 230-239):**
- Triggered when `current_q == 3` and `prev_q == 2`
- Sets `cycle.q3_start_bar := bar_index`
- Marks Q2 as complete: `cycle.q2_end_bar := bar_index - 1`
- Archives Q3 position into `hist_q3`

**Q4 Start (lines 242-251):**
- Triggered when `current_q == 4` and `prev_q == 3`
- Sets `cycle.q4_start_bar := bar_index`
- Marks Q3 as complete: `cycle.q3_end_bar := bar_index - 1`
- Archives Q4 position into `hist_q4`

**Returns**: The function returns `current_q` which becomes the new `prev_q` value for the next bar (lines 304-308 assign this with `:=` operator).

**Critical Note**: When Q1 starts again (new cycle), the OLD Q1 position is archived (line 212: `array.push(hist_q1, cycle.q1_start_bar)`), then the cycle object is updated with the NEW Q1 position (line 206: `cycle.q1_start_bar := bar_index`). This means Q2/Q3/Q4 positions from the previous cycle remain in the cycle object until they're overwritten by the new cycle's quarters. Historical rendering (TIER 2) skips these positions using the deduplication checks (e.g., line 348: `if bar_pos != cycle.q1_start_bar`).

#### Visual Rendering Flow (Where Implementation Changes Will Happen)

On `barstate.islast` (line 323 for historical, line 392 for current), rendering executes:

**Phase 1 - Current Dividers (lines 449-453):**
Calls `f_render_current_dividers(cycle, cycle_name, old_dividers)` for each of the 5 cycles. This function:

1. Checks rendering conditions (line 392): `show_quarter_dividers` is true, `f_should_show_cycle(cycle_name)` returns true (cycle is active for this timeframe), `barstate.islast` is true, and `cycle.q1_start_bar` is not `na` (cycle has started)

2. Deletes old Q1 dividers (lines 396-399): If `old_dividers.q1_line` is not `na`, calls `line.delete()`. Same for `old_dividers.q1_label`.

3. Creates new Q1 dividers (lines 402-405): Creates line at `cycle.q1_start_bar` position from `line_bottom` to `line_top` with `extend=extend.both`. If `show_quarter_labels` is true, creates label at `cycle.q1_start_bar + 1` (one bar to the right), at `line_bottom` Y-coordinate, with `yloc=yloc.price`.

4. Repeats for Q2/Q3/Q4 (lines 408-441): But only if `cycle.q*_start_bar` is not `na` (quarter has started in current cycle).

5. Returns new UDT instance (line 444): `QuarterDividers.new(new_q1_line, new_q1_label, ...)` containing all the line/label references.

The return value is assigned back to the cycle's dividers variable with `:=` (e.g., line 449: `monthly_dividers := f_render_current_dividers(...)`), ensuring the references persist for the next rendering pass.

**Phase 2 - Historical Dividers (lines 455-459):**
Calls `f_render_historical_dividers(cycle_name, cycle, hist_q*_bars arrays, hist_q*_lines arrays, hist_q*_labels arrays)` for each of the 5 cycles. This function:

1. Checks rendering conditions (line 323): Same as current dividers.

2. Batch deletes old historical visuals (lines 327-342): Uses `while` loops to pop and delete all line/label objects from all 4 quarters' storage arrays. This clears the slate for fresh rendering.

3. Redraws Q1 historical dividers (lines 345-353):
   - If `hist_q1_bars` array has positions: Loops through each position
   - Gets bar position: `int bar_pos = array.get(hist_q1_bars, i)`
   - Checks deduplication: `bar_pos != cycle.q1_start_bar` (not current Q1)
   - Checks lookback: `(bar_index - bar_pos) <= divider_lookback` (within window)
   - If both true: Creates line and pushes to `hist_q1_lines`, creates label (if enabled) and pushes to `hist_q1_labels`

4. Repeats for Q2/Q3/Q4 (lines 356-386): Same pattern with respective arrays and cycle position checks.

**Critical Performance Note**: The historical rendering loop iterates through ALL positions in the arrays (up to 100 per quarter), but only renders those within the lookback window. With unlimited lookback, this changes: the loop will still iterate through 100 positions (array cap), but the lookback check `(bar_index - bar_pos) <= divider_lookback` will ALWAYS pass if lookback is unlimited (or set to 0 meaning "no limit"). This means rendering will max out at 100 dividers per quarter per cycle, not thousands, because the array cap limits storage.

### What Needs to Change for Unlimited Lookback

The task requires making `divider_lookback` configurable to allow unlimited rendering. Here's what needs modification:

**1. Input Setting Change (Line 29):**
Current: `divider_lookback = input.int(500, "Divider Lookback", minval=50, maxval=2000, group="Visual Settings")`

Needs to become: Allow `0` to mean "unlimited," while keeping the ability to set limits. Options:
- Change `minval` to `0` and remove `maxval` constraint entirely
- Document that `0` means unlimited in the tooltip
- Consider adding a separate boolean input like `unlimited_lookback = input.bool(false, "Unlimited Lookback")` for clarity, but this adds UI clutter

**2. Lookback Check Logic (Lines 348, 359, 370, 381):**
Current: `if bar_pos != cycle.q1_start_bar and (bar_index - bar_pos) <= divider_lookback`

Needs to become: Skip the lookback check if unlimited is enabled. Two approaches:

**Approach A - Conditional Check:**
```pinescript
bool is_within_lookback = divider_lookback == 0 or (bar_index - bar_pos) <= divider_lookback
if bar_pos != cycle.q1_start_bar and is_within_lookback
```

**Approach B - Separate Code Paths:**
Keep the existing check but add a separate rendering branch for unlimited mode. This is cleaner but requires duplication.

**3. Performance Considerations:**
With unlimited lookback enabled:
- Maximum rendered dividers per quarter = 100 (array cap)
- Maximum rendered dividers per cycle = 400 (4 quarters × 100)
- Maximum rendered dividers for one active cycle = 400 (only one cycle displays per timeframe)
- Maximum visual objects = 800 (400 lines + 400 labels)

This EXCEEDS the `max_lines_count=500` and `max_labels_count=500` limits set in line 2!

**Solutions:**
- **Option 1**: Increase limits in indicator declaration (line 2) to `max_lines_count=1000, max_labels_count=1000`. PineScript v6 supports up to 500 per type, but extending to 1000 might not be allowed. Need to verify PineScript limits.
- **Option 2**: Reduce array cap from 100 to 50 per quarter, which would max out at 400 total objects (200 lines + 200 labels), safely under 500 per type.
- **Option 3**: Make array cap configurable via input setting, allowing users to balance history vs performance.
- **Option 4**: Implement intelligent culling—only render dividers that are actually visible in the current viewport (requires detecting visible bar range, which PineScript v6 doesn't directly expose).

**Recommended Approach**: Increase limits to 1000 per type (if allowed) or reduce array cap to 62 per quarter (62 × 4 × 2 = 496 objects per type, just under 500).

**4. Documentation Requirements:**
- Add tooltip to `divider_lookback` input explaining that `0` means unlimited
- Add warning about performance impact with unlimited lookback on fast-cycling timeframes
- Update build log and documentation to reflect new capability

### What Needs to Change for Fixed Label Positioning

The task requires labels to remain fixed at divider bottoms regardless of Y-axis scroll/zoom. Here are the viable approaches:

**Approach 1: Switch to yloc.belowbar (Recommended)**

Change all label creations from:
```pinescript
label.new(bar_pos + 1, line_bottom, "Q1", ..., yloc=yloc.price, ...)
```

To:
```pinescript
label.new(bar_pos + 1, low[bar_index - (bar_pos + 1)], "Q1", ..., yloc=yloc.belowbar, ...)
```

**How It Works:**
- `yloc.belowbar` positions the label below the candlestick at the specified bar
- The Y-coordinate becomes relative to the low of that bar, not an absolute price
- `low[bar_index - (bar_pos + 1)]` gets the low price at the bar where the label is positioned
- Labels will always appear just below the bottom wick, regardless of viewport scaling

**Benefits:**
- Labels remain viewport-relative, never drift
- Simple implementation (just change yloc mode and Y-coordinate calculation)
- No need to track or recalculate `line_bottom` dynamically

**Drawbacks:**
- Labels appear at bar lows, not at a consistent "bottom of chart" position
- On volatile bars with long wicks, labels might be far from where users expect
- Doesn't maintain the "30% below visible range" aesthetic from v6

**Approach 2: Remove extend=extend.both and Draw Fixed-Length Lines**

Change line creation from:
```pinescript
line.new(bar_pos, line_bottom, bar_pos, line_top, ..., extend=extend.both)
```

To:
```pinescript
line.new(bar_pos, line_bottom, bar_pos, line_top, ..., extend=extend.none)
```

Then keep labels at `line_bottom` with `yloc=yloc.price`. Recalculate `line_bottom` and `line_top` on EVERY `barstate.islast` pass (lines 315-319 already do this).

**How It Works:**
- Lines are drawn from `line_bottom` to `line_top` without infinite extension
- `line_bottom` is recalculated based on current visible range (lines 315-319)
- Labels at `line_bottom` will track with the recalculated position
- On each chart scale/scroll, PineScript re-renders on `barstate.islast`, recalculating boundaries

**Benefits:**
- Labels remain at consistent "bottom of divider" position visually
- Maintains the v6 aesthetic of labels below chart area
- Line boundaries adapt to viewport changes

**Drawbacks:**
- Loses the infinite extension visual (lines don't extend through entire chart vertically)
- Requires recalculating `line_bottom`/`line_top` on every render (already happening, so no performance cost)
- Lines might not reach the top/bottom of viewport if calculation is off

**Approach 3: Dynamic Label Repositioning (Complex)**

Keep `extend=extend.both` for lines, but recalculate label Y-coordinates on every render based on current viewport.

**How It Works:**
- Lines remain infinitely extending
- On each `barstate.islast`, delete and recreate labels with updated `line_bottom` calculation
- Labels track the "bottom of visible range" as viewport changes

**Benefits:**
- Best of both worlds: infinite lines + dynamic label positioning

**Drawbacks:**
- Already the current implementation! The problem is that `line_bottom` is a PRICE level, not a viewport position. When viewport scales change, the price level `line_bottom` doesn't move with the viewport—it stays at the same price, which might now be off-screen.

**The Real Issue**: PineScript v6's `yloc.price` mode doesn't have a "viewport-relative percentage" option. We can't say "position this label at 30% below the bottom of the visible viewport in pixel space." We can only say "position at this price level" or "position relative to bar high/low."

**Approach 4: Hybrid - Use yloc.belowbar with Offset (BEST SOLUTION)**

Use `yloc.belowbar` to anchor labels to bars, but position them at the divider bar, not the label bar:

```pinescript
label.new(bar_pos, low[bar_index - bar_pos], "Q1", ..., yloc=yloc.belowbar, ...)
```

**How It Works:**
- Label is positioned at `bar_pos` (divider bar), not `bar_pos + 1`
- Y-coordinate is the low of the divider bar: `low[bar_index - bar_pos]`
- `yloc.belowbar` makes it viewport-relative
- Label appears at the bottom of the divider line's bar, staying with the line regardless of scale

**Benefits:**
- Labels stay with dividers visually
- Viewport-relative positioning prevents drift
- Simple implementation
- Maintains visual association with divider lines

**Drawbacks:**
- Labels are at bar lows, not "bottom of chart"
- But this is actually more intuitive! Labels mark the START of quarters, which is the bar position itself.

**Recommended Implementation**: Use Approach 4 (yloc.belowbar hybrid).

**Implementation Steps:**

1. **Change label Y-coordinate calculation** (lines 352, 363, 374, 385, 405, 417, 429, 441):
   - Replace `line_bottom` with `low[bar_index - bar_pos]` for historical (lines 352, 363, 374, 385)
   - Replace `line_bottom` with `low[bar_index - (cycle.q*_start_bar)]` for current (lines 405, 417, 429, 441)
   - Note: The array indexing `low[offset]` retrieves the low price from `offset` bars ago

2. **Change yloc parameter**:
   - Replace `yloc=yloc.price` with `yloc=yloc.belowbar`

3. **Adjust label X-position** (optional):
   - Current uses `bar_pos + 1` (one bar to the right) for label X-coordinate
   - Consider changing to `bar_pos` (same bar as divider) for tighter visual coupling
   - Or keep `+1` offset for readability if labels overlap with line

4. **Update label style** (optional):
   - Current uses `label.style_label_left` which aligns text to the left of the label anchor
   - Consider `label.style_label_center` for centered alignment on the divider
   - Or keep left alignment if using `+1` offset

5. **Remove line_bottom/line_top calculations** (optional):
   - Lines 315-319 calculate these values, but they're only used for line rendering now
   - If we keep `extend=extend.both`, lines extend infinitely regardless of `line_bottom`/`line_top` values
   - We still need Y-coordinates for `line.new()`, so keep the calculation but simplify it
   - Could use simple constants like `line.new(bar_pos, low, bar_pos, high, ..., extend=extend.both)` since extend makes the exact values irrelevant

**Testing Requirements:**
- Verify labels remain visible when scrolling Y-axis up/down
- Verify labels stay at divider bottoms when zooming Y-axis in/out
- Verify labels don't overlap with price bars
- Test on all 5 timeframes (M1, M5, M15, H1, H4)
- Test with different chart types (candlestick, bar, line)
- Test with logarithmic vs linear Y-axis scaling

### Technical Reference Details

#### Data Structures

**QuarterDividers UDT (lines 169-177):**
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

Stores line and label object references for all 4 quarters of one cycle. Used for TIER 1 (current dividers). One instance per cycle: `monthly_dividers`, `weekly_dividers`, `daily_dividers`, `session_dividers`, `micro_dividers` (lines 180-184).

**FractalCycle UDT (from TTL_Types/2 library):**
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

Tracks the state of one fractal cycle. Five instances: `monthly_cycle`, `weekly_cycle`, `daily_cycle`, `session_cycle`, `micro_cycle` (lines 83-87).

**Historical Storage Arrays:**
- **Bar position arrays** (lines 90-113): `hist_monthly_q1` through `hist_micro_q4` (20 arrays: 5 cycles × 4 quarters). Store `bar_index` values where quarter transitions occurred. Capped at 100 positions each.
- **Visual object arrays** (lines 116-159): `hist_monthly_q1_lines` and `hist_monthly_q1_labels` through `hist_micro_q4_lines` and `hist_micro_q4_labels` (40 arrays: 5 cycles × 4 quarters × 2 types). Store line and label object references for TIER 2 rendering. No size cap—filled/cleared on each render.

#### Key Functions

**f_process_cycle_quarters(cycle, current_q, prev_q, hist_q1, hist_q2, hist_q3, hist_q4)** (lines 196-254):
- Universal cycle processor for quarter transitions
- Detects Q1/Q2/Q3/Q4 starts by comparing `current_q` vs `prev_q`
- Updates cycle object (`cycle.q*_start_bar`, `cycle.q*_end_bar`, etc.)
- Archives bar positions into historical arrays
- Maintains 100-position cap per array
- Returns `current_q` (becomes next bar's `prev_q`)

**f_render_current_dividers(cycle, cycle_name, old_dividers)** (lines 389-446):
- TIER 1 renderer for current cycle dividers
- Checks rendering conditions (show settings, active cycle, barstate.islast)
- Deletes old line/label objects from previous pass
- Creates new line/label objects at current cycle's quarter positions
- Returns new QuarterDividers UDT instance

**f_render_historical_dividers(cycle_name, cycle, hist_q1_bars, ..., hist_q4_labels)** (lines 322-386):
- TIER 2 renderer for historical dividers
- Batch-deletes all old historical visual objects
- Iterates through historical bar position arrays
- Checks deduplication (not current cycle position)
- Checks lookback window (within divider_lookback bars)
- Creates new line/label objects for qualifying positions
- Pushes new objects into visual storage arrays

**f_should_show_cycle(cycle_name)** (lines 74-76):
- Determines if a cycle should render on current timeframe
- Checks if cycle toggle is enabled (e.g., `show_monthly`)
- Checks if cycle matches `active_cycle` (timeframe filtering)
- Returns boolean

**f_get_label_size(size_str)** (lines 47-48):
- Converts label size string ("tiny", "small", "normal", "large") to PineScript size constant
- Used for label rendering

#### Input Settings Affecting Rendering

**Visual Settings Group (lines 28-32):**
- `show_quarter_dividers`: Boolean, default true (line 28)
- `divider_lookback`: Int, default 500, min 50, max 2000 (line 29) - **NEEDS MODIFICATION**
- `show_quarter_labels`: Boolean, default true (line 30)
- `label_size`: String, default "small", options ["tiny", "small", "normal", "large"] (line 31)
- `divider_fractal_filter`: String, default "Auto-Detect", options ["Auto-Detect", "Monthly", "Weekly", "Daily", "Session", "Micro"] (line 32)

**Cycle Display Settings (lines 21-25):**
- `show_monthly`, `show_weekly`, `show_daily`, `show_session`, `show_micro`: Booleans, default true
- These enable/disable individual cycles, combined with timeframe filtering

**Cycle Colors (lines 35-40):**
- `divider_color_universal`: Color, default #4d4d4d (grey) with 70% transparency (line 35)
- Individual cycle colors for debug table only (not used for dividers in v7)

#### PineScript v6 Constraints

1. **Object Limits**: `max_boxes_count=500, max_lines_count=500, max_labels_count=500` (line 2)
   - Exceeding these limits causes silent failures (objects don't render)
   - Unlimited lookback with 5 cycles active could theoretically exceed limits
   - Only one cycle active per timeframe (filtering system) mitigates this

2. **Array Indexing**: `low[offset]` retrieves low price from `offset` bars ago
   - `low[0]` is current bar, `low[1]` is previous bar, etc.
   - To get low at specific bar_index, calculate offset: `low[bar_index - target_bar_index]`

3. **yloc Modes**:
   - `yloc.price`: Y-coordinate is absolute price level (current implementation)
   - `yloc.abovebar`: Y-coordinate is relative to bar high (viewport-relative)
   - `yloc.belowbar`: Y-coordinate is relative to bar low (viewport-relative)

4. **barstate.islast**: True only on the last bar of the chart
   - All rendering code executes only on this bar
   - Previous bars' visual objects persist until explicitly deleted

5. **extend Options**:
   - `extend.none`: Line is drawn only between specified coordinates
   - `extend.both`: Line extends infinitely in both directions (current implementation)
   - `extend.left`, `extend.right`: Extend in one direction only

#### File Locations

**Implementation Files:**
- Main indicator: `D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\01_Core_Cycles\TTL_v7_01_Cycles.pine`
- Test script: `D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\01_Core_Cycles\TTL_v7_01_Test.pine`

**Documentation:**
- Build log: `D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\docs\build_log.md`
- Coding session notes: `D:\Time Traders indicator\TTL  IndiProject files\TTL_v7_Rebuild\docs\coding_session_2025-01-07.md`

**Related Tasks:**
- Previous divider bug fix: `C:\Users\garic\TTL-Indi\sessions\tasks\h-fix-ttl-v7-divider-rendering.md`

**Library Dependencies:**
- TTL_Types/2: Published library (GmoneyT/TTL_Types/2) containing FractalCycle UDT
- TTL_CycleEngine/3: Published library (GmoneyT/TTL_CycleEngine/3) containing quarter calculation functions

### Implementation Recommendations

**For Unlimited Lookback:**

1. Change line 29 input setting:
   ```pinescript
   divider_lookback = input.int(500, "Divider Lookback (0=Unlimited)", minval=0, group="Visual Settings", tooltip="Number of bars to look back for historical dividers. Set to 0 for unlimited lookback (may impact performance on fast-cycling timeframes).")
   ```

2. Update lookback check in lines 348, 359, 370, 381:
   ```pinescript
   bool is_within_lookback = divider_lookback == 0 or (bar_index - bar_pos) <= divider_lookback
   if bar_pos != cycle.q*_start_bar and is_within_lookback
   ```

3. Consider reducing array cap from 100 to 62 in lines 213, 226, 238, 250:
   ```pinescript
   if array.size(hist_q1) > 62  // Keep last 62 (62*4*2=496 objects < 500 limit)
   ```

4. Add performance warning to documentation and tooltip

**For Fixed Label Positioning:**

1. Change all label Y-coordinates in historical rendering (lines 352, 363, 374, 385):
   ```pinescript
   label new_label = label.new(bar_pos, low[bar_index - bar_pos], "Q1", style=label.style_label_center, color=color.new(color.white, 100), textcolor=div_color, size=f_get_label_size(label_size), yloc=yloc.belowbar, textalign=text.align_center)
   ```

2. Change all label Y-coordinates in current rendering (lines 405, 417, 429, 441):
   ```pinescript
   new_q1_label := label.new(cycle.q1_start_bar, low[bar_index - cycle.q1_start_bar], "Q1", style=label.style_label_center, color=color.new(color.white, 100), textcolor=div_color, size=f_get_label_size(label_size), yloc=yloc.belowbar, textalign=text.align_center)
   ```

3. Consider removing `+ 1` offset from X-coordinates for tighter visual coupling

4. Update label style from `label.style_label_left` to `label.style_label_center`

5. Optionally simplify line_bottom/line_top calculations (lines 315-319) since labels no longer use them

**Testing Checklist:**

- [ ] Unlimited lookback (0) renders all historical dividers within array cap (62 per quarter)
- [ ] Limited lookback (e.g., 500) only renders dividers within window
- [ ] Labels remain visible when scrolling Y-axis vertically
- [ ] Labels remain at divider bottoms when zooming Y-axis in/out
- [ ] Labels don't overlap with price bars in normal conditions
- [ ] Performance acceptable on M1 chart with Micro cycles (fast-cycling)
- [ ] Performance acceptable on H4 chart with Monthly cycles (slow-cycling)
- [ ] Visual validation passes on all 5 timeframes (M1, M5, M15, H1, H4)
- [ ] No PineScript errors in console
- [ ] Object count limits not exceeded (check with unlimited lookback + fast cycles)

## User Notes
<!-- Any specific notes or requirements from the developer -->

## Work Log
<!-- Updated as work progresses -->
