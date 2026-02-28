+++
title = 'Dict SQL Perfetto'
date = 2026-02-18T07:07:07+01:00
draft = false
+++

### Intro

> A reference guide for handy SQL queries to avoid keeping them all in memory (WIP).

### Documentation
> [Perfetto SQL Getting Started](https://perfetto.dev/docs/analysis/perfetto-sql-getting-started)

### SyncFrameState Latency Analysis

> This query identifies how long the `HardwareRenderer` blocks the **Main Thread** to hand over data to the **RenderThread**. If the duration exceeds 2–3ms, it typically indicates too many new textures or excessive changes in the `DisplayList` within a single frame.

```SQL
SELECT
  ts,
  dur / 1e6 AS dur_ms,
  name
FROM slice
WHERE name = 'syncFrameState'
  AND dur / 1e6 > 2.0
ORDER BY dur DESC
```

### Expected vs. Actual Jank Mapping

> A query targeting the Frame Timeline system tables to find frames where the actual duration exceeded the expected duration by more than 20%.

```SQL
SELECT
  act.ts,
  act.dur / 1e6 AS actual_dur_ms,
  exp.dur / 1e6 AS expected_dur_ms,
  (act.dur - exp.dur) / 1e6 AS jank_delta_ms
FROM actual_frame_timeline_slice act
JOIN expected_frame_timeline_slice exp ON act.display_frame_token = exp.display_frame_token
WHERE act.dur > exp.dur * 1.2
ORDER BY jank_delta_ms DESC
```

## Monitor Contention Analysis


### Monitor Contention Analysis (Grouped with Event Count)


- Identifies threads holding monitors (locks).
- Displays the Owner Thread (the blocker).
- Displays the Blocked Thread (the victim).
- Extracts the Method Name.
- Shows the Occurrence Count.
- Calculates Total Duration.
- Finds the Maximum Duration per slice.
- Shows the Process Name.
- Note: Replace `your_main_thread` with your app's actual main thread name.


```SQL
SELECT
  SUBSTR(
    s.name, 
    INSTR(s.name, 'owner ') + 6, 
    INSTR(s.name, ' (') - (INSTR(s.name, 'owner ') + 6)
  ) AS owner_thread_name,
  t.name AS blocked_thread,
SUBSTR(
    SUBSTR(s.name, INSTR(s.name, ' at ') + 4), 
    1, 
    INSTR(SUBSTR(s.name, INSTR(s.name, ' at ') + 4), '(') - 1
  ) AS method_name,
  s.name AS lock_details,
  p.name AS process_name,
  COUNT(*) AS occurrence_count,
  SUM(dur) / 1e6 AS total_dur_ms,
  MAX(dur) / 1e6 AS max_single_dur_ms
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t USING (utid)
JOIN process p USING (upid)
WHERE s.name LIKE 'monitor contention%' AND blocked_thread LIKE 'your_main_thread%'
GROUP BY owner_thread_name, blocked_thread, lock_details, process_name
ORDER BY total_dur_ms DESC
```