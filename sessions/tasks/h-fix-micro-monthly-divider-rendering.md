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
<!-- Added by context-gathering agent -->

## User Notes
- User specifically requested: "seek for the best coding solutions while tackling this bug"
- Use code agent for fixes, then verify with code-review agent
- Reference v6 implementation as gold standard for correct behavior
- Focus on micro cycle first (more critical visual bug), then monthly cycle

## Work Log
<!-- Updated as work progresses -->
