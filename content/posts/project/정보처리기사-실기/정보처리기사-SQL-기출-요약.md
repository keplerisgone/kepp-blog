---
title: "정보처리기사 SQL 기출 요약"
date: "2026-07-13T17:31:15+09:00"
lastmod: "2026-07-13T17:31:15+09:00"
draft: false
description: ""
categories: ["project", "정보처리기사-실기"]
tags: []
ShowToc: true
---

## 결과 추론

### `COUNT`, `DISTINCT`, `ON DELETE CASCADE`

- **유형:** 집계 함수, 중복 제거, 외래키의 연쇄 삭제 결과 추론
- **규칙:** `COUNT(컬럼명)`은 `NULL`이 아닌 값의 개수를 세고, `ON DELETE CASCADE`는 부모 행 삭제 시 자식 행도 함께 삭제한다.
- **함정:** `SELECT DISTINCT COUNT(컬럼)`은 `COUNT(DISTINCT 컬럼)`과 다르다.
- **예시:** 부서 코드가 20인 직원 3명 → `COUNT` 결과 3 → 부서 코드 20 삭제 후 직원 3명도 삭제 → 남은 직원 4명

### `SELECT DISTINCT`와 `COUNT(DISTINCT)`

- **유형:** 조회 결과의 튜플 수, 중복 제거 결과 추론
- **규칙:** `DISTINCT`는 중복값을 제거하고, `COUNT(DISTINCT DEPT)`는 조건에 맞는 행에서 서로 다른 학과 값의 개수를 센다.
- **함정:** `SELECT DEPT`는 전체 학생 수만큼 200행이 나오지만, `SELECT DISTINCT DEPT`는 학과 종류인 전기과·전산과·전자과 3행만 나온다.
- **예시:** 전기과 50명 + 전산과 100명 + 전자과 50명 → `SELECT DEPT`는 200행, `SELECT DISTINCT DEPT`는 3행, `WHERE DEPT = '전산과'`에서 `COUNT(DISTINCT DEPT)`는 1

### 다중 행 서브쿼리와 `ALL`

- **유형:** 다중 행 서브쿼리, `ALL` 비교 연산자
- **규칙:** `> ALL(서브쿼리)`은 서브쿼리 결과의 모든 값보다 커야 한다는 뜻이다.
- **함정:** `MAX(단가)`와 논리 결과는 비슷할 수 있지만, 빈칸이 집계 함수 자리가 아니라면 `ALL`, `ANY`, `SOME`과 같은 비교 한정자를 써야 한다.
- **예시:** `단가 > ALL(1000, 2000, 3000)`은 `단가 > 3000`과 같은 의미다.

### `COUNT(컬럼)`과 `NULL`

- **유형:** `COUNT(컬럼)`, `IN`, `OR`, `NULL` 제외 여부
- **규칙:** `COUNT(COL2)`는 조건을 만족하는 행 중 `COL2`가 `NULL`이 아닌 행만 센다.
- **함정:** `WHERE` 조건에 맞는 행이 1·3·5번이어도 1번의 `COL2`가 `NULL`이면 집계에서 제외된다.
- **예시:** `COL1 IN (2, 3)` 또는 `COL2 IN (3, 5)`를 만족하는 행이 1번(`NULL`), 3번(5), 5번(3)이라면 `COUNT(COL2)`는 2다.

## 관계 대수

### 프로젝트(Projection)

- **유형:** 프로젝트 연산 결과 추론
- **규칙:** `π속성명(릴레이션)`은 지정한 속성만 추출하므로 `πTTL(EMPLOYEE)`는 `TTL` 열만 출력한다.
- **함정:** 프로젝트는 행에 조건을 거는 연산이 아니라 열을 고르는 연산이므로 `AGE`, `INDEX`는 결과에서 제외된다.
- **예시:** `EMPLOYEE`에서 `TTL`만 추출 → `TTL`, 부장, 대리, 과장, 차장

## 데이터 조작 및 정의

### `UPDATE`

- **유형:** `UPDATE`의 주요 구성 요소
- **규칙:** `UPDATE`는 갱신 대상 테이블, `SET`은 변경할 열과 값, `WHERE`는 변경할 행의 조건을 지정한다.
- **함정:** `WHERE`가 없으면 모든 행이 변경된다. `SET`에는 여러 열을 지정할 수 있고, 조건에는 비교 연산자, `AND`, `OR`, `IN`, `BETWEEN` 등을 사용할 수 있다.
- **예시:** `UPDATE 학생 SET 학과 = '컴퓨터공학과', 학년 = 3 WHERE 학번 = 1001;`

