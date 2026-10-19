---
layout: post
title: "문자열 연결 쿼리의 한계 넘기 - PreparedStatement와 플레이스홀더(?)"
date: 2026-10-19 09:10:00 +0900
categories: [Java]
tags: [java, jdbc, preparedstatement, placeholder, sql-injection]
mermaid: true
---

## 들어가며 (Situation)

지난 글에서 `Statement`에 입력값을 문자열로 이어 붙여 쿼리를 만들면 SQL Injection에 취약하다는 점을 짚고 넘어갔습니다. `chap02-statement` 프로젝트의 `b_preparedstatements` 패키지가 그 해결책인 **`PreparedStatement`** 를 다룹니다. 앞선 `Application01`, `02`를 `PreparedStatement`로 바꿔 보는 실습입니다.

> 실제 서비스에서 겪은 문제가 아니라, 수업 실습을 정리한 글입니다. 동작 설명은 공식 문서로 보완했습니다. 실행 결과(조회된 메뉴 데이터)는 싣지 않았습니다.

## 문제 상황 (Task)

- 쿼리에 자바 변수를 넣을 때 따옴표(`'" + menuName + "'"`)를 직접 붙여야 하는 불편함을 없앨 수 있을까요?
- 입력값이 쿼리의 구조를 바꾸지 못하게 하려면 어떻게 해야 할까요?
- 수업 주석은 "`PreparedStatement`는 컴파일된 SQL을 재사용해 성능이 향상된다"고 설명했는데, 항상 그럴까요?

## 해결 과정 (Action)

### 1. Statement vs PreparedStatement

| 항목 | `Statement` | `PreparedStatement` |
|------|-------------|---------------------|
| 쿼리 작성 | 실행 시점에 `executeQuery(sql)`로 문자열 전달 | 생성 시점에 `prepareStatement(sql)`로 미리 전달 |
| 변수 삽입 | 문자열 연결 (`'" + 변수 + "'"`) | `?` 자리에 `setXxx()`로 값 바인딩 |
| 실행 | `stmt.executeQuery(sql)` | `pstmt.executeQuery()` (인자 없음) |
| 입력값 취급 | SQL 코드의 일부가 될 수 있음 | 값으로만 전달 |

### 2. 전체 조회: 인자가 사라진 executeQuery

```java
pstmt = con.prepareStatement("SELECT * FROM TBL_MENU");
rset = pstmt.executeQuery();
```

쿼리는 `prepareStatement`에 넘기고, `executeQuery()`에는 인자가 없습니다. 이 쿼리에는 `?`가 없어서 `Statement`와 결과가 같습니다.

### 3. 플레이스홀더로 조건 바인딩

```java
pstmt = con.prepareStatement(
    "SELECT MENU_CODE, MENU_PRICE , ORDERABLE_STATUS FROM TBL_MENU WHERE MENU_NAME = ?");

// ? 의 갯수에 따라 parameterIndex 가 결정된다
pstmt.setString(1, menuName);

rset = pstmt.executeQuery();
```

```mermaid
flowchart LR
    A["SQL 틀<br/>WHERE MENU_NAME = ?"] --> B["setString(1, 입력값)"]
    B --> C["executeQuery()"]
    C --> D["ResultSet"]
```

수업 주석이 정리한 규칙은 다음과 같습니다.

- `?`는 **플레이스홀더**입니다. 자바 변수를 직접 삽입하는 대신 `?`로 두고 메서드로 채웁니다.
- 파라미터 인덱스는 **1부터** 시작하며 `?`의 등장 순서를 따릅니다. 0이 아니라는 점이 배열과 다릅니다.
- `?`는 반드시 `set` 메서드로 모두 채워야 합니다. 값을 채우지 않으면 실행 시 `SQLException`이 납니다.
- 타입에 맞는 메서드를 씁니다. 문자열은 `setString`, 정수는 `setInt`입니다.

### 4. SQL Injection이 막히는 이유

`Statement`에서는 입력 `x' OR '1'='1`이 쿼리 문자열의 일부가 되어 `WHERE` 조건을 바꿨습니다. `PreparedStatement`에서는 입력이 **값**으로만 전달되므로, 같은 입력이 "메뉴 이름이 `x' OR '1'='1`인 행"을 찾는 조건이 될 뿐입니다. (예시, 수업에서 실행하지 않음) 이 때문에 따옴표를 직접 붙일 필요도 없고, 입력에 따옴표가 있어도 쿼리가 깨지지 않습니다.

