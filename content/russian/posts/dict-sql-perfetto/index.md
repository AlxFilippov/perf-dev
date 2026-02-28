+++
title = 'Справочник SQL Perfetto'
date = 2026-02-18T07:07:07+01:00
draft = false
+++

### Интро

> Справочник, где копятся удобные SQL запросы, чтобы не держать их в памяти (в процессе заполнения)

### Документация
> [Perfetto SQL Getting Started](https://perfetto.dev/docs/analysis/perfetto-sql-getting-started)

### Анализ задержки синхронизации (SyncFrameState)

>Этот запрос покажет, как долго HardwareRenderer блокирует Main Thread для передачи данных в RenderThread. Если время > 2-3 мс, значит, в кадре слишком много новых текстур или изменений в DisplayList.

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

### Сопоставление Expected vs Actual Jank

> Запрос к системным таблицам Frame Timeline, который находит кадры, где реальная длительность превысила ожидаемую более чем на 20%.

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

## Анализ Monitor contention


### Анализ monitor contention с группировкой и подсчетом количества событий

> * Находит потоки, которые удерживают мониторы (locks).
> * Выводит поток, который блокирует
> * Выводит поток, которого блокируют
> * Имя метода
> * Количество блокировок
> * Общий Duration
> * Максимальный Duration за слайсы
> * Имя процесса
> * `your_main_thread` заменить на имя `main_thread` вашего приложения


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