### `DELETE`

- **유형:** `DELETE`의 기본 형식, 조건부 삭제, 문자열 조건
- **규칙:** `DELETE FROM 테이블명 WHERE 조건;` 형식으로 쓰며 문자열 값은 작은따옴표로 감싼다.
- **함정:** `DELETE` 뒤에는 열 이름이 아닌 `FROM 테이블명`을 쓴다. `WHERE`를 생략하면 테이블의 모든 행이 삭제된다.
- **예시:** `DELETE FROM 학생 WHERE 이름 = '민수';`

### `ALTER TABLE`

- **유형:** 열 추가, 테이블 구조 변경
- **규칙:** `ALTER TABLE 테이블명 ADD 컬럼명 데이터타입;` 형식으로 기존 테이블에 새 속성을 추가한다.
- **함정:** 데이터를 수정하는 `UPDATE`가 아니라 테이블 구조를 바꾸는 DDL이므로 `ALTER`를 써야 한다.
- **예시:** `ALTER TABLE 학생 ADD 주소 VARCHAR(20);`

### DDL

- **유형:** 테이블 구조의 정의·변경·삭제
- **규칙:** DDL은 데이터 자체가 아니라 데이터베이스 객체의 구조를 정의하며 대표적으로 `CREATE`, `ALTER`, `DROP`, `TRUNCATE`가 있다.
- **함정:** `INSERT`, `UPDATE`, `DELETE`는 데이터를 조작하는 DML이고, `CREATE`, `ALTER`, `DROP`은 구조를 다루는 DDL이다.
- **예시:** `CREATE TABLE 학생(학번 CHAR(10));` / `ALTER TABLE 학생 ADD 주소 VARCHAR(20);` / `DROP TABLE 학생;` / `TRUNCATE TABLE 학생;`

## 조회와 정렬

### `WHERE IN`

- **유형:** 열 조회, `WHERE` 조건, `IN` 연산자
- **규칙:** `IN (값1, 값2)`는 열 값이 괄호 안의 값 중 하나와 일치하는 행을 조회한다.
- **함정:** 학년이 숫자 자료형이면 3과 4에 작은따옴표를 붙이지 않는다. 조회할 열은 문제에서 요구한 학번과 이름만 쓴다.
- **예시:** `SELECT 학번, 이름 FROM 학생 WHERE 학년 IN (3, 4);`

### `ORDER BY`

- **유형:** 특정 열을 기준으로 한 내림차순 정렬
- **규칙:** `ORDER BY 컬럼명 ASC|DESC` 형식으로 정렬한다. `ASC`는 오름차순, `DESC`는 내림차순이다.
- **함정:** 조회할 열과 정렬 기준 열을 구분해야 한다. 점수 기준 내림차순이라면 `ORDER BY score DESC`를 사용한다.
- **예시:** `SELECT name, score FROM 성적 ORDER BY score DESC;`

### `LIKE`와 `ORDER BY`

- **유형:** 문자열 패턴 검색, 내림차순 정렬
- **규칙:** `LIKE '이%'`는 이름이 ‘이’로 시작하는 값을 찾고, `ORDER BY 가입일 DESC`는 가입일을 기준으로 내림차순 정렬한다.
- **함정:** `GROUP BY`는 데이터를 묶는 절이다. 특정 글자로 시작하는 문자열 조건에는 `LIKE`를 써야 한다.
- **예시:** `SELECT * FROM 회원 WHERE 이름 LIKE '이%' ORDER BY 가입일 DESC;`

## 그룹과 집계

### `GROUP BY`와 `COUNT`

- **유형:** 그룹화, 집계 함수, 별칭
- **규칙:** `GROUP BY 학과`는 학과별로 행을 묶고, `COUNT(*)`는 각 학과의 튜플 수를 세며, `AS`는 출력 열 이름을 지정한다.
- **함정:** `AS 학과별튜플수`는 계산 조건이 아니라 `COUNT(*)` 결과 열의 별칭이다.
- **예시:** `SELECT 학과, COUNT(*) AS 학과별튜플수 FROM 학생 GROUP BY 학과;`

