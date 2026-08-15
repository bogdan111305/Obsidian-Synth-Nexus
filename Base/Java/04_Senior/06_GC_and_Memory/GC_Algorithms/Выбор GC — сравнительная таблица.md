# Выбор GC — сравнительная таблица

> Выбор GC — компромисс между throughput, latency и heap size, третьего не дано одновременно в максимуме. Эта заметка — точка входа: 6 алгоритмов рядом на одних осях. За механикой каждого — переходи по ссылкам в строках таблицы.

## Связанные темы
[[Epsilon GC]], [[Parallel и Serial GC]], [[G1GC — архитектура и tuning]], [[Shenandoah GC]], [[ZGC и Generational ZGC]], [[GC и латентность — практика]], [[Write Barriers и Card Table]], [[JVM флаги для GC]]

---

## Сравнительная таблица

| GC | Throughput | Типичная пауза | Heap диапазон | Concurrent фазы | Сложность tuning | Когда выбирать |
|---|---|---|---|---|---|---|
| [[Epsilon GC\|Epsilon]] | Максимальный (0% GC overhead) | Нет пауз — `OutOfMemoryError` при исчерпании heap | Любой, фиксированный (`Xms=Xmx`) | Нет (GC не запускается) | Нет | Benchmarking baseline, ultra-short-lived jobs, поиск утечек по факту OOM |
| [[Parallel и Serial GC\|Serial]] | Средний (однопоточный) | Секунды на Full GC, однопоточный mark-compact | < 100 МБ | Нет | Низкая | Embedded, однопоточные приложения, CI/тесты |
| [[Parallel и Serial GC\|Parallel]] | Максимальный среди "живых" GC | Minor: 50-200 мс; Full: 2-10 сек (heap 8 GB) | Любой, комфортно < 4 GB | Нет | Низкая | Batch/ETL, overnight reporting, throughput > latency |
| [[G1GC — архитектура и tuning\|G1]] | Высокий | 10-200 мс (мягкая цель `MaxGCPauseMillis`) | 4-100 GB | Marking (Initial/Remark — короткие STW, само marking — concurrent) | Средняя | Default-выбор: web API, стандартные сервисы, смешанный workload |
| [[Shenandoah GC\|Shenandoah]] | Высокий (немного ниже G1) | < 10 мс, на практике 1-5 мс | 4-64 GB | Marking + Evacuation + Update Refs — почти всё concurrent, 4 короткие STW-паузы | Средняя | Red Hat OpenJDK/Temurin, latency SLA < 10 мс, ZGC недоступен (Java 11/12) |
| [[ZGC и Generational ZGC\|ZGC (Generational)]] | Высокий (~4x non-gen после Java 21) | < 1 мс независимо от heap | 8 MB — 16 TB | Marking + Relocation — почти всё concurrent, 3 STW-паузы <1 мс | Низкая (сильный автотюнинг) | Latency SLA < 10 мс, heap > 32 GB (где G1 деградирует) |

## Решающее дерево

```
Известный bounded lifetime, GC вообще не нужен?         → Epsilon
Однопоточное приложение / heap < 100 МБ?                 → Serial
Batch/offline обработка, latency SLA нет?                 → Parallel
Latency SLA мягкий (>200 мс приемлемо)?                    → G1 (default)
Latency SLA 10-50 мс?                                       → G1 с tuning (см. [[GC и латентность — практика]])
Latency SLA < 10 мс, Java 11/12 или Red Hat build?           → Shenandoah
Latency SLA < 10 мс, heap > 32 GB?                            → ZGC (Generational)
```

## Throughput vs Latency — откуда берётся цена

Каждый шаг от Parallel → G1 → Shenandoah/ZGC снижает паузы, но не бесплатно:

- **Write/load barrier overhead** на каждую операцию со ссылками — растёт по мере ухода от чисто STW-алгоритмов к concurrent compacting. Сравнение overhead по типам барьеров — [[Write Barriers и Card Table]]#Сравнение подходов.
- **CPU, отданный GC-потокам** параллельно с приложением (`ConcGCThreads`) — на CPU-saturated системе (мало ядер, контейнеры с 1-2 vCPU) concurrent GC может не успевать и деградировать хуже, чем простой STW-алгоритм.
- **Memory overhead**: Shenandoah — Brooks pointer, +8 байт на каждый объект; ZGC — 3x virtual address space под colored pointers (не физическая память, но ограничение на 32-bit/urезанный VA).

Итог: G1 — единственный из трёх "современных" GC без per-object overhead и без постоянной конкуренции за CPU с приложением — поэтому он default, а не Shenandoah/ZGC.

## Heap size — почему G1 не масштабируется бесконечно

G1 делит heap на регионы фиксированного числа (`heap / 2048`), и cross-region ссылки отслеживаются через Remembered Sets (RSet) — их объём растёт с heap и после определённого порога (~30-64 GB) съедает всё больше времени STW-пауз на сканирование. ZGC не имеет этой проблемы: паузы не зависят от heap size в принципе, поэтому на heap > 32 GB он предпочтительнее даже при том же latency SLA, что и G1 закрывает на меньшем heap.

---

## Вопросы на интервью
- По каким трём осям сравнивают GC-алгоритмы и почему нельзя максимизировать все три одновременно?
- Почему G1 деградирует на heap > 32 GB, а ZGC — нет?
- Какой GC выбрать для batch ETL-джобы на heap 16 GB без latency SLA и почему не G1?
- Чем Shenandoah отличается от ZGC при одинаковом требовании < 10 мс, и когда предпочесть первый?
- Почему G1 — default, если Shenandoah/ZGC дают паузы на порядок меньше?
