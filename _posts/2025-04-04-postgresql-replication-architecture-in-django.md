---
layout: post
title: "Django에서 PostgreSQL Replication 설계하기: Physical & Logical 복제와 HA 아키텍처 실험"
date: 2025-04-04 12:00:00 +09:00
categories: [Backend, Database]
tags: [Django, PostgreSQL, Replication, Streaming-Replication, Logical-Replication, HA, WAL, DRF]
---

<h2><strong>1. 데이터베이스 복제를 공부하게 된 이유</strong></h2>

DRF 기반 서비스를 운영하면서 AWS 환경에서 장애가 발생하더라도 서비스가 중단되지 않는 구조를 설계하고 싶었다.

현재 시스템은 단일 PostgreSQL 인스턴스를 사용하는 구조였는데, 이 방식은 다음과 같은 한계가 있었다.

- 단일 장애 지점 (Single Point Of Failure)
- 읽기 요청 증가에 따른 DB 부하 집중
- 서버 장애 발생 시 전체 서비스 영향 가능

특히 서비스 특성상 대부분의 Query가 `SELECT` 연산에 집중되어 있었다.

읽기 트래픽이 쓰기 트래픽보다 많다는 점을 고려하여 아래 목표를 설정했다.

- 실시간 복제(Streaming Replication) 구성
- 조회 전용 Standby DB 구축
- 복제 지연(Replication Lag) 확인
- 장애 발생 시 자동 또는 수동 전환 가능 여부 검증
- Primary DB와 Replica DB 간 데이터 정합성 검증

실제 시스템에 적용하기 전에 PostgreSQL 복제 동작 원리를 이해하고 아키텍처 방향을 먼저 검증하는 것을 목표로 했다.

-----

<h2><strong>2. 데이터베이스 복제란?</strong></h2>

데이터베이스 복제(Replication)는 하나의 데이터베이스 데이터를 여러 인스턴스에 복사하여 저장하는 기술이다.  
이를 통해 시스템 부하를 분산하고 장애 상황에서도 서비스 연속성을 확보할 수 있다.

### 2.1 왜 복제가 필요한가?

사용자가 증가하면 DB의 부하는 자연스럽게 증가하게 된다.
특히 서비스 운영 환경에서는 다음과 같은 문제가 발생할 수 있다.

- 트랜잭션 충돌 증가
- Dead Lock 발생 확률 증가
- 응답 지연 발생
- 서버 장애 상황에서 서비스 중단 가능성

또한 단일 데이터베이스 구조는 Single Point of Failure 문제가 존재한다.  
데이터베이스 서버에 장애가 발생하면 복구 전까지 전체 서비스가 영향을 받게 된다.  

이를 해결하기 위해 복제 구조를 사용하면

- 읽기 요청을 Standby DB로 분산 가능
- Primary 장애 시 복제 DB로 전환 가능
- 데이터 손실 위험 최소화 가능

-----

<h2><strong>3. Master-Slave 구조 (Primary-Replica 구조)</strong></h2>

전통적인 복제 구조는 다음과 같다.  

Primary (Master)
- INSERT / UPDATE / DELETE 수행
- 기준 데이터베이스

Replica (Slave)
- 읽기 전용
- Primary의 복제본

대부분의 서비스는 읽기 요청이 압도적으로 많기 때문에 Replica를 통해 `SELECT` 트래픽을 분산하는 것이 일반적인 전략이다.

-----

<h2><strong>4. WAL (Write-Ahead Logging)</strong></h2>

PostgreSQL Replication을 이해하기 위해서는 WQL 개념은 필수적이다.  
WAL은 프랜잭션 로깅 방식으로, 데이터 파일을 수정하기 전에 변경 내용을 먼저 로그에 기록하는 구조이다.

### 4.1 WAL 동작 방식
1. 모든 변경 사항을 먼저 WAL에 기록
2. 로그가 안전하게 저장된 후 실제 데이터 파일 수정
3. 장애 발생 시 WAL 기반 복구 가능
4. WAL 로그를 Replica로 전송하여 복제 수행

이 방식은 다음 장점을 제공한다.

