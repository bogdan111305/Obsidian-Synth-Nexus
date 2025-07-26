# Работа с большими данными в JDBC

## Оглавление
1. [Streaming ResultSet и fetch size](#streaming)
2. [Scrollable и updatable ResultSet](#scrollable)
3. [Работа с BLOB/CLOB](#blob)
4. [Best practices](#best-practices)
5. [FAQ](#faq)

---

## 1. Streaming ResultSet и fetch size <a name="streaming"></a>
- Для больших выборок используйте setFetchSize и TYPE_FORWARD_ONLY
- Пример:
```java
Statement stmt = conn.createStatement(ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
stmt.setFetchSize(1000); // или Integer.MIN_VALUE для MySQL streaming
ResultSet rs = stmt.executeQuery("SELECT * FROM big_table");
while (rs.next()) {
    // обработка
}
```
- Для PostgreSQL/MySQL: используйте streaming для экономии памяти

## 2. Scrollable и updatable ResultSet <a name="scrollable"></a>
- Позволяет перемещаться по результатам в любом направлении
- Пример:
```java
Statement stmt = conn.createStatement(ResultSet.TYPE_SCROLL_INSENSITIVE, ResultSet.CONCUR_UPDATABLE);
ResultSet rs = stmt.executeQuery("SELECT * FROM users");
rs.last();
int rowCount = rs.getRow();
rs.beforeFirst();
```

## 3. Работа с BLOB/CLOB <a name="blob"></a>
- Для больших файлов используйте getBinaryStream/setBinaryStream
- Пример:
```java
PreparedStatement ps = conn.prepareStatement("INSERT INTO files (data) VALUES (?)");
ps.setBinaryStream(1, new FileInputStream(file), file.length());
ps.executeUpdate();
```
- Для чтения:
```java
ResultSet rs = stmt.executeQuery("SELECT data FROM files WHERE id=?");
if (rs.next()) {
    InputStream in = rs.getBinaryStream(1);
    // чтение потока
}
```

## 4. Best practices <a name="best-practices"></a>
- Используйте streaming для больших выборок
- Не загружайте большие файлы целиком в память
- Используйте try-with-resources для потоков
- Для BLOB/CLOB — работайте с потоками, а не массивами

## 5. FAQ <a name="faq"></a>
- Как обработать таблицу с миллионами строк?
- Как загрузить/выгрузить большой файл через JDBC?
- Как узнать размер результата до выборки?
- Как избежать OutOfMemory при работе с большими данными?

---

**Рекомендуемые материалы:**
- [JDBC Streaming Large Data](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-implementation-notes.html#connector-j-reference-implementation-notes-streaming)
- [PostgreSQL Large Objects](https://jdbc.postgresql.org/documentation/head/binary-data.html) 