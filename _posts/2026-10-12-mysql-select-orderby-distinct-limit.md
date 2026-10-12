---
layout: post
title: "원하는 열을, 원하는 순서로, 원하는 만큼 - SELECT·ORDER BY·DISTINCT·LIMIT"
date: 2026-10-12 10:00:00 +0900
categories: [MySQL]
tags: [mysql, sql, select, order-by, limit]
mermaid: true
---

## 들어가며 (Situation)

MySQL 수업의 첫 단원은 `select`입니다. 수업 주석에 "SQL의 90% 이상을 차지하는 문법"이라고 적혀 있을 만큼 모든 쿼리의 뼈대가 됩니다. 이 글은 `01_select`, `02_orderby`, `04_distinct`, `05_limit` 실습을 한 흐름으로 묶어 정리한 것입니다. 실습 DB는 메뉴(`tbl_menu`)와 카테고리(`tbl_category`) 테이블입니다.

> 실제 서비스에서 겪은 문제가 아니라, 수업 실습을 정리한 글입니다. 쿼리는 수업에서 직접 실행하며 확인했고, 동작 설명은 MySQL 공식 문서로 보완했습니다.

## 문제 상황 (Task)

- `select`로 열을 고르고 `as`로 이름을 바꾸는 것까지는 쉽습니다. 그런데 `order by`에서 `null`이 있는 열을 정렬하면 `null`은 맨 위일까요, 맨 아래일까요?
- `distinct`는 "중복 제거"라고만 배웠는데, 열이 여러 개일 때는 무엇을 기준으로 중복을 판단할까요?
- `limit 1, 5`는 "1번부터 5개"일까요, "2번째부터 5개"일까요?

## 해결 과정 (Action)

### 1. 쿼리는 쓴 순서대로 실행되지 않는다

수업에서는 `select 무엇을 from 어디에서`로 외웠지만, 실제 처리 순서는 다릅니다. 아래 순서를 알면 뒤에서 나올 "별칭을 어디서 쓸 수 있는가"가 설명됩니다.

```mermaid
flowchart LR
    A["FROM<br/>테이블 선택"] --> B["WHERE<br/>행 필터"]
    B --> C["GROUP BY / HAVING<br/>그룹화·그룹 필터"]
    C --> D["SELECT<br/>열 선택·별칭"]
    D --> E["DISTINCT<br/>중복 제거"]
    E --> F["ORDER BY<br/>정렬"]
    F --> G["LIMIT<br/>개수 제한"]
```

이 흐름은 개념적인 처리 순서입니다. 실제 실행은 옵티마이저가 최적화하므로 내부 동작과 완전히 같지는 않습니다.

### 2. SELECT: 열 고르기와 별칭

```sql
select
    menu_name as '메뉴 이름',
    menu_price as '가격'
from
    tbl_menu;
```

| 문법 | 의미 | 주의점 |
|------|------|--------|
| `select *` | 모든 열 조회 | 학습·확인용으로는 편하지만, 필요한 열만 적는 쪽이 의도가 분명합니다 |
| `as 별칭` | 결과 열의 이름 지정 | 공백이 들어가면 따옴표나 백틱이 필요합니다 |
| `now()`, `concat()` | 테이블 없이 쓰는 내장 함수 | `from` 없이도 `select`가 동작합니다 |

수업 코드는 별칭에 작은따옴표(`'메뉴 이름'`)를 쓰고, 정렬 예제에서는 백틱(`` `연산결과` ``)을 씁니다. 별칭에는 둘 다 쓸 수 있지만, 백틱은 원래 열·테이블 같은 **식별자**를 감싸는 용도라서 식별자를 다시 참조하는 곳에서는 백틱이 더 안전합니다.

### 3. ORDER BY: 별칭으로 정렬이 되는 이유

```sql
select
    menu_code,
    menu_price,
    menu_code * menu_price as `연산결과`
from
    tbl_menu
order by
    `연산결과` desc;
```

`order by`는 `select` **뒤에** 실행되므로 `select`에서 만든 별칭을 쓸 수 있습니다. 반대로 `where`는 `select`보다 먼저 실행되어서 별칭을 쓸 수 없습니다. (표준 SQL은 `where`에서의 별칭 참조를 허용하지 않는다고 공식 문서에 적혀 있습니다.) 정렬 방향의 기본값은 `asc`(오름차순)입니다.

### 4. NULL 정렬: 수업 주석을 따라가 보기

`tbl_category`의 `ref_category_code`에는 `null`이 있습니다. 수업은 세 가지 쿼리를 차례로 보여 줍니다.

| 쿼리 | 결과의 의미 |
|------|-------------|
| `order by ref_category_code asc` | MySQL은 `NULL`을 가장 작은 값으로 취급하므로 **`null`이 맨 위** |
| `order by ref_category_code is null asc` | `is null`의 결과는 0(거짓) 또는 1(참)이라서, `null`이 아닌 행(0)이 먼저 나오고 **`null`이 뒤로** 밀림 |
| `order by ref_category_code desc` (예시) | `null`이 맨 아래 |

