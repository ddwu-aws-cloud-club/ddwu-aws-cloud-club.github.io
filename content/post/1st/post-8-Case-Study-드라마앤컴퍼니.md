---
title: "Case Study: 드라마앤컴퍼니"
date: "2024-01-20"
author: "정채원, 신이현, 류혜수, 홍성주 (✍️ 유빈)"
tags: ["드라마앤컴퍼니", "ELB", "Auto Scaling"]
categories: ["Case Study", "1st"]
---

> 2023.01.20 / 18:00 - 19:30   
jitsi (온라인)

# 드라마앤컴퍼니의 인프라 아키텍처

## 당면 과제

> 초기에 드라마앤컴퍼니는 국내의 타 클라우드 서비스를 이용하였으나, 잦은 서버 및 네트워크 점검 등의 외부 요인으로 리멤버 서비스가 중단되는 문제가 있었습니다. 뿐만 아니라 서버 사양 변경 불가, 불편한 웹 콘솔, API가 제공되지 않는 점 등의 불편함이 컸기 때문에 서버 이전을 고려하게 되었습니다. 드라마앤컴퍼니의 임세준 CTO는 “클라우드를 사용하지 않으면 서비스가 급성장했을 때 기술적으로 인프라가 이를 절대로 뒷받침할 수 없습니다. 서버를 구매할 경우에는 발주에서 입고까지 몇 주가 걸리고, 호스팅 서비스를 이용한다 해도 직접 소프트웨어 설치 및 하드웨어 관리에 드는 리소스가 급증합니다. 스타트업 특성 상 적은 인력과 합리적인 비용이 중요했고 다양한 매니지드 서비스를 제공하는 클라우드 외에는 다른 대안이 없었습니다.” 라고 말했습니다.
> 

![img](/case_drama/2.51.50.png "img")

### 데이터베이스

## 드라마앤컴퍼니 소개

- 드라마앤컴퍼니는 명함 관리 애플리케이션 ‘리멤버’를 개발
- 리멤버는 받은 명함을 스마트폰으로 촬영하면 수기로 명함 정보를 입력해주는 서비스


💡 ***About Cloud***

### AWS 클라우드 도입 이유

- 잦은 서버 및 네트워크 점검 등으로 리멤버 서비스가 중단되는 이슈 발생
- 서버 사양 변경 불가, 불편한 웹 콘솔, API 제공 X

### 데이터베이스

- 초기 : MySQL용 Amazon RDS 사용
- 현재 : `Amazon RDS for Aurora` (Amazon Aurora)로 이전


## AWS Aurora
![img](/case_drama/Untitled.png "img")

***AWS Aurora***

- MySQL 및 PostgreSQL과 호환되는 완전 관리형 데이터베이스 엔진
- MySQL 및 PostgreSQL 데이터베이스에서 사용하는 코드, 도구 및 애플리케이션 모두 Aurora에서 사용 가능

Amazon Aurora는 표준 **MySQL 데이터베이스보다 최대 5배 빠르고** 표준 **PostgreSQL 데이터베이스보다 3배 빠릅니다**. 또한 1/10의 비용으로 상용 데이터베이스의 보안, 가용성 및 안정성을 제공합니다. 하드웨어 프로비저닝, 데이터베이스 설정, 패치 및 백업과 같은 시간 소모적인 관리 작업을 자동화하는 RDS에서 Amazon Aurora의 모든것을 관리합니다.

### AuroraDB Cluster

> **`Aurora DB 클러스터 = 기본 인스턴스 + 복제본 인스턴스`**
> 

![img](/case_drama/Untitled-1.png "img")
|  | 기본 인스턴스 | 복제본 인스턴스 |
| --- | --- | --- |
| 지원 작업 | 읽기 및 쓰기 작업 | 읽기만 가능 |
| 보유 개수 | 1개 | 최대 15개 |

![4](/case_drama/10.47.57.png "4")

🔽 **Client 요청부터 `AuroraDB`까지의 흐름**

