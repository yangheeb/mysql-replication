# MySQL Replication 장애 조치(Failover) 및 복구 실습

본 문서는 Docker 환경에서 구성한 Source–Replica 구조에서 장애 발생 시 수동 승격(Promote)과 재동기화 과정을 검증한 실습 기록입니다.
비동기(File/Position 기반) 복제를 기준으로 진행합니다.

---

# 1. 사전 확인 (Source / Replica 공통)

먼저 두 서버 모두 동일한 데이터 상태에서 시작하는지 확인합니다.

## 1-1. 데이터베이스 목록 확인

```sql
SHOW DATABASES;
```

## 1-2. 테스트 데이터베이스 선택

```sql
USE testdb;
```

## 1-3. 기존 데이터 확인

```sql
SELECT * FROM products;
```

Replica 서버에서도 동일한 결과가 조회되어야 정상적인 복제 상태입니다.

---

# 2. 정상 동기화 상태 검증

## 2-1. Source에서 데이터 입력

```sql
INSERT INTO products VALUES ('test_data');
```

## 2-2. Replica에서 데이터 확인

```sql
SELECT * FROM products;
```

Source에서 입력한 데이터가 Replica에서도 동일하게 조회되면 복제가 정상 동작 중임을 의미합니다.

---

# 3. Source 장애 발생

운영 중 장애 상황을 가정하고 Source 컨테이너를 강제 종료합니다.

```bash
docker stop mysql-source
```

---

# 4. 장애 발생 후 Replica 상태 확인

Replica 서버에서 복제 상태를 확인합니다.

```sql
SHOW REPLICA STATUS\G;
```

확인해야 할 항목:

```
Replica_IO_Running: Connecting 또는 No
Replica_SQL_Running: Yes 또는 No
```

Source와의 연결이 끊어졌기 때문에 IO Thread가 정상 동작하지 않는 상태임을 확인할 수 있습니다.

---

# 5. Replica 승격 (Promote)

이제 Replica를 독립적인 Source 서버로 전환합니다.

## 5-1. 복제 중지

```sql
STOP REPLICA;
```

## 5-2. 기존 Source 연결 정보 제거

```sql
RESET REPLICA ALL;
```

이 명령으로 기존 Source와의 종속 관계가 완전히 제거됩니다.

---

# 6. 승격된 서버에서 쓰기 검증

이제 해당 서버는 독립적인 Source 역할을 수행합니다.

```sql
INSERT INTO products VALUES ('i_am_new_master');
SELECT * FROM products;
```

정상적으로 INSERT가 수행되면 승격이 완료된 것입니다.

---

# 7. 기존 Source 복구 및 데이터 정합성 맞추기

이제 중단되었던 기존 Source를 다시 살리고, 승격된 서버와 동일한 데이터 상태로 맞춰야 합니다.

## 7-1. Source 컨테이너 재시작

```bash
docker start mysql-source
```

## 7-2. 컨테이너 접속

```bash
docker exec -it mysql-source /bin/bash
```

## 7-3. MySQL 접속

```bash
mysql -u root -p
```

## 7-4. 데이터베이스 선택

```sql
USE testdb;
```

---

## 7-5. 테이블 생성 (존재 여부 안전 처리)

```sql
CREATE TABLE IF NOT EXISTS products (
  text VARCHAR(40)
);
```

테이블이 이미 존재하는 경우에도 에러 없이 넘어가도록 처리합니다.

---

## 7-6. 승격된 서버와 동일한 데이터 입력

```sql
INSERT INTO products VALUES
('test_data'),
('i_am_new_master');
```

기존 Replica가 보유하고 있던 데이터와 동일하게 맞춰줍니다.

---

## 7-7. 현재 binlog 위치 확인

```sql
SHOW MASTER STATUS;
```

예시 결과:

```
File: mysql-bin.000004
Position: 669
```

이 File과 Position 값을 복제 재설정 시 사용합니다.

---

# 8. Replica를 다시 종속 서버로 설정

이제 승격했던 서버를 다시 Replica로 구성합니다.

## 8-1. 복제 중지

```sql
STOP REPLICA;
```

## 8-2. 복제 정보 재설정

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-source',
  SOURCE_USER='gugu',
  SOURCE_PASSWORD='1234',
  SOURCE_LOG_FILE='mysql-bin.000004',
  SOURCE_LOG_POS=669;
```

## 8-3. 복제 재시작

```sql
START REPLICA;
```

---

# 9. 복구 완료 상태 확인

```sql
SHOW REPLICA STATUS\G;
```

정상 상태 확인 항목:

```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
Last_Errno: 0
Last_SQL_Errno: 0
```

IO와 SQL 스레드가 모두 정상 동작하고, 지연 시간이 0이면 복구가 완료된 상태입니다.

---

# 10. 전체 흐름 요약

1. 정상 복제 상태 확인
2. Source 강제 종료
3. Replica 상태 점검
4. STOP REPLICA
5. RESET REPLICA ALL
6. Replica 승격 및 쓰기 검증
7. Source 재기동
8. 데이터 정합성 수동 동기화
9. SHOW MASTER STATUS로 binlog 위치 확인
10. CHANGE REPLICATION SOURCE TO 재설정
11. START REPLICA
12. SHOW REPLICA STATUS로 정상화 검증
