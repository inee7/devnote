> Connector/J가 5.1 이상이고 서버 프리페어 스테이트먼트를 쓴다는 가정.

먼저 ~MySql의 Connector/J의 기본적인 데이타를 가져오는 방식을~ 보면 **클라이언트커서방식**이라고 DB에서 쿼리 결과를 모두 가져와서 애플리케이션 서버 메모리에 모두 올려 애플리케이션 단에서 커서를 이동하며 가져온다.🤔 배치로 돌아와서 JdbcCursorItemReader에 setFetchSize가 있는데 JDBC Statement에 fetchSize 옵션을 주는 일을 한다. Jdbc표준을 생각하고 개발하면 대량에 데이터에 OOM을 당할 때가 있다. 🤬 MySql 경우의 connector/j 는 표준 JDBC규격에 맞지 않게 statement에 fetchsize 옵션을 먹이지 못한다. 😰 따라서 아무리 fetchSize를 줘도 모든 row를 db에서 가져와서 배치 서버 메모리에서 하나하나 읽어 가며 처리한다. 대량 이면 OOM 난다. ( 참고로 이 방식은 ResultSet이 RowDataStatic 을 이용한다.) 그래서 대안으로 MySql connector/j 에서 제공 하는게 있다.

```java
stmt = conn.createStatement(java.sql.ResultSet.TYPE_FORWARD_ONLY, java.sql.ResultSet.CONCUR_READ_ONLY);
stmt.setFetchSize(Integer.MIN_VALUE);

```

을 만들면 스트리밍 방식이라고 커서가 한 행 씩 DB에서 가져와서 OOM은 나지 않을것이다. 다만 매 행 DB리소스를 가져 와야 하니 성능은 매우 느리다. 위 처리는 직접 jdbc 코드를 짤 때 해당 되는 거니 JdbcCursorItemReader에 적용하려면 JdbcCursorItemReader에 fetchSize로 Integer.MIN_VALUE만 주면 된다. (이때 ResultSet은 RowDataDynamic을 사용) 나머지 statement옵션은 이미 JdbcCursorItemReader이 적용하고 있다. (이건 HibernateCursorItemReader로 해결 못한다. 이유는 nagative size 를 막아놨다. 커스텀하면 가능한데 매우 로직 복잡해질듯)

```java
StatementImpl.java 코드를 보면 스트리밍 방식을 체크하는 로직이 있다.

/**
     * We only stream result sets when they are forward-only, read-only, and the
     * fetch size has been set to Integer.MIN_VALUE
     *
     * @return true if this result set should be streamed row at-a-time, rather
     *         than read all at once.
     */
    protected boolean createStreamingResultSet() {
        return ((this.resultSetType == java.sql.ResultSet.TYPE_FORWARD_ONLY) && (this.resultSetConcurrency == java.sql.ResultSet.CONCUR_READ_ONLY)
                && (this.fetchSize == Integer.MIN_VALUE));
    }

```

대량의 데이타면 스트리밍 방식이 너무 느려 환장할 노릇 일 것이다. 🤯 이럴 때는 서버커서를 이용하면 된다. 이 방식은 statement 옵션이 아니라 커넥션 자체의 옵션이다. 따라서 DataSource 객체에 옵션을 넣어야한다. (정확히는 connection)

```java
conn = DriverManager.getConnection("jdbc:mysql://localhost/?useCursorFetch=true", "user", "s3cr3t");
stmt = conn.createStatement();
stmt.setFetchSize(100);
rs = stmt.executeQuery("SELECT * FROM your_table_here");

```

이렇게 url뒤에 직접 옵션을 붙여도 되고

```java
dataSource.addDataSourceProperty("useCursorFetch", "true");

```

이런식으로 처리해도 된다. 이렇게 하면 statement에 적용된 fetchSize만큼 애플리케이션 메모리에 들고오고 처리하기를 반복한다. (이때 ResultSet은 RowDataCursor를 사용) 자, 그럼 대량의 데이타에서 서버커서를 이용해서 OOM을 피할거라는 기대를 안고 XXXCursorItemReader를 쓰면 .... 클라이언트 커서 방식을 피하지 못한다. 😳 ~~지금까지 잘 따라 왔는데 허무하지 않게 원인과 해결법을 찾아냈다.~~

디버깅해보며 따라가봤는데 AbstractCursorItemReader에 verifyCursorPosition라는 메소드가 있다.

```java
AbstractCursorItemReader.java

private void verifyCursorPosition(long expectedCurrentRow) throws SQLException {
		if (verifyCursorPosition) {
			if (expectedCurrentRow != this.rs.getRow()) {
				throw new InvalidDataAccessResourceUsageException("Unexpected cursor position change.");
			}
		}
	}

```

커서의 위치를 체크하는 로직인데 ResultSet에 getRow를 통해 현재 rowNumber를 가져온다. 이때 getRow내부에서는 앞에서 말한 RowData를 이용해서 결과를 가져온다. 클라이언트커서 : RowDataStatic 스트리밍방식: RowDataDynamic 서버커서방식: RowDataCursor 황당하게도 이 세가지에서 row number 가져오는 메소드 로직이 다른게 이 큰 원인이었다. RowDataCursor는 기대하는 row number보다 +1이 더 들어간다. (해당 클래스에 next와 getRow 두군데에서 +1씩 이루어져서 발생) 열심히 구글링 해보니 매우 소수로 이 문제 때문에 에러를 겪는 사례가 있더라. 심지어 mysql bug report에도 아주 옛날에 글이 있더라. [https://bugs.mysql.com/bug.php?id=78321](https://bugs.mysql.com/bug.php?id=78321) 그럼 이걸 해결 하기 위해 SpringBatch에 AbstractCursorItemReader 코드를 바꾸거나 Connector/J 에 RowData 코드를 바꿔야하는데... 아직 릴리즈 된것도 PR이 없더라...

.

.

.

.

.

이걸 해결하기 위한 방법을 찾아냈다 다행히도. `cursorItemReader.setVerifyCursorPosition(false);` 을 주면 AbstractCursorItemReader에서 커서 위치를 비교하지 않는다. 이 메소드 자체가 버그? 포인트라 무시하는게 좋은 것 같고 커서 위치가 다르면 어떡하지 라는 걱정이 있는데 음.... 코드를 쭉 따라갔을때는 그럴 포인트를 찾지 못했다. 그래서 대량의 데이타를 처리할 때

```java
dataSource.addDataSourceProperty("useCursorFetch", "true")

와

cursorItemReader.setVerifyCursorPosition(false);

```

를 적용 해보자.

참고: [MySQL :: MySQL Connector/J 8.0 Developer Guide :: 6.4 JDBC API Implementation Notes](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-implementation-notes.html)[허원철의 개발 블로그](https://heowc.dev/2019/02/09/using-mysql-jdbc-to-handle-large-table-1/)