1. Client는 Router53에서 정의한 도메인으로 API 요청
2. Route53은 들어오는 트래픽을 ELB와 연결
3. ELB는 들어온 트래픽을 여러 EC2 인스턴스로 분산
4. EC2에 올린 AP 서버는 필요한 데이터를 **`AuroraDB`**에서 읽거나 쓰는 등 DB 작업 수행

- **드라마앤컴퍼니의 AuroraDB 활용**
    - 자동 장애 조치
    - 자동 스토리지 용량 확장
    
    → DB 관리 비용을 획기적으로 줄임
    

### AWS Aurora 사용법

> 여러가지 사용 방법을 지원하지만 주요한 세 가지 방법은 다음과 같다.
> 

1. **AuroraDB 단독 사용 → `드라마앤컴퍼니 채택 방식`**


💡 Aurora DB를 단독으로 사용하는 경우, 데이터베이스 인스턴스를 **프로비저닝**하고 데이터를 저장한다. 이는 **전통적인 방식**으로 데이터베이스를 관리하고 확장할 때 사용한다.



2. **AuroraDB Serverless 사용**


💡 Aurora Serverless는 사용자의 애플리케이션 요구에 따라 **자동으로 크기가 조절**되는 서버리스 데이터베이스 서비스이다. 서버리스 모델은 사용량에 따라 자동으로 확장 및 축소되므로 유동적이며 비용 효율적이다. Aurora Serverless는 예측할 수 없거나 **가변적인 워크로드에 적합**하다.



3. **AuroraDB와 RDS 연결**


💡 AuroraDB는 RDS에 속하며, **MySQL 및 PostgreSQL과 호환**된다. 따라서 AuroraDB를 사용하면서 다른 RDS 데이터베이스와 연결하여 데이터를 공유하고 활용할 수 있다. 
이는 **다양한 데이터베이스 솔루션을 하나의 애플리케이션에서 혼용**해야 할 때 유용하다.



## AWS RDS vs AWS Aurora

![img](/case_drama/Untitled-2.png "img")

1. **`데이터베이스 엔진 호환성`**
- Aurora: **MySQL 및 PostgreSQL만** 호환
- RDS: MySQL, PostgreSQL, Oracle, MariaDB 등 **다양한 데이터베이스 엔진 지원**

2. **`성능 및 확장성`**
- Aurora: 분산 스토리지 아키턱처 사용 + 자동 확장과 읽기 전용 **replica를 활용**하여 성능 향상
- RDS: 일반적으로 Aurora 보다 **성능 및 확장성에 제한**이 있음

3. **`가격`**
- Aurora: RDS보다 **비쌈** → 성능과 기능 차이를 고려할 때 비용 효율적
- RDS: 다양한 엔진과 인스턴스 유형을 지원하므로 **옵션에 따라 가격 변경**

4. **`다중 가용 영역`**
- Aurora: **멀티 AZ 지원 X** → **Aurora Global Database**를 통해 여러 지역에서 동시에 CRUD 가능
- RDS: **멀티 AZ 지원 O**으로 가용성을 높일 수 있음

**이런 상황에선 RDS, 저런 상황에선 Aurora!**


💡 **RDS**

다양한 데이터베이스 엔진을 지원하고 비교적 사용이 간편하기 때문에 단순한 요구사항이나 비용을 최소화하고 싶을 때 선택




💡 **Aurora**

대규모 트래픽이 예상되거나, 자동화된 기능(자동 백업, 자동 확장, 자동 복원) 등 비용 대비 성능이 중요한 경우에 선택



### 캐시

## ElastiCache

![img](/case_drama/Untitled-3.png "img")

ElasticCache란?

인 메모리 데이터베이스 캐싱 시스템을 제공

→  애플리케이션이 데이터를 검색 할 수있는 성능, 속도 및 중복성을 향상

 

### Memcached와 Redis 비교

Memcached와 Redis 엔진은 둘 다 인 메모리 키 - 값 저장소

![img](/case_drama/Untitled-4.png "img")

데이터 지속성 (Persistence)