- Crash Recovery 안정성 확보
- Disk I/O Optimization
- Streaming Replication 지원

PostgreSQL Streaming Replication은 WAL Streaming 기반으로 동작한다.

-----

<h2><strong>5. PostgreSQL Replication 종류</strong></h2>

PostgreSQL Replication은 크게 두 가지로 나뉜다.  

### 5.1 Physical Replication (물리적 복제)

Physical Replication은 클러스터 레벨 복제 방식이다.

특징:

- WAL Block Level 복제 수행
- 전체 Database Cluster 동기화
- Replica Node는 Read-only 구조

적용 상황:

- HA Failover 구조 필요
- 읽기 트래픽 분산이 주요 목적
- 운영 안정성이 최우선

설정은 비교적 단순하지만 특정 테이블 선택 복제는 불가능하다.

### 5.2 Logical Replication (논리적 복제)

Logical Replication은 데이터 논리 레벨 복제 방식이다.

Publisher → Subscriber 구조로 동작한다.

특징:

- Table 단위 선택 복제 가능
- Subscriber Node Write 허용 가능
- 이기종 Database 연동 가능

단점은 다음과 같다.

- Schema 자동 동기화 불가능
- 운영 복잡도 증가

-----

<h2><strong>6. HA 아키텍처 설계 전략</strong></h2>

실제 시스템에서는 다음 구조를 목표로 설계했다.

- Primary Database → 운영 Write 처리
- Physical Replica → Read Traffic 분산
- Logical Replica → 장애 대응 Backup Write Node

이 구조는 단순 복제 구조를 넘어 HA 시스템을 구성하는 전략이다.

---

<h2><strong>7. Troble Shooting 참고</strong></h2>

Replication 구성 중 발생할 수 있는 문제 해결을 위해 다음 차료를 참고 하였다.  
https://blog.ex-em.com/1792?category=1010730  

이러한 개념 정리를 바탕으로 이제 Physical Replication과 Logical Replication을 모두 구성하여  
Primary + Physical Replica + Logical Replica 구조를 설계하였다.

-----

<h2><strong>8. 복제 아키텍처 설계 방향</strong></h2>

실제 시스템 적용을 고려하여 다음과 같은 복제 구조를 목표로 설계하였다.

- Primary Database: 메인 운영 DB
- Physical Replica Database: 읽기 부하 분산용 읽기 전용 DB
- Logical Replica Database: 장애 상황에서 쓰기 전환이 가능한 백업 DB

Physical Replica는 조회 트래픽 분산을 담당하며, Logical Replica는 장애 대응 및 서비스 연속성 확보를 위한 전략으로 활용할 예정이다.

-----

<h2><strong>9. Physical Replication과 Logical Replication 선택 기준</strong></h2>

두 가지 복제 방식은 목적과 운영 환경에 따라 선택해야 한다.

### 9.1 Physical Replication을 선택하는 경우

- 전체 데이터베이스 클러스터 수준의 복제가 필요한 경우  
- 읽기 트래픽 분산이 주요 목적일 경우  
- HA (High Availability) 구성이 필요한 경우  
- 설정이 비교적 단순하고 안정성이 중요한 경우  

Physical Replication은 Streaming Replication 기반으로 동작하며  
Primary 장애 발생 시 Replica 서버로 전환하는 구조에 적합하다.

### 9.2 Logical Replication을 선택하는 경우

- 특정 테이블 또는 데이터만 선택적으로 복제하고 싶은 경우  
- 이기종 데이터베이스 연동이 필요한 경우  
- Replica 서버에서도 쓰기 작업이 필요한 경우  
- 장애 상황에서 서비스 쓰기 포인트를 변경해야 하는 경우  

Logical Replication은 Publisher → Subscriber 구조로 동작하며  
운영 난이도는 상대적으로 높지만 유연성이 높은 구조이다.

-----

<h2><strong>10. 실험 계획 및 검증 전략</strong></h2>

현재 설계한 Primary + Physical Replica + Logical Replica 구조를 기반으로 실제 운영 환경에서 다음 항목들을 중점적으로 검증할 예정이다.

