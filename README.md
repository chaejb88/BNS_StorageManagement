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

3. DB의 docker-compose.yml (예시)
services:
  bns-maindb-server:
    image: 'timescale/timescaledb:2.17.2-pg17'
    container_name: bns-maindb-server
    environment:
      - POSTGRES_PASSWORD=marisight123!@#
      - POSTGRES_USER=marisight
      - POSTGRES_DB=has_onb_001
    ports:
      - "5432:5432"
    volumes:
      - /data/device/pgdata:/var/lib/postgresql/data
      - ./backups/maindb:/backups
    extra_hosts:
      - host.docker.internal:host-gateway
    logging:
      driver: "syslog"
      options:
        tag: "docker.bns-maindb-server"
| | maintenance_work_mem| 256MB | Index 생성 및 Vacuum 작업 속도 향상 |
| 기타 | checkpoint_timeout | 15min | 불필요한 I/O 부하 감소 |