### `GROUP BY`와 `HAVING`

- **유형:** 그룹화, 그룹 조건, `MIN`, `MAX`, `AVG`, 별칭
- **규칙:** `GROUP BY 과목이름`으로 과목별 그룹을 만들고, `HAVING AVG(점수) >= 90`으로 평균이 90점 이상인 과목만 남긴다.
- **함정:** `WHERE`는 그룹화 전 개별 행의 조건이고, `HAVING`은 그룹화 후 집계 결과의 조건이다. 평균 점수 조건에는 `HAVING`을 써야 한다.
- **예시:** `SELECT 과목이름, MIN(점수) AS 최소점수, MAX(점수) AS 최대점수 FROM 성적 GROUP BY 과목이름 HAVING AVG(점수) >= 90;`

## 인덱스

### `CREATE INDEX`

- **유형:** 인덱스 생성, 검색 성능 향상
- **규칙:** `CREATE INDEX 인덱스명 ON 테이블명(컬럼명);` 형식으로 특정 열을 기준으로 한 검색용 자료 구조를 생성한다.
- **함정:** 인덱스를 생성해도 테이블의 행, 값, 출력 결과는 바뀌지 않는다. 검색을 빠르게 하기 위한 보조 구조만 추가된다.
- **예시:** `CREATE INDEX idx_name ON student(name);` 실행 후 `student` 테이블은 그대로이고, `name = 'brown'`과 같은 조건 검색에서 `idx_name`을 이용할 수 있다.

### 주요 인덱스 명령

- **일반 인덱스:** 특정 열을 기준으로 검색 성능을 높인다. 예: `CREATE INDEX idx_name ON student(name);`
- **고유 인덱스:** 중복값을 허용하지 않는다. 예: `CREATE UNIQUE INDEX idx_stid ON student(stid);`
- **인덱스 삭제:** 생성된 인덱스를 삭제한다. 예: `DROP INDEX idx_name;`
- **복합 인덱스:** 두 개 이상의 열을 묶어 생성한다. 예: `CREATE INDEX idx_name_score ON student(name, score);`

## 조인

### `CROSS JOIN` 결과 추론

- **유형:** 카티션 곱, `LIKE` 패턴 조건, 조인 후 `COUNT(*)` 결과 추론
- **규칙:** `CROSS JOIN`은 두 테이블의 모든 행 조합을 만든 뒤 `WHERE` 조건으로 필요한 조합만 필터링한다.
- **함정:** A 3행 × B 2행 = 6개 조합을 먼저 만들고, `A.NAME LIKE B.RULE`을 만족하는 조합만 센다.
- **예시:** Smith는 `S%`에 매칭, Scott은 `S%`에 매칭, Smith와 Scott은 `%T%`에도 각각 매칭 → 총 4개

### 조인 종류

- **CROSS JOIN:** 조건 없이 두 테이블의 모든 조합을 만든다. 예: `SELECT * FROM A CROSS JOIN B;`
- **INNER JOIN:** 조인 조건을 만족하는 행만 출력한다. 예: `SELECT * FROM A INNER JOIN B ON A.id = B.id;`
- **LEFT OUTER JOIN:** 왼쪽 테이블은 모두 남기고 오른쪽 테이블은 조건에 맞는 행만 붙인다. 예: `SELECT * FROM A LEFT JOIN B ON A.id = B.id;`
- **RIGHT OUTER JOIN:** 오른쪽 테이블은 모두 남기고 왼쪽 테이블은 조건에 맞는 행만 붙인다. 예: `SELECT * FROM A RIGHT JOIN B ON A.id = B.id;`
- **FULL OUTER JOIN:** 양쪽 테이블의 모든 행을 남기고, 조건에 맞지 않는 부분은 `NULL`로 채운다. 예: `SELECT * FROM A FULL OUTER JOIN B ON A.id = B.id;`
- **SELF JOIN:** 같은 테이블을 자기 자신과 조인한다. 예: `SELECT * FROM 직원 A JOIN 직원 B ON A.상사ID = B.직원ID;`
- **NATURAL JOIN:** 두 테이블에서 이름이 같은 열을 기준으로 자동 조인한다. 예: `SELECT * FROM A NATURAL JOIN B;`