### 10.1 Replication 안정성 검증

- Replication Lag 발생 여부 측정  
- WAL Streaming 상태 지속성 확인  
- Network 장애 상황에서 복제 동작 확인  
- Disk I/O 부하 영향 분석  

Replication Lag는 HA 시스템에서 가장 중요한 지표 중 하나이다.  
Primary와 Replica 간 데이터 반영 지연이 일정 임계값을 초과하면 장애 위험 신호로 판단할 수 있다.

-----

<h2><strong>11. Django ORM 레벨 Routing 전략</strong></h2>

복제 구조를 애플리케이션 레벨에서 활용하기 위해서는 Database Routing 정책이 필요하다.

Django에서는 다음 방식으로 분산 전략을 구현할 수 있다.

- Read Query → Replica Database Routing  
- Write Query → Primary Database Routing  

Routing 구현 시 고려할 사항은 다음과 같다.

- Transaction Scope 유지  
- Lazy Evaluation Query 처리  
- Connection Pool 상태 관리  

특히 Django ORM은 Query 실행 시점이 중요하므로 Transaction 영역을 명확하게 정의해야 한다.

-----

<h2><strong>12. WAL 기반 데이터 동기화 검증</strong></h2>

Streaming Replication은 WAL Log 전달을 기반으로 동작한다.

검증 방법은 다음과 같다.

- Primary 데이터 변경 수행  
- WAL Position Offset 비교  
- Replica Apply Lag 확인  

PostgreSQL 내부에서는 아래 메타 정보를 활용할 수 있다.

- `pg_stat_replication`  
- `pg_replication_slots`  
- `pg_current_wal_lsn()`  

이 지표들은 Replication 상태를 판단하는 핵심 데이터이다.

-----

<h2><strong>13. 장애 감지 및 알림 시스템 설계</strong></h2>

운영 안정성을 위해 장애 감지 자동화 시스템을 구성하는 것이 중요하다.

추천 구조는 다음과 같다.

- Health Check Agent → DB 상태 주기 검사  
- Monitoring Server → 이상 상태 판단  
- Notification System → Slack / Email 알림  

장애 판단 기준은 아래와 같이 정의하였다.

- Replication Connection Loss  
- Replication Lag Threshold 초과  
- Primary Node 접근 실패  
- Transaction 응답 지연 증가  

-----

<h2><strong>14. 데이터 정합성 검증 프로세스</strong></h2>

HA 시스템에서 가장 중요한 요소는 데이터 일관성이다.

검증 방법은 다음과 같다.

- Transaction Commit Verification  
- Row Count 비교  
- Checksum 기반 데이터 비교  
- Logical Sequence Position 비교  

정기적인 Integrity Audit Job을 구성하면 운영 안정성을 크게 향상시킬 수 있다.

-----

<h2><strong>15. 실험 결과 및 시스템 의미</strong></h2>

이번 실험을 통해 PostgreSQL Replication 기반 HA 아키텍처를 설계하고 실제 동작 가능성을 검증하였다.

Physical Replication은 읽기 트래픽 분산 구조에서 안정적으로 동작했으며 WAL Streaming 방식이 데이터 동기화에 효과적임을 확인하였다.

Logical Replication은 특정 데이터 단위 복제가 가능하여 장애 상황에서 쓰기 전환 전략으로 활용할 수 있음을 확인하였다.

HA 시스템 설계에서는 단순 복제 서버 구성보다 다음 요소가 중요하다.

- Replication Lag Monitoring  
- Failover Automation Strategy  
- ORM Routing Policy  
- 장애 감지 및 알림 시스템  
- 데이터 정합성 검증  

-----

<h2><strong>16. 마무리</strong></h2>

데이터베이스 복제는 단순 구현 기술이 아니라 서비스 전체 안정성을 결정하는 아키텍처 요소이다.

실제 운영 환경에서는 애플리케이션 계층 장애 처리, 모니터링 자동화, 그리고 복구 정책이 함께 설계되어야 한다.

앞으로도 실제 서비스 환경에서 검증 가능한 구조를 기준으로 시스템을 지속적으로 개선해 나갈 예정이다.