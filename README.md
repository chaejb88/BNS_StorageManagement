📦 BNS_StorageManagement
실시간 데이터 처리를 위한 스토리지 관리 및 데이터베이스 최적화 프로젝트입니다.
🛠 Tech Stack
| Category | Stack |
|---|---|
| Framework | Spring Boot |
| Language | Java |
| Database | PostgreSQL, TimescaleDB |
| Infrastructure | Docker, Ubuntu (Disk Partitioning) |
| Main DB | ship_data_hub |
| Created At | 2025.07.10 |
🏗 Infrastructure & Database Optimization
1. 스토리지 독립 및 파티셔닝 (Infrastructure)
데이터의 안전성을 보장하고 I/O 병목 현상을 방지하기 위해 물리 디스크를 논리적으로 분할하여 관리합니다.
 * 디스크 파티셔닝: fdisk를 활용하여 신규 디스크(sdb)를 3개의 영역으로 분할
 * 마운트 경로 설정:
   * /data/management: 시스템 관리 데이터 저장용
   * /data/audit-log: 감사 로그 기록 전용 공간
   * /data/device: (Main) 실시간 장치 데이터 및 하이퍼테이블 저장용
 * 권한 최적화: Docker 컨테이너 내 PostgreSQL 사용자(UID 999)와 소유권을 일치시켜 접근 권한 이슈 해결
   sudo chown -R 999:999 /data/device/pgdata

2. PostgreSQL & TimescaleDB 성능 최적화 (Optimization)
대량 데이터 삭제(drop_chunks) 및 빈번한 집계/정렬 작업이 발생하는 운영 환경에 맞춰 컨테이너 내부 설정을 최적화했습니다.
🛠 설정 변경 절차
 * 컨테이너 내부 진입
   docker exec -it bns-maindb-server bash

 * 설정 파일(postgresql.conf) 수정
   vi /var/lib/postgresql/data/postgresql.conf

 * 주요 설정값 반영
   | 카테고리 | 설정 항목 | 수정값 | 목적 |
   | :--- | :--- | :--- | :--- |
   | WAL 관리 | max_wal_size | 2GB | 대량 삭제/변경 시 WAL 폭증 방지 |
   | | min_wal_size | 80MB | Checkpoint 지점 안정화 |
   | AutoVacuum | autovacuum | on | Dead Tuple 자동 정리 활성화 |
   | | autovacuum_naptime | 10s | Vacuum 실행 주기 단축 (반응성 향상) |
   | 리소스 할당 | shared_buffers | 512MB | 캐시 성능 최적화 (RAM의 약 25%) |
   | | work_mem | 32MB | 복잡한 정렬 및 집계 쿼리 속도 개선 |
   | | maintenance_work_mem | 256MB | Index 생성 및 인프라 작업 속도 향상 |
   | 기타 | checkpoint_timeout | 15min | 불필요한 디스크 I/O 부하 감소 |
3. Docker Deployment (docker-compose.yml)
분리된 파티션을 컨테이너 내 볼륨으로 매핑하여 독립적인 데이터 저장 환경을 구축했습니다.
version: '3.8'
services:
  bns-maindb-server:
    image: 'timescale/timescaledb:2.17.2-pg17'
    container_name: bns-maindb-server
    environment:
      - POSTGRES_PASSWORD=marisight123!@#  # 실제 운영 시 환경 변수 분리 권장
      - POSTGRES_USER=marisight
      - POSTGRES_DB=has_onb_001
    ports:
      - "5432:5432"
    volumes:
      - /data/device/pgdata:/var/lib/postgresql/data # 파티셔닝된 경로 매핑
      - ./backups/maindb:/backups
    extra_hosts:
      - host.docker.internal:host-gateway
    logging:
      driver: "syslog"
      options:
        tag: "docker.bns-maindb-server"