### 5. 수업 주석의 "성능 향상"은 어디까지 맞을까

수업 주석은 이렇게 설명했습니다. `Statement`는 매번 SQL이 DB로 전송되어 파싱과 컴파일을 거치지만, `PreparedStatement`는 최초 1회만 컴파일하고 같은 SQL을 다시 요청하면 재사용해서 성능이 좋아진다고 했습니다.

이 설명은 **서버 측 prepared statement**의 동작입니다. 제가 이해한 바로는 MySQL Connector/J는 기본값이 클라이언트 측에서 `?`를 값으로 치환해 보내는 방식(`useServerPrepStmts=false`)이어서, 기본 설정에서는 이 이점이 그대로 적용되지 않을 수 있습니다. 서버 측 준비 구문을 쓰려면 접속 URL에 `useServerPrepStmts=true` 같은 옵션이 필요하다고 알고 있습니다. 이번에 직접 측정하거나 설정을 바꿔 보지는 않았으므로, 공식 문서로 확인이 필요한 부분입니다.

| 이점 | 기본 설정에서 | 비고 |
|------|---------------|------|
| SQL Injection 방지 | 적용됨 | 바인딩으로 값과 구조를 분리 |
| 따옴표·이스케이프 처리 편의 | 적용됨 | `setString`이 처리 |
| 컴파일 재사용(성능) | 설정에 따라 다름 | 서버 측 준비 구문과 캐시 설정 필요 |

정리하면 `PreparedStatement`를 쓰는 가장 확실한 이유는 성능보다 **안전성과 편의성**입니다.

### 6. 수업 코드에서 눈에 띈 점

- 자원 해제는 `close(con)`, `close(rset)`, `close(pstmt)` 순서로, 지난 글에서 짚은 역순 해제와 같은 이슈가 있습니다.
- `PreparedStatement`를 인자로 받는 `close` 오버로딩은 `Statement` 타입 `close`를 그대로 쓸 수 있습니다. `PreparedStatement`가 `Statement`의 하위 인터페이스이기 때문입니다.
- 결과가 없을 때의 안내 메시지는 아직 없습니다.

## 결과 (Result)

정량 지표는 없고, 아래를 정리했습니다.

- 입력값은 `?` 플레이스홀더에 `setXxx(인덱스, 값)`으로 바인딩하며, 인덱스는 1부터 시작합니다.
- 바인딩 덕분에 입력이 SQL 구조를 바꾸지 못해 SQL Injection이 방어됩니다.
- "컴파일 재사용으로 빨라진다"는 서버 측 준비 구문 설정에 달려 있어, 도입 이유를 성능으로만 설명하면 부정확합니다.

## 더 학습하면 좋은 개념

- **Connector/J의 `useServerPrepStmts`와 `cachePrepStmts`** — 컴파일 재사용을 실제로 얻으려면 필요한 옵션들이며, 수업 주석과 기본 동작의 차이를 이해하는 열쇠입니다.
- **SQL Injection 방어의 한계** — 컬럼명이나 `ORDER BY`처럼 `?`로 바인딩할 수 없는 자리는 별도 검증이 필요합니다.
- **`try-with-resources`** — `PreparedStatement`와 `ResultSet`의 자동 해제로 `close` 호출 순서 실수를 없앱니다.
- **배치 실행(`addBatch` / `executeBatch`)** — 같은 쿼리를 값만 바꿔 여러 번 실행할 때 효과적인 기능입니다.
- **OWASP 입력 검증과 쿼리 파라미터화** — 보안 관점에서 `PreparedStatement`가 왜 기본 규칙인지 정리해 둔 자료입니다.

## 참고 자료

- [Java SE API - java.sql.PreparedStatement](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/PreparedStatement.html)
- [Oracle - JDBC Basics: Using Prepared Statements](https://docs.oracle.com/javase/tutorial/jdbc/basics/prepared.html)
- [MySQL Connector/J - Connection Properties (useServerPrepStmts)](https://dev.mysql.com/doc/connector-j/en/connector-j-reference-configuration-properties.html)
- [MySQL 8.4 - Prepared Statements](https://dev.mysql.com/doc/refman/8.4/en/sql-prepared-statements.html)
- [OWASP - Query Parameterization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html)
