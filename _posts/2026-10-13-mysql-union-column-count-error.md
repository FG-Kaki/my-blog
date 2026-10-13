---
layout: post
title: "UNION이 컬럼 개수 오류를 내는 이유 - 쿼리 결과 합치기"
date: 2026-10-13 00:40:00 +0900
categories: [MySQL]
tags: [mysql, sql, union, union-all]
---

## 들어가며 (Situation)

수업 `11_union.sql`은 "각 쿼리문의 결과를 합치는 Union, 수학의 합집합"으로 시작합니다. 시나리오는 이렇습니다. `users` 중 `active`와 `suspended` 사용자에게는 이메일을 보내고, 탈퇴한 `deleted` 유저는 제외합니다.

> 실제 서비스에서 겪은 문제가 아니라, 수업 실습을 정리한 글입니다. 동작 설명은 MySQL 공식 문서로 보완했습니다.

## 문제 상황 (Task)

사용자의 이름, 이메일, 상태를 조회하는데, 두 상태를 `or`로 묶지 않고 쿼리 두 개의 결과를 합치려 했습니다. 수업 코드는 그대로 실행하면 실패하는 형태였습니다.

## 해결 과정 (Action)

### 1. 수업 코드에서 발견한 문제

```sql
select `email`, `status` from users where status = 'active'
union
select `name`, `email`, `status` from users where status = 'suspended';
```

첫 번째 `select`는 컬럼이 2개, 두 번째는 3개입니다. 수업 주석이 말한 규칙 그대로입니다.

| 규칙 | 설명 |
|------|------|
| 컬럼 개수 동일 | 모든 `select`의 컬럼 수가 같아야 함 |
| 컬럼 순서 | 위치(순서)로 대응되므로 같은 의미끼리 맞춰야 함 |
| 데이터 타입 | 서로 호환 가능해야 함 |

컬럼 수가 다르면 MySQL은 오류("The used SELECT statements have a different number of columns")를 냅니다. 이 문구는 공식 문서에 있는 오류 메시지이고, 위 수업 코드를 직접 실행해 출력을 확인한 것은 아닙니다.

### 2. 수정

```sql
select `name`, `email`, `status` from users where status = 'active'
union
select `name`, `email`, `status` from users where status = 'suspended';
```

결과 컬럼 이름은 **첫 번째 `select`** 의 이름을 따릅니다. 그래서 별칭도 첫 쿼리에 붙입니다.

### 3. UNION vs UNION ALL

| 구분 | 중복 행 | 비용 |
|------|---------|------|
| `union` | 제거 (`distinct`와 같은 효과) | 중복 제거 때문에 상대적으로 비쌈 |
| `union all` | 유지 | 상대적으로 가벼움 |

수업은 "중복된 행은 제거하여 순수한 합집합"이라고 설명했습니다. 이 예제처럼 서로 다른 `status`를 조회하면 겹치는 행이 없으므로 `union all`로도 같은 결과이고 더 효율적입니다.

### 4. 더 단순한 방법

이 문제는 사실 한 쿼리로도 풀립니다. (예시 코드)

```sql
select name, email, status from users where status in ('active', 'suspended');
```

`union`은 서로 다른 **테이블**이나 다른 조건 구조의 결과를 합칠 때 더 빛납니다. 예를 들어 이전 JOIN 글에서 다룬 `FULL OUTER JOIN`을 `left join`과 `right join`의 `union`으로 흉내 내는 경우입니다.

## 결과 (Result)

정량 지표는 없고, 아래를 정리했습니다.

- `union`의 전제 조건은 컬럼 개수·순서·타입의 호환입니다.
- 결과 컬럼명은 첫 번째 `select`를 따릅니다.
- 중복이 없다고 확신하면 `union all`을 씁니다.
- 같은 테이블의 조건 차이라면 `in`이나 `or`가 더 간단할 수 있습니다.

## 더 학습하면 좋은 개념

- **`UNION ALL`과 중복 제거 비용** — 대용량 데이터에서 정렬·임시 테이블 비용을 이해하는 출발점입니다.
- **`INTERSECT` / `EXCEPT`** — 교집합·차집합 연산으로 MySQL 8.0.31부터 지원됩니다.
- **`FULL OUTER JOIN` 흉내 내기** — `left join union right join` 패턴으로 JOIN 글과 연결됩니다.
- **`ORDER BY`와 `LIMIT`의 적용 범위** — `union` 전체 결과에 정렬을 걸 때는 괄호 사용법을 알아야 합니다.

## 참고 자료

- [MySQL 8.4 - UNION Clause](https://dev.mysql.com/doc/refman/8.4/en/union.html)
- [MySQL 8.4 - INTERSECT Clause](https://dev.mysql.com/doc/refman/8.4/en/intersect.html)
- [MySQL 8.4 - JOIN Clause](https://dev.mysql.com/doc/refman/8.4/en/join.html)
