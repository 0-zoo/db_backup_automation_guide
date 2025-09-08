# DB Backup Automation


> 본 프로젝트는 리눅스 환경에서 MySQL을 직접 설치하고(emp/dept 샘플 테이블 기반), crontab과 shell script를 이용하여 DB 백업을 자동화하는 실습 프로젝트입니다.
> 
> 
> 또한 MySQL을 포함한 주요 DB 벤더사별 백업 기술 문서를 조사하여, DevOps 관점에서 데이터베이스 백업 전략을 체계적으로 정리하였습니다.
>
> 
---

# 👥 팀원 소개

| 황지환 | 이영주 |
|--------|--------|
| [![황지환](https://github.com/jihwan77.png?size=100)](https://github.com/jihwan77) | [![이영주](https://github.com/0-zoo.png?size=100)](https://github.com/0-zoo) |

> 👉 각 이미지 클릭 시 해당 팀원의 GitHub 프로필로 이동합니다.


---

## 📑 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [실습 환경](#2-실습-환경)
3. [MySQL 설치 및 서비스 등록](#3-mysql-설치-및-서비스-등록)
4. [샘플 스키마 (EMP & DEPT 테이블)](#4-샘플-스키마-emp--dept-테이블)
5. [DB 백업 자동화](#5-db-백업-자동화)
   - 5.1 [mysqldump 기본 백업](#51-mysqldump-기본-백업)
   - 5.2 [crontab 자동화 설정](#52-crontab-자동화-설정)
   - 5.3 [Shell Script & Tar 압축](#53-shell-script--tar-압축)
   - 5.4 [파일명 규칙 및 보관 정책](#54-파일명-규칙-및-보관-정책)
6. [벤더사별 DB 백업 기술 조사](#6-벤더사별-db-백업-기술-조사)
7. [백업 주기 전략](#7-백업-주기-전략)
8. [DevOps 관점에서의 의의](#8-devops-관점에서의-의의)
9. [향후 개선 방향](#9-향후-개선-방향)
10. [참고 문헌 및 공식 문서](#10-참고-문헌-및-공식-문서)
---

## 1. 프로젝트 개요

- **목적**: 데이터베이스의 안정적인 운영을 위해 필수적인 **백업/복구 전략** 설계
- **범위**:
    1. MySQL 직접 설치 및 서비스 관리
    2. 샘플 데이터(EMP, DEPT 테이블) 생성
    3. `mysqldump` 기반의 논리적 백업 + `cron` 자동화
    4. Shell script를 활용한 압축 및 보관
    5. 벤더사별 DBMS 백업 기술 문헌 비교

---

## 2. 실습 환경

- OS: Ubuntu 22.04 LTS
- DBMS: MySQL 8.0 (네이티브 설치)
- Language: SQL, Shell Script
- Tools: `mysqldump`, `tar`, `cron`

---

## 3. MySQL 설치 및 서비스 등록

```bash
sudo apt update
sudo apt install mysql-server -y

# 서비스 등록 확인
sudo systemctl status mysql
sudo systemctl enable mysql
```

---

## 4. 샘플 스키마 (EMP & DEPT 테이블)

오라클 전통 샘플 스키마를 기반으로 `dept`, `emp` 테이블 생성.

```sql
CREATE DATABASE fisa;
USE fisa;

CREATE TABLE dept (
    deptno INT PRIMARY KEY,
    dname VARCHAR(14),
    loc VARCHAR(13)
);

CREATE TABLE emp (
    empno INT PRIMARY KEY,
    ename VARCHAR(10),
    job VARCHAR(9),
    mgr INT,
    hiredate DATE,
    sal DECIMAL(7,2),
    comm DECIMAL(7,2),
    deptno INT,
    FOREIGN KEY (deptno) REFERENCES dept(deptno)
);
```

---

## 5. DB 백업 자동화

### 5.1 mysqldump 기본 백업

```bash
mysqldump -u root -pPASSWORD fisa > /root/db.sql
```

### 5.2 crontab 자동화 설정

매일 새벽 2시마다 백업 실행 및 7일 단위 보관 정책 적용:

```bash
0 2 * * * mysqldump -u root -pPASSWORD fisa > /root/db.sql

0 3 * * * find /root/backup -name "*.tar" -type f -mtime +7 -exec rm -f {} \;
```

### 5.3 Shell Script & Tar 압축

`/root/db_backup/backup.sh`:

```bash
#!/bin/bash

# DB 접속 정보
USER="root"
PASSWORD="fisa"
DATABASE="fisa"

# 백업 저장 경로
BACKUP_DIR="/root/db_backup"

# 파일명: DB명_년월일시분초.sql
SQL_FILE="$BACKUP_DIR/${DATABASE}_$(date +%Y%m%d_%H%M%S).sql"
TAR_FILE="${SQL_FILE}.tar.gz"

# 덤프 수행
mysqldump -u $USER -p$PASSWORD $DATABASE > "$SQL_FILE"

# 압축
tar -czf "$TAR_FILE" "$SQL_FILE"

# 원본 SQL은 삭제 (압축만 보관)
rm "$SQL_FILE"

# 로그
echo "[$(date)] $DATABASE 백업 완료: $TAR_FILE"
```

실행 권한:

```bash
chmod +x /root/db_backup/backup.sh
```

### 5.4 파일명 규칙 및 보관 정책

- 파일명 형식: `db_<DBNAME>_YYYYMMDD_HHMMSS.tar.gz`
- 보관 정책:
    - 최근 7일치 일 단위 보관
    

---

## 6. 벤더사별 DB 백업 기술 조사

### 🔹 MySQL / MariaDB

- **주요 백업 도구**: `mysqldump`, `mysqlpump`, Percona XtraBackup
- **특징**
    - `mysqldump`: 논리적 백업, 이식성 높음
    - `mysqlpump`: 병렬 지원으로 더 빠른 속도
    - `XtraBackup`: 대규모 운영 환경에 적합, 서비스 중단 없이 백업 가능
- **권장 백업 주기**
    - Full dump: 하루 1회 (새벽)
    - Binary Log 기반 증분: 5~15분 단위
    - 소규모/실습 환경: 하루 1회 full dump만으로 충분
- **파일명 형식**: `mysql_<DBNAME>_YYYYMMDD_HHMMSS.sql`
- **공식 문서**: [MySQL Backup Docs](https://dev.mysql.com/doc/refman/8.0/en/backup-methods.html)

---

### 🔹 PostgreSQL

- **주요 백업 도구**: `pg_dump`, `pg_dumpall`, PITR (Point-in-Time Recovery)
- **특징**
    - `pg_dump`: 데이터베이스 단위 백업
    - `pg_dumpall`: 클러스터 전체 백업
    - WAL 로그 아카이빙을 통해 특정 시점으로 복구 가능
- **권장 백업 주기**
    - Full dump: 하루 1회
    - WAL 로그 아카이빙: 5~10분 단위
    - 학습/테스트 환경: 하루 1회 full dump
- **파일명 형식**: `pg_<DBNAME>_YYYYMMDD_HHMMSS.sql`
- **공식 문서**: [PostgreSQL Backup Docs](https://www.postgresql.org/docs/current/backup.html)

---

### 🔹 Oracle

- **주요 백업 도구**: RMAN, Data Pump (`expdp` / `impdp`)
- **특징**
    - RMAN: 블록 단위 물리적 백업, 증분/병렬 지원
    - Data Pump: 테이블/스키마 단위 논리적 백업
- **권장 백업 주기**
    - Full backup (RMAN): 주 1회
    - Incremental backup: 매일 또는 6~12시간 단위
    - Archive log: 실시간 아카이빙
- **파일명 형식**
    - `oracle_rman_FULL_YYYYMMDD.bak`
    - `oracle_dp_<SCHEMA>_YYYYMMDD.dmp`
- **공식 문서**: [Oracle RMAN Docs](https://docs.oracle.com/en/database/oracle/oracle-database/)

---

### 🔹 MS SQL Server

- **주요 백업 도구**: SSMS, `BACKUP DATABASE`
- **특징**
    - Full, Differential, Transaction Log 조합 가능
    - GUI와 스크립트 모두 지원
- **권장 백업 주기**
    - Full backup: 주 1회
    - Differential backup: 매일 1회
    - Transaction Log backup: 15~30분 단위
- **파일명 형식**
    - `mssql_<DBNAME>_FULL_YYYYMMDD.bak`
    - `mssql_<DBNAME>_LOG_YYYYMMDD.trn`
- **공식 문서**: [MS SQL Backup Docs](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/)

---

### 🔹 MongoDB (NoSQL)

- **주요 백업 도구**: `mongodump`, `mongorestore`, Oplog 기반 증분
- **특징**
    - `mongodump`: BSON 기반, 개발/테스트 환경에 적합
    - Oplog Tail: 증분 백업 가능
    - Atlas(클라우드): 자동 스냅샷 지원
- **권장 백업 주기**
    - Full dump: 하루 1회
    - Oplog 기반 증분: 수 분 ~ 1시간 단위
    - 소규모 테스트 환경: 하루 1회 full dump
- **파일명 형식**: `mongo_<DBNAME>_YYYYMMDD.archive`
- **공식 문서**: [MongoDB Backup Docs](https://www.mongodb.com/docs/manual/core/backups/)


---

## 7. 백업 주기 전략

데이터베이스 백업 주기는 **데이터 중요도, 변경 빈도, 서비스 가용성 요구사항**에 따라 달라진다.

일반적으로는 **Full Backup + Differential/Incremental Backup + Log Backup**을 조합해 데이터 손실 가능성을 최소화하지만, 본 프로젝트에서는 서비스 특성과 환경적 제약을 고려해 **Full Backup 중심의 전략**을 채택하였다.

---

### 🔹 Full Backup (전체 백업)

- 주기: **하루 1회 (새벽 2시)**
- 전체 데이터베이스를 백업하여 단일 파일로 관리
- 장점: 복원 절차가 단순하고 안정성이 높음
- 단점: 백업 시간과 스토리지 사용량이 크지만, 상대적으로 데이터 규모가 크지 않다는 점에서 허용 가능한 수준

👉 **선택 이유:** 데이터 무결성을 보장하면서 복구 절차를 단순화할 수 있기 때문에, 유지·운영 측면에서 효율적이다.

---


### 🔹 Differential / Incremental Backup (차등/증분 백업)

- Differential: Full Backup 이후 변경된 모든 데이터를 백업 (매일 1회)
- Incremental: 직전 백업 이후 변경분만 백업 (6~12시간 단위)
- 장점: 백업 속도와 저장 공간을 절약할 수 있음
- 단점: 복구 시 여러 파일을 조합해야 하므로 절차가 복잡해짐

👉 **정책상 제외한 이유:**

데이터 변경 빈도가 높지 않고, 시스템 가용성보다 단일 복구 절차의 안정성이 더 중요한 상황에서는 Incremental보다는 Full Backup이 더 합리적이다.

---

### 🔹 Log Backup (Binary Log / WAL / Transaction Log)

- 주기: **5~15분 단위**
- 실시간 변경 사항을 기록하여 특정 시점(Point-in-Time) 복구 가능
- 장점: 데이터 손실 최소화
- 단점: 로그 저장소 관리와 복원 시 로그 재적용 과정이 필요

👉 **정책상 제외한 이유:**

고가용성이 필수인 미션 크리티컬 서비스라면 반드시 필요하지만, 이번 환경에서는 로그 관리의 복잡성 대비 이점이 크지 않아 적용하지 않았다.

---

### 📌 최종 전략 선택 배경

1. **데이터 규모와 변경 빈도**: 대규모 트랜잭션 DB가 아닌 비교적 단순한 스키마 → Full Backup만으로도 안정적인 복구 가능
2. **운영 효율성**: 복잡한 Incremental/Log 백업 체계를 구축하는 것보다, 관리와 복구가 단순한 Full Backup 전략이 더 실효적
3. **복원 용이성**: 재난 복구(DR) 시 가장 빠르고 직관적인 방법은 Full Backup 기반의 단일 복구 절차

---

### 🔹 보관 정책

- **최근 7일치 백업 유지**
- **8일 이상 된 백업은 자동 삭제** → `cron` + `find` 명령어 활용
- 이 방식은 스토리지 사용량을 효율적으로 관리하면서도, 최근 일주일 내 어느 시점으로든 복원이 가능하도록 보장한다.

---

### 📊 전략 비교

| 전략 요소 | 전형적 운영 환경 | 본 프로젝트에서의 선택 | 선택 근거 |
| --- | --- | --- | --- |
| **Full Backup** | 주 1회 또는 매일 수행 | **매일 1회 수행** | 데이터 안정성과 복원 단순성 확보 |
| **Differential** | 매일 1회 | 적용하지 않음 | 데이터 변경 빈도가 높지 않아 필요성 낮음 |
| **Incremental** | 6~12시간 단위 | 적용하지 않음 | 복잡성 대비 효용 낮음 |
| **Log Backup** | 5~15분 단위 | 적용하지 않음 | 가용성 요구가 높지 않아 제외 |

---

## 8. DevOps 관점에서의 의의

본 프로젝트를 통해 데이터베이스 설치 및 단순 백업 실습을 넘어, **운영 자동화와 DevOps 원칙**에 대해 고찰해볼 수 있었다**.** DevOps는 개발(Development)과 운영(Operations)의 경계를 허물고, **자동화·표준화·지속 가능성**을 핵심 가치로 삼는다. DevOps 관점에서 본 프로젝트의 의의를 살펴보았다.

### 🔹 1. 다수 사용자 계정 환경에서의 공통 환경 구축

- 서버 내에는 `ubuntu`, `user01`, `user03`, `user04` 등 다수의 사용자 계정이 존재하였다.
- Java 및 MySQL 실행 환경을 **전역 PATH 설정**(`/etc/profile`, `/etc/environment`)을 통해 통합함으로써, 모든 사용자가 동일한 환경에서 개발 및 운영을 수행할 수 있도록 하였다.
- 이를 통해 계정별 환경 불일치로 인한 오류를 사전에 차단하고, **운영 환경 표준화의 필요성**을 실감할 수 있었다.
- 이는 DevOps 실무에서 필수적인 **일관된 환경 관리(Consistency of Environments)** 원칙을 반영한 경험이었다.

---

### 🔹 2. 자동화 스크립트를 통한 반복 업무 최적화

- `mysqldump` 명령어를 단발성으로 실행하는 것에 그치지 않고, **cron + shell script**를 활용하여 정기적이고 자동화된 백업 프로세스를 구축하였다.
- 백업 시점에 따라 SQL 덤프 파일을 생성하고, `tar` 압축을 통해 보관 및 용량 최적화를 수행하였다.
- 관리자의 수동 개입 없이 백업이 이루어지는 구조를 통해 **반복 업무의 자동화와 운영 효율성 증대**를 경험하였다.
- 이는 DevOps가 지향하는 **사람의 개입 최소화와 자동화(Automation of Operations)** 개념을 실습으로 검증한 것이다.

---

### 🔹 3. 벤더사별 DB 백업 기술 비교를 통한 표준화된 정책 수립

- MySQL, PostgreSQL, Oracle, MS SQL Server, MongoDB 등 다양한 DBMS의 공식 백업 방안을 조사·비교하였다.
- 각 벤더별 도구(`mysqldump`, `pg_dump`, RMAN, T-SQL BACKUP, mongodump)의 특징과 **논리/물리 백업, 증분 백업, 시점 복구(PITR)** 지원 여부를 분석하였다.
- 이를 통해 특정 벤더 기술에 종속되지 않고, **Full + Differential/Incremental + Log Backup**을 포함한 **표준화된 백업 주기 및 보관 정책**의 중요성을 확인하였다.
- 즉, 데이터베이스 종류와 상관없이 일관된 백업 전략을 수립하는 것이 **DevOps 관점의 핵심 운영 원칙**임을 체득하였다.

---

### 🔹 4. DevOps 문화적 가치와의 연계

- 서버 환경 설정, 데이터베이스 관리, 백업 및 복구, 문서화 과정을 직접 수행하면서 DevOps의 핵심 가치인 **자동화(Automation)**, **재현성(Reproducibility)**, **지속 가능성(Sustainability)**을 경험하였다.
- 단순한 기술 구현을 넘어, **데이터 운영을 코드화(Infrastructure as Code, IaC)** 하고, 주기적 백업을 통해 **지속 가능한 서비스 운영 체계**를 마련하는 것이 DevOps 문화와 직결됨을 학습하였다.

---

## 9. 향후 개선 방향

본 프로젝트에서는 기본적인 **MySQL 설치 및 백업 자동화 구조**를 구현하였으나, 실제 운영 환경에서는 더 높은 신뢰성과 확장성을 요구한다. 이를 반영하여 다음과 같은 개선 방안을 고려할 수 있다.

### 🔹 1. `systemd` 서비스로 백업 스크립트 관리

- 현재는 `cron`을 통해 주기적으로 백업 스크립트를 실행한다.
- 향후에는 `systemd` 서비스로 등록하여, **서비스 단위로 시작/중지/상태 확인**이 가능하도록 개선할 수 있다.
- `systemctl` 명령을 활용하면 장애 발생 시 자동 재시작, 로깅 일원화, 부팅 시 자동 실행 등 **운영 안정성**이 강화된다.

---

### 🔹 2. 외부 스토리지와 연계한 원격 백업 (S3/NFS 등)

- 현재 백업 파일은 로컬 디스크에만 저장된다.
- 장애나 디스크 손상에 대비하기 위해, **클라우드 스토리지(AWS S3)**, **NFS(Network File System)**, 또는 **백업 서버**로 전송하는 방안을 도입할 수 있다.
- 이를 통해 **오프사이트 백업(Off-site Backup)** 이 가능해지며, 데이터 손실 위험을 최소화할 수 있다.
- 장점: 지리적으로 분산된 환경에서도 동일한 데이터를 안전하게 보관할 수 있음.

---

### 🔹 3. 모니터링 및 시각화 도구 연계 (Grafana, ELK 등)

- 현재는 로그 파일을 직접 확인해야 백업 성공 여부를 판단할 수 있다.
- 백업 스크립트 실행 로그와 결과를 **Prometheus/Grafana** 또는 **ELK Stack(Elasticsearch, Logstash, Kibana)**과 연계하면,
    
    **백업 성공/실패 현황, 실행 시간, 파일 크기, 보관 현황**을 실시간으로 시각화할 수 있다.
    
- 관리자 대시보드에서 백업 현황을 한눈에 확인하고, 실패 시 알람(Notification)을 받을 수 있어 **운영 가시성(Observability)**이 크게 향상된다.

---

### 📌 기대 효과

1. **운영 신뢰성 강화**: systemd 기반 서비스 관리로 안정적인 스케줄링 보장
2. **데이터 안전성 확보**: 원격 스토리지 연계로 재해 복구(Disaster Recovery, DR) 체계 강화
3. **가시성 및 대응 속도 향상**: 모니터링/시각화 기반의 실시간 상태 확인 및 알람 체계

---

## 10. 참고 문헌 및 공식 문서

- [MySQL Backup and Recovery](https://dev.mysql.com/doc/refman/8.0/en/backup-methods.html)
- [PostgreSQL Backup & Restore](https://www.postgresql.org/docs/current/backup.html)
- [Oracle RMAN Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/)
- [MS SQL Backup Documentation](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/)
- [MongoDB Backup Methods](https://www.mongodb.com/docs/manual/core/backups/)