수업 주석 "is null 오름차순 시 null 값이 뒤로 빠지고 오름차순이 된다"는 이 원리 때문입니다. 정확히는 "`null` 여부로 먼저 정렬하고, 그 안에서는 정렬하지 않는다"입니다. `null`을 뒤로 보내면서 값도 오름차순으로 정렬하려면 기준을 두 개 주어야 합니다. (예시 코드)

```sql
select *
from tbl_category
order by
    ref_category_code is null asc,   -- 1순위: null 은 뒤로
    ref_category_code asc;           -- 2순위: 값 오름차순
```

### 5. DISTINCT: 열 하나가 아니라 "선택한 열의 조합"

```sql
select distinct category_code
from tbl_menu
order by category_code;
```

`distinct`는 첫 번째 열 하나에만 걸리는 것이 아니라 **`select`한 열 전체의 조합**이 같은 행을 하나로 합칩니다. 수업 예제는 열이 하나라서 구분이 안 되지만, 아래처럼 열이 두 개면 (카테고리, 판매상태) 조합 단위로 중복이 제거됩니다. (예시 코드)

```sql
select distinct category_code, orderable_status
from tbl_menu;
```

### 6. LIMIT: 오프셋은 "건너뛸 행 수"

```sql
select *
from tbl_menu
order by menu_price desc
limit 1, 5;
```

`limit [offset,] row_count`에서 offset은 **건너뛸 행의 수**입니다. 따라서 `limit 1, 5`는 가격이 가장 높은 1행을 건너뛰고, 2번째 행부터 5개(2~6위)를 가져옵니다. 수업 주석의 "시작할 행의 번호(인덱스 번호)"는 0부터 세는 번호라는 뜻입니다. `order by` 없이 `limit`만 쓰면 어떤 행이 나올지 보장되지 않으므로, `limit`는 항상 `order by`와 함께 씁니다.

같은 의미를 `limit 5 offset 1`로도 쓸 수 있고, 페이징에서는 아래 식이 됩니다.

| 페이지 | 한 페이지 10개일 때 | 쿼리 |
|--------|--------------------|------|
| 1 | offset 0 | `limit 0, 10` |
| 2 | offset 10 | `limit 10, 10` |
| n | `(n-1) * 10` | `limit (n-1)*10, 10` |

## 결과 (Result)

이번 정리는 성능 개선이 아니라 개념 정리라서 정량 지표는 없고, 아래 내용을 얻었습니다.

- 쿼리의 처리 순서(`FROM → WHERE → … → SELECT → ORDER BY → LIMIT`)를 알면 별칭 사용 가능 범위가 설명됩니다.
- `NULL`은 오름차순에서 맨 앞에 오고, `is null`로 정렬하면 뒤로 보낼 수 있습니다.
- `distinct`는 선택한 열 조합 단위이고, `limit`의 첫 숫자는 건너뛸 개수입니다.

## 더 학습하면 좋은 개념

- **SQL 논리적 쿼리 처리 순서** — 이 글 1번의 흐름도를 `JOIN`, `GROUP BY`, 서브쿼리까지 확장해 보면 "왜 여기선 별칭이 안 되지?"라는 에러를 스스로 설명할 수 있습니다.
- **오프셋 페이징 vs 커서(Keyset) 페이징** — `limit 100000, 10`은 앞의 10만 행을 읽고 버립니다. 데이터가 많아질 때 느려지는 이유와 대안을 알아 두면 좋습니다.
- **`EXPLAIN`과 인덱스** — `order by`와 `limit`이 인덱스를 타는지 확인하는 도구입니다.
- **`NULL`의 3값 논리(TRUE/FALSE/UNKNOWN)** — `null` 정렬과 비교가 왜 특이한지의 뿌리입니다. 다음 글(WHERE 편)과 이어집니다.
- **`COLLATE`(정렬 규칙)** — 한글·대소문자 정렬 순서는 문자셋과 collation이 결정합니다.

## 참고 자료

- [MySQL 8.4 - SELECT Statement (LIMIT 포함)](https://dev.mysql.com/doc/refman/8.4/en/select.html)
- [MySQL 8.4 - ORDER BY Optimization](https://dev.mysql.com/doc/refman/8.4/en/order-by-optimization.html)
- [MySQL 8.4 - Problems with NULL Values](https://dev.mysql.com/doc/refman/8.4/en/problems-with-null.html)
- [MySQL 8.4 - DISTINCT Optimization](https://dev.mysql.com/doc/refman/8.4/en/distinct-optimization.html)
- [MySQL 8.4 - Problems with Column Aliases](https://dev.mysql.com/doc/refman/8.4/en/problems-with-alias.html)
