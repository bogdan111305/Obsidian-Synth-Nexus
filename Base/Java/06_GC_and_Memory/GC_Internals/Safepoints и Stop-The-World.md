# Safepoints и Stop-The-World

> **Safepoint** — момент выполнения, в котором JVM может безопасно остановить поток: стек известен, регистры разобраны, объекты на куче целостны. **STW (Stop-The-World)** — пауза всех потоков для выполнения GC-фазы или иного глобального действия.

## Связанные темы
[[GC Roots и достижимость объектов]], [[Write Barriers и Card Table]], [[G1GC — архитектура и tuning]], [[ZGC и Generational ZGC]], [[Safepoints и Stop-The-World]]

---

## Что такое Safepoint

Поток достигает safepoint:
- **Between bytecodes** — интерпретатор проверяет флаг на каждом обратном переходе (backbranch)
- **At method return** — возврат из метода
- **At loop back-edge** — в скомпилированном коде C2 вставляет `safepoint poll` в заголовок цикла
- **At allocation** — при попытке аллокации

JIT-компилятор C2 вставляет `safepoint poll`: чтение из специальной страницы памяти (`polling page`). Когда JVM хочет STW — страница выставляется как non-readable → SIGSEGV → signal handler переводит поток в safepoint.

```
// Псевдокод скомпилированного цикла с safepoint poll
loop:
  mov rax, [polling_page]   // safepoint check (страница readable → continue)
  ; ... тело цикла
  jnz loop
```

## Stop-The-World — механизм

1. JVM выставляет флаг `SafepointSynchronize::_state = synchronizing`
2. Polling page → non-readable
3. Каждый поток при следующем poll получает SIGSEGV → входит в safepoint handler → suspended
4. JVM ждёт пока **все** Java-потоки остановятся (`time-to-safepoint`)
5. Выполняет GC или иное действие
6. Возобновляет все потоки

> [!WARNING]
> **Time-to-safepoint** — время ожидания, пока все потоки дойдут до ближайшего poll. Длинный native метод или "safepoint-hostile loop" (without polls) задерживает STW. Это отдельная latency overhead, не считая самой GC-паузы.

## Что происходит во время STW

| Операция | Требует STW | Примечание |
|----------|-------------|------------|
| G1 Minor GC (Young) | Да | Evacuation молодого поколения |
| G1 Initial Mark | Да | Короткая пауза для начала concurrent marking |
| G1 Remark | Да | Завершение marking |
| G1 Mixed GC | Да | Evacuation mixed regions |
| ZGC — большинство фаз | Нет | Concurrent |
| ZGC Init Mark / Remap | Да | <1 мс |
| Thread stack scanning | Да | Нужны стоп-потоки для сканирования стеков |
| Class unloading | Да | |
| Deoptimization | Да | |
| Biased lock revocation | Да | Поэтому biased locking убрали в Java 15+ |
| Heap inspection (jmap) | Да | Полная STW — не делать на проде |

## Safepoint-hostile паттерны

```java
// Проблема: C2 может НЕ вставить safepoint poll в counted loop
// (JVM считает её конечной и предполагает быстрый выход)
for (int i = 0; i < Integer.MAX_VALUE; i++) {
    // тяжёлые вычисления без вызовов методов
    sum += i;
}
// Результат: поток не входит в safepoint → STW пауза растёт

// Решение в Java 10+: JVM вставляет safepoints в counted loops по умолчанию
// -XX:+UseCountedLoopSafepoints (по умолчанию true с Java 10)
```

## Диагностика

```bash
# Включить логирование safepoint пауз
-Xlog:safepoint=info

# Детальный лог: причина + time-to-safepoint + cleanup time + sync time
-Xlog:safepoint*=debug

# Пример вывода:
# [gc,safepoint] Application time: 0.1234567 seconds
# [gc,safepoint] Entering safepoint region: G1CollectForAllocation
# [gc,safepoint] Leaving safepoint region
# [gc,safepoint] Total time for which application threads were stopped: 0.0023456 seconds
```

```bash
# JFR event: jdk.SafepointBegin, jdk.SafepointEnd
# Смотреть: time_to_safepoint — если > 10 мс, есть safepoint-hostile код
jcmd <pid> JFR.start duration=60s filename=safepoints.jfr
jfr print --events jdk.SafepointBegin,jdk.SafepointEnd safepoints.jfr
```

## Операции, НЕ требующие safepoint

Concurrent GC фазы (G1 Concurrent Marking, ZGC, Shenandoah) работают параллельно с приложением. Это достигается через:
- **Write barriers** — приложение уведомляет GC об изменениях ссылок
- **Colored pointers** (ZGC) — состояние объекта в битах самого pointer
- **Brooks pointers** (Shenandoah) — forwarding pointer в заголовке объекта

---

## Вопросы на интервью
- Что такое safepoint и как JVM останавливает потоки?
- Почему STW пауза = time-to-safepoint + actual GC time?
- Что такое safepoint-hostile loop? Как диагностировать?
- Какие операции всегда требуют STW, а какие нет?
- Зачем убрали biased locking в Java 15?

## Подводные камни
- `jmap -histo` и `jmap -dump` требуют полного STW — недопустимо на продакшн-сервисе
- Длинные native методы (JNI) не входят в safepoint пока не вернутся в Java — JVM их не ждёт, но native thread не может двигать объекты
- `-XX:+UseCountedLoopSafepoints` включён по умолчанию с Java 10, но в старых версиях это была частая причина задержанных пауз
- Метрики GC логов показывают только GC время, без time-to-safepoint — надо логировать safepoint отдельно