- Memcached는 모든 데이터가 유실되지만, Redis는 데이터를 Disk에도 저장하기 때문에 메모리에서 유실된 데이터를 복구할 수 있음
- AOF(Append Only File) Log 와 RDB(Redis Database Backup File) Snapshot 이라는 기능
    
     → in-memory 에 저장된 데이터를 Disk에 백업해주는 기능
    

Data eviction

- LRU (Least Recently Used) 방법
- redis는 추가적인 방법을 통해서도 데이터 관리 가능
    - No Eviction (데이터를 삭제하지 않고 메모리가 차면 key 추가를 허용하지 않음) - 메모리가 부족하면 에러 반환
    - TTL (Time to Live : TTL 표기가 된 데이터를 먼저 지우고 나머지는 아직 유효한 것으로 간주함)
    

Redis **5가지 데이터 형식**

- **String, Set, Sorted Set, Hash, List**

## ElatiCache for Redis

`클러스터`, `복제(Replication) 노드`, `샤드` 설정들을 해주어야 함

클러스터: 여러 컴퓨터가 연결돼 하나의 시스템처럼 동작하는 것, 데이터베이스에서는 안정성과 가용성을 높이기 위해 여러 노드가 협력하는 구조.

샤드: 대용량 데이터를 분산 저장하기 위한 방법, 큰 테이블이나 데이터를 작은 조각으로 나눠 각각을 다른 노드에 저장함으로써 전체 시스템의 성능을 향상시키고 부하를 분산함.

1. 싱글 클러스터 노드(No Replication)
2. 클러스터 모드 없이 복제(Replication)만 지원(클러스터 모드 X)
3. 클러스터 모드와 Replication 모두 지원(클러스터 모드 O)

Primary Node와 Replica Node로 구성하는게 일반적!

 →  shard로 여러 벌 준비하여 data partinioning을 수행하는 것 == 클러스터 모드(clsuter mode)

Untitled%205.png

- 노드(Primary Node) 문제 발생 → Replica Nodes 사용 → 복제 지연으로 인해 일부 데이터 손실 최소화 가능
- 특정 가용 영역에 문제 발생 → Multi-AZ 구성으로 타 가용 영역의 Replica Node 사용

제일 좋은 것은 3번!!! 

트래픽도 여러 샤드로 분산시킬 수 있고 데이터를 Master 노드 + 복제 해놓기 때문에 Master에 장애가 발생해도 복제본을 사용할 수 있음. 

### 각각의 특징 정리

![img](/case_drama/Untitled-6.png "img")

스케일 업(수직확장) 과 스케일 아웃(수평확장)

![img](/case_drama/Untitled-7.png "img")

• 각 단일 서버(하드웨어)의 성능을 증가시켜서 더 많은 요청을 처리하는 방법

즉, 스케일 업은 단일 하드웨어의 성능을 높이기 위하여 CPU, 메모리, 하드디스크를 업그레이드 하거나 추가하는 것

![img](/case_drama/Untitled-8.png "img")

- 동일한 사양의 새로운 서버(하드웨어)를 추가하는 방법

데이터의 증가나 요청량이 증가하더라도 동일하거나 비슷한 사양의 새로운 하드웨어를 추가하면 대응이 가능

즉, 위의 `클러스터 모드`에서는 `Scale Out`을 한다는 큰 장점이 있다 

