# Работа с большими данными в JDBC

> [!QUOTE] Два основных инструмента для больших данных в JDBC: `setFetchSize` + `TYPE_FORWARD_ONLY` для больших выборок (streaming ResultSet), и `setBinaryStream` / `getBinaryStream` для BLOB/CLOB файлов.

## Оглавление
1. [[#Streaming ResultSet и fetch size]]
2. [[#Scrollable и Updatable ResultSet]]
3. [[#Работа с BLOB и CLOB]]
4. [[#Best practices]]
5. [[#FAQ]]

---

## Streaming ResultSet и fetch size

По умолчанию JDBC загружает весь `ResultSet` в память. Для таблиц с миллионами строк это приведёт к `OutOfMemoryError`.

**Решение: streaming (построчная обработка)**

```java
Statement stmt = conn.createStatement(
    ResultSet.TYPE_FORWARD_ONLY,
    ResultSet.CONCUR_READ_ONLY
);
stmt.setFetchSize(1000);    // PostgreSQL, SQL Server: порциями по 1000 строк
// stmt.setFetchSize(Integer.MIN_VALUE); — для MySQL: включает истинный streaming

ResultSet rs = stmt.executeQuery("SELECT * FROM big_table");
while (rs.next()) {
    // обрабатываем строку — не храним всё в памяти
    process(rs.getString("column"));
}
```

> [!WARNING] Для MySQL streaming (`setFetchSize(Integer.MIN_VALUE)`) — соединение блокируется до завершения обхода ResultSet. Нельзя выполнять другие запросы через это же соединение во время чтения. Используйте отдельное `Connection`.

> [!INFO] `TYPE_FORWARD_ONLY` + `CONCUR_READ_ONLY` — обязательные флаги для streaming. `TYPE_SCROLL_INSENSITIVE` буферизует все строки в памяти, что отменяет весь смысл streaming.

---

## Scrollable и Updatable ResultSet

Позволяет перемещаться по результату в любом направлении и изменять данные:

```java
Statement stmt = conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE,
    ResultSet.CONCUR_UPDATABLE
);
ResultSet rs = stmt.executeQuery("SELECT * FROM users");

rs.last();
int totalRows = rs.getRow();    // количество строк без COUNT(*)
rs.beforeFirst();               // вернуться к началу

// обновление строки через ResultSet
while (rs.next()) {
    if ("inactive".equals(rs.getString("status"))) {
        rs.updateString("status", "archived");
        rs.updateRow();
    }
}
```

> [!WARNING] Scrollable ResultSet загружает все строки в память (на стороне клиента или драйвера). Не используйте для больших выборок.

---

## Работа с BLOB и CLOB

**BLOB** (Binary Large Object) — бинарные данные (файлы, изображения).
**CLOB** (Character Large Object) — текстовые данные большого объёма.

Запись BLOB:

```java
try (PreparedStatement ps = conn.prepareStatement(
        "INSERT INTO files (name, data) VALUES (?, ?)")) {
    ps.setString(1, file.getName());
    ps.setBinaryStream(2, new FileInputStream(file), file.length());
    ps.executeUpdate();
}
```

Чтение BLOB:

```java
try (PreparedStatement ps = conn.prepareStatement(
        "SELECT data FROM files WHERE id = ?")) {
    ps.setInt(1, fileId);
    try (ResultSet rs = ps.executeQuery()) {
        if (rs.next()) {
            try (InputStream in = rs.getBinaryStream("data");
                 OutputStream out = new FileOutputStream("output.bin")) {
                byte[] buffer = new byte[8192];
                int bytesRead;
                while ((bytesRead = in.read(buffer)) != -1) {
                    out.write(buffer, 0, bytesRead);
                }
            }
        }
    }
}
```

> [!WARNING] Не читайте весь BLOB в `byte[]` через `rs.getBytes()` для больших файлов — это загружает файл целиком в heap. Всегда используйте потоки (`InputStream` / `OutputStream`).

> [!INFO] Для CLOB используйте `getCharacterStream()` / `setCharacterStream()` — аналог BLOB, но для текста (`Reader` / `Writer`).

---

## Best practices

- `setFetchSize` + `TYPE_FORWARD_ONLY` + `CONCUR_READ_ONLY` для больших выборок
- Обрабатывайте строки потоково — не накапливайте в `List`
- BLOB/CLOB — только через потоки, не через `getBytes()` / `getString()`
- `try-with-resources` для потоков внутри `ResultSet`
- Для MySQL streaming: отдельное соединение на время обхода
- Scrollable ResultSet только для небольших наборов данных

---

## FAQ

**Как обработать таблицу с миллионами строк?**
`setFetchSize(1000)` + `TYPE_FORWARD_ONLY`. Обрабатывайте строки в цикле без накопления в памяти.

**Как загрузить/выгрузить большой файл через JDBC?**
BLOB + `setBinaryStream` / `getBinaryStream`. Передавайте `InputStream` напрямую в `PreparedStatement`.

**Как избежать OutOfMemoryError при больших выборках?**
Streaming ResultSet. Для MySQL — `Integer.MIN_VALUE`, для PostgreSQL — любое положительное значение `setFetchSize`.

**Как узнать количество строк до выборки?**
Выполните отдельный `SELECT COUNT(*) FROM table WHERE ...` перед основным запросом.

---

**Связанные заметки:** [[Основы]] | [[Транзакции в JDBC]] | [[Ошибки и диагностика в JDBC]]

**Материалы:**
- [MySQL Streaming Large Results](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-implementation-notes.html)
- [PostgreSQL Binary Data](https://jdbc.postgresql.org/documentation/head/binary-data.html)
