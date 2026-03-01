+++
title = 'Как я потратил несколько часов на поиск слайсов с Trace.asyncBeginSection'
summary = 'Руководство по поиску асинхронных меток в Perfetto SQL. Разбираем разницу между thread_track и process_track, исправляем пустые таблицы в Macrobenchmark и учимся правильно работать с Trace.beginAsyncSection'
date = 2026-02-15T07:07:07+01:00
draft = false
+++

## Документация
> [Trace.beginSection/endSection](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:tracing/tracing/src/androidMain/kotlin/androidx/tracing/Trace.android.kt;l=123)
> [Trace.beginAsyncSection/endAsyncSection](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:tracing/tracing/src/androidMain/kotlin/androidx/tracing/Trace.android.kt;l=169)

## История

Нужно было собрать раставленные кастомные метрики через `Trace.beginAsyncSection` c кодовой базы с помощью `Тестов и Macrobenchmark` и вывести в один отчет после прогона тестов.

В [Macrobenchmark](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-overview) через [TraceMetric](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:benchmark/benchmark-macro/src/main/java/androidx/benchmark/macro/Metric.kt;l=486?q=TraceMetric&sq=) пытаюсь достать кастомные метрики по примеру с [ActivityResumeMetric](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:benchmark/benchmark-macro/src/main/java/androidx/benchmark/macro/Metric.kt;l=449?q=TraceMetric) 

#### Пример из документации, на который опирался
```SQL
 SELECT 
      slice.name as name,
      slice.ts as ts,
      slice.dur as dur
     FROM slice
        INNER JOIN thread_track on slice.track_id = thread_track.id
        INNER JOIN thread USING(utid)
        INNER JOIN process USING(upid)
        WHERE
            process.name LIKE ${captureInfo.targetPackageName}
                AND slice.name LIKE "activityResume"
```
##### Результат 

**Пустота**. Ни одной строчки не получил в таблице.
При этом я их вижу в `TraceViewer`. Метрики находятся в суперпозиции.

#### Детективное расследование 
Пошел еще раз в документацию и в описание написано `asynchronous events do not need to be nested`.
Но за этой фразой скрывается куда более важный для SQL-анализа факт: асинхронные события не привязаны к конкретному потоку (TID). Они привязаны к Процессу (PID).
Следовательно, в базе данных Perfetto они лежат не в `thread_track`, а в `process_track`. 
Когда мы делаем `INNER JOIN thread_track`, мы буквально говорим базе: Дай мне только то, что имеет `Thread ID`. А у асинхронного слайса его нет — он внутри своего процесса.

* Галлюцинации `AI` 
> `AI` мне говорит (на момент написания этого текста еще раз проверил), что это есть в документации, но на деле нету - так и живем.

##### Решение отсутствия метрик, снятых через Trace.beginAsyncSection/endAsyncSection

* Так работаем с метриками раставленными через `Trace.beginSection/endSection` , потому что они снимаются в рамках одного потока.
```SQL
  FROM slice
        INNER JOIN thread_track on slice.track_id = thread_track.id
        INNER JOIN thread USING(utid)
        INNER JOIN process USING(upid)
```

* А так работаем с `beginAsyncSection/endAsyncSection`.
```SQL
FROM slice
INNER JOIN process_track ON slice.track_id = process_track.id
INNER JOIN process USING(upid)
```

#### В чем разница? 
* **Асинхронные слайсы живут в process_track, а синхронные — в thread_track**

### Вывод 

Заранее смотрите, каким способом сняты метрики в коде, в зависимости от этого корректируете свои `SQL` к `Perfetto` таблицам. 


