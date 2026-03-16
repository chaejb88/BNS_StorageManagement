# BNS_StorageManagement

* Framework : Spring Boot
* Language : JAVA
* Database : Postgresql, TimescaleDB
* Target DB : ship_data_hub
* Infrastructure : Docker
* Date of original creation : 2025.07.10

# Infrastructure & Database Optimization
1. 스토리지 독립 및 파티셔닝 (Infrastructure)
데이터 안전성과 I/O 병목 현상 방지를 위해 물리 디스크를 분할하여 관리합니다.
 * 디스크 추가 및 파티셔닝: fdisk를 이용해 신규 디스크(sdb)를 3개 영역으로 분할
 * 마운트 경로:
   * /data/management: 관리 데이터용
   * /data/audit-log: 로그 기록용
   * /data/device: 실시간 장치 데이터 및 하이퍼테이블용 (Main DB)
 * 권한 최적화: Docker 컨테이너의 PostgreSQL 사용자(UID 999)와 소유권 일치
   sudo chown -R 999:999 /data/device/pgdata

2. PostgreSQL & TimescaleDB 성능 최적화 (Optimization)
대량의 데이터 삭제(drop_chunks) 및 빈번한 정렬 작업이 발생하는 운영 환경에 맞춰 컨테이너 내부 설정을 직접 최적화했습니다.

🛠 설정 변경 절차

① 컨테이너 접속 및 설정 파일 편집

*컨테이너 내부 진입
docker exec -it bns-maindb-server(예시) bash

*설정 파일 위치 확인 및 편집 (vi 사용)
vi /var/lib/postgresql/data/postgresql.conf

② 주요 설정값 수정 (postgresql.conf)
운영 환경에 맞춰 아래 항목들을 반영했습니다.
| 카테고리 | 설정 항목 | 수정값 | 목적 |
| :--- | :--- | :--- | :--- |
| WAL 관리 | max_wal_size | 2GB | 대량 삭제/변경 시 WAL 폭증 방지 |
| | min_wal_size | 80MB | Checkpoint 안정화 |
| AutoVacuum | autovacuum | on | Dead Tuple 자동 정리 활성화 |
| | autovacuum_naptime| 10s | Vacuum 실행 주기 단축 (반응성 향상) |
| 리소스 | shared_buffers | 512MB | DB 서버 가용 RAM의 약 25% 할당 (캐시 성능) |
| | work_mem | 32MB | 복잡한 정렬 및 집계 쿼리 성능 개선 |
| | maintenance_work_mem| 256MB | Index 생성 및 Vacuum 작업 속도 향상 |
| 기타 | checkpoint_timeout | 15min | 불필요한 I/O 부하 감소 |
| | maintenance_work_mem| 256MB | Index 생성 및 Vacuum 작업 속도 향상 |
| 기타 | checkpoint_timeout | 15min | 불필요한 I/O 부하 감소 |

Ubuntu 설정 방법
# 1) 컨테이너 내부로 진입

```bash
docker exec -it bns-maindb-server bash
```

---

# 2) PostgreSQL 설정 파일 위치 확인

TimescaleDB 공식 이미지 기준:

```
/var/lib/postgresql/data/postgresql.conf
```

또는

```
/var/lib/postgresql/data/pgdata/postgresql.conf
```

---

# 3) postgresql.conf 열기

컨테이너 내부에서:

```bash
vi /var/lib/postgresql/data/postgresql.conf
```

---

# 4) 수정 설정들

---

## ✔ 4-1) WAL 크기 조정

### max_wal_size

```conf
max_wal_size = 2GB
```

### min_wal_size

```conf
min_wal_size = 80MB
```

이 설정은:

- TimescaleDB에서 대량 DELETE / drop_chunks 후 WAL 폭증 방지
- checkpoint 빈도 조절
- 디스크 사용량 안정화


---

## ✔ 4-2) autovacuum 관련 설정


```conf
autovacuum = on
autovacuum_max_workers = 5
autovacuum_naptime = 10s
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.05
```

이 설정은:

- fallback DELETE 후 dead tuple이 많이 생기므로
- autovacuum이 빠르게 처리하도록 조정한 것

---

## ✔ 4-3) checkpoint 관련 설정

```conf
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
```

이건:

- drop_chunks / DELETE 후 checkpoint가 너무 자주 발생하지 않도록
- 하지만 너무 늦게도 발생하지 않도록
- 안정적인 I/O 유지 목적

---

## ✔ 4-4) shared_buffers

```conf
shared_buffers = 512MB
```

TimescaleDB는 shared_buffers를 RAM의 25% 정도로 권장.  
jb 환경에서는 512MB로 설정했던 것.

---

## ✔ 4-5) work_mem

```conf
work_mem = 32MB
```

정렬/집계가 많은 hypertable 환경에서 기본값(4MB)은 너무 작아서 조정.

---

## ✔ 4-6) maintenance_work_mem

```conf
maintenance_work_mem = 256MB
```

VACUUM / ANALYZE / drop_chunks 후 정리 작업 속도 향상.

---

## ✔ 4-7) TimescaleDB background worker 활성화

이건 postgresql.conf가 아니라 docker-compose에서 설정했지만  
컨테이너 내부에서 확인했던 값:

```conf
shared_preload_libraries = 'timescaledb'
```

컨테이너 내부에서 확인:

```sql
SHOW shared_preload_libraries;
```

결과:

```
timescaledb
```

---

# 5) 설정 변경 후 PostgreSQL 재시작

컨테이너 내부에서 PostgreSQL만 재시작할 수 없기 때문에  
우분투에서 컨테이너를 재시작했다:

```bash
docker restart bns-maindb-server
```

또는 docker-compose 사용 시:

```bash
docker-compose down
docker-compose up -d
```


3. DB의 docker-compose.yml (예시)

services: <br/>
  bns-maindb-server:<br/>
    image: 'timescale/timescaledb:2.17.2-pg17'<br/>
    container_name: bns-maindb-server<br/>
    environment:<br/>
      - POSTGRES_PASSWORD=marisight123!@#<br/>
      - POSTGRES_USER=marisight<br/>
      - POSTGRES_DB=has_onb_001<br/>
    ports:<br/>
      - "5432:5432"<br/>
    volumes:<br/>
      - /data/device/pgdata:/var/lib/postgresql/data<br/>
      - ./backups/maindb:/backups<br/>
    extra_hosts:<br/>
      - host.docker.internal:host-gateway<br/>
    logging:<br/>
      driver: "syslog"<br/>
      options:
        tag: "docker.bns-maindb-server"