### 모니터링
[리멤버는 서비스 모니터링을 어떻게 하고 있을까? - DRAMA&COMPANY](https://blog.dramancompany.com/2022/06/how-remember-monitors/)

리멤버 서비스 내에서는 “라이브” 회원 간에 명함을 교환하거나 개인 정보에 변동 사항이 있을 때 푸쉬 알림 기능을 지원하기 위해 Amazon Simple Notification Service(Amazon SNS)도 사용 중입니다. (…중략….) 드라마앤컴퍼니는 Amazon CloudWatch를 통해 서버의 상태를 실시간으로 확인하여 문제가 발생할 경우 각 담당자들에게 메일을 발송하는 Amazon Simple Email Service(Amazon SES)도 이용하고 있습니다.

### AWS 서비스 소개


👀 Cloud Watch?
Amazon CloudWatch는 Amazon Web Services(AWS) 리소스 및 AWS에서 실행되는 애플리케이션을 실시간으로 모니터링 할 수 있다. CloudWatch를 사용하여 리소스 및 애플리케이션에 대해 측정할 수 있는 변수인 지표를 수집하고 추적할 수 있다.




💬 SNS?
**Amazon Simple Notification Service(Amazon SNS)**는 구독 중인 엔드포인트 또는 클라이언트에 메시지를 전달 또는 전송하는 것을 조정하고 관리한다. CloudWatch와 함께 Amazon SNS를 사용하여 경보 임계값에 도달한 경우 메시지를 전송한다.




🗨️ SES?
Amazon Simple Email Service(SES)는 사용자의 이메일 주소와 도메인을 사용해 이메일을 보내고 받기 위한 경제적이고 손쉬운 방법을 제공하는 이메일 플랫폼



### CloudWatch를 사용하여 Amazon SNS 주제 모니터링 하는 법

### 리멤버가 모니터링하는 것들

****Application Performance****

> AP는 일반적으로 다음과 같은 일반적인 지표를 추정한다. CPU 사용량, 평균 응답 시간, 오류율, 트랜잭션 추적, 인스턴스, 요청, 가동시간 중 리멤버는 현재 New Relic으로 throughtput, 평균 응답 시간, 오류 발생율, 슬로우 트랜잭션 및 cpu/메모리 사용율을 주로 확인한다.
> 

****Infrastructure Metrics****

> 서버 프로세스가 아닌 다른 요인으로 서버에 문제가 발생했을 경우 New Relic으로 확인하고 있다. 특히 트래픽 별 IO양, 디스크 사용율은 물론 Infrastructure의 관점의 여러 가지 metrics를 통해 Load average와 CPU 사용율을 중심으로 문제가 되는 지점을 파악할 수 있다. 이후 동일한 문제가 발생하지 않기 위해 자동화.
> 

****SLI/SLO****

> 아직 제대로 된 거 없고 이제 막 시작하는 단계!
> 

**특정 서비스의 health 를 나타내는 지표**

> 리멤버 도메인 특성상 관리해야 하는 지표가 있을 경우 상황에 따라 개발자들이 인지 후 조치를 위해 CloudWatch에 custom metric 전송 후 alarm을 만들어 관찰한다.
> 

### 리벰버가 모니터링 도구로써 AWS CloudWatch와 New Relic를 사용하는 법, 그리고 차이점은?

(1) 문제 발생 시 Slack으로 메시지를 수신

- NewRelic의 경우
    - terraform 모듈을 이용해 각 애플리케이션 마다 Slack으로 알람 메시지 전송하는 Alert 관리
    - Alert 발생 시 과정을 스레드로 기록
- AWS CloudWatch + SNS
    - slack으로 알람 메시지가 전송

(2) 사용자 문의 및 모니터링 중 이상징후 발견

- 특정 사용자의 개인적인 문제 대응
    - Logs에서 해당 유저가 요청한 API목록 확인
- 서비스 지표에 이상징후에 대한 대응.
    - 두 모니터링 서비스 모니터링

### 리멤버가 CloudWatch를 사용할 때 주의했던 점

#### 애플리케이션 로그를 CloudWatch 에 쌓지 말자

AWS의 CloudWatch Log는 여러 서비스와 연동이 편리하지만 비용이 다른 로그 저장소들에 비해서 매우 높다.

CloudWatch Log는 수집 항목이 $0.76 per GB에 비해 NewRelic에서 한 달 기간 동안 스토어 비용 포함해 $0.3 per GB이다.

NewRelic처럼 대체제가 있기 때문에 어플리케이션 로그를 포함한 여러 형태, 여러 소스의 로그들을 CloudWatch에 쌓이지 않게 할 수 있다면 로그를 남기지 않는 것이 비용 절감에 매우 도움이 된다.