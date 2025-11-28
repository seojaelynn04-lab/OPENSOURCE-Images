# Redis 조사 보고서 (정리본)
## 1. 도입 배경 및 필요성
### 1.1 Redis 탄생 배경<br>

Redis는 이탈리아 개발자 (Salvatore Sanfilippo)가 MySQL 기반 서비스의 확장 문제를 해결하기 위해 만든 인메모리 데이터베이스<br>
당시 개발자가 운영중이던 서비스는 쓰기 요청이 비정상적으로 많은 구조였고, 페이지가 로딩될 때마다 자바스크립트 코드가 DB에 write 작업을 발생시킴 -> 초당 수 만건의 쓰기 작업을 기존 MySQL 기반 구조로는 감당하기 어려움<br>
이 문제를 해결하기 위해 디스크 I/O 병목을 제거하고 실시간에 가까운 데이터 저장·조회가 가능한 시스템을 직접 만들기 시작했고, 그 결과 Redis 베타 버전이 탄생<br>
<br>
(출처 : https://www.eu-startups.com/2011/01/an-interview-with-salvatore-sanfilippo-creator-of-redis-working-out-of-sicily/
 -> Salvatore Sanfilippo 인터뷰 기사)<br>

### 1.2 기존 데이터베이스 대비 Redis의 개선점<br>

**① 기존 RDBMS 한계**<br>

높은 지연 시간 : 디스크 기반 구조로 인해 읽기·쓰기 시 물리적인 I/O가 필요함 → 속도 제한 존재<br>

동시성 증가 시 성능 하락 : Connection 수가 늘어날수록 처리 비용이 커짐<br>

캐싱 필요성 증가 : 반복 조회가 많은 서비스는 DB 부하를 초래<br>

**② 인메모리 스토리지의 등장**<br>
Redis는 모든 데이터를 RAM에 저장하기 때문에 디스크 지연을 완전히 제거함 : 엄청난 응답속도 제공<br>
RDBMS처럼 강제된 테이블 구조가 아닌 스키마리스 Key-Value 모델을 기반으로 하되, 문자열뿐 아니라 리스트, 해시, 셋 등 다양한 자료구조를 지원해 복잡한 로직을 단일 명령으로 처리할 수 있게 함<br>
<br>
(출처 :https://www.elancer.co.kr/blog/detail/768 -> 개발 테크 블로그)<br>
(출처 :https://redis.io/learn/develop/node/nodecrashcourse/whatisredis -> redis 공식 사이트)<br>

## 2. Redis 아키텍처 분석
### 2.1 기본 동작 구조<br>

**① 단일 스레드 기반**<br>
Redis는 모든 명령을 단일 스레드로 처리 (겉보기에는 느릴 것 같지만)<br>

Lock 경쟁이 없어 성능 예측 가능<br>

Context switching 비용 없음<br>

데이터 일관성 확보가 쉬움<br>

Redis는 내부적으로 I/O Multiplexing(이벤트 루프 기반 Non-blocking I/O)을 사용하기 때문에 명령 실행은 하나씩 처리하지만, 여러 요청을 한꺼번에 받아 빠르게 큐에서 처리할 수 있음<br>

**② In-Memory 저장 구조**<br>

모든 연산이 RAM에서 이뤄짐<br>

디스크 I/O가 없어 병목 현상이 거의 없음<br>

실시간 처리에 최적화된 구조<br>
<br>
https://adjh54.tistory.com/447#1)%20Redis(Remote%20Dictionary%20Server)%20%EB%B0%8F%20%EA%B5%AC%EC%A1%B0-1)<br>

**③ 영속성(Persistence) 지원**<br>
Redis는 인메모리 DB지만 데이터 유지를 위한 다양한 저장 방식도 제공<br>

AOF(Append Only File) : 모든 write 연산을 로그로 기록 → 복구 정확도 높음<br>
단점) 파일이 커지고 로딩 시간이 길어짐<br>

RDB(Snapshot) : 특정 시점 기준으로 메모리 내용을 통째로 저장 → 빠름<br>
단점) 스냅샷 사이 구간의 데이터는 유실 가능<br>

혼합 모드 : 로딩 속도 + 안전성을 둘 다 확보하는 가장 현실적인 방식<br>
<br>
(출처: https://inpa.tistory.com/entry/REDIS-%F0%9F%93%9A-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%98%81%EA%B5%AC-%EC%A0%80%EC%9E%A5%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%9D%98-%EC%98%81%EC%86%8D%EC%84%B1#aof_%ED%8C%8C%EC%9D%BC%EB%AA%85_%EC%A7%80%EC%A0%95)<br>

### 2.2 전체 구조 설명<br>

Redis는 기본적으로 메모리 기반 데이터 구조 서버로 동작하며 필요 시 디스크에 영속화할 수 있음<br>

![image](https://github.com/seojaelynn04-lab/OPENSOURCE-Images/blob/main/image01.png)<br>
(간단한 구조 설명)<br>
Client → Redis 서버에 명령 요청<br>
Redis 서버 → 명령을 해석하고 메모리에 데이터를 저장·수정<br>
Master–Slave 구조 → Master의 데이터를 Slave로 복제<br>
Persistence → RDB/AOF 방식으로 디스크에 저장 가능<br>
(※ 다이어그램 설명: Redis 서버 중심으로 클라이언트, 데이터 저장 구조, 복제 구조가 시각적으로 표현된 형태)<br>

### 3. 주요 기능
**(1) 캐싱**<br>

인메모리 캐시로 활용하여 액세스 지연 시간을 줄이고, 처리량을 늘리며, NoSQL 데이터베이스의 부담을 줄여줌<br>
대부분의 웹 서비스에서 사용<br>

**(2) 세션 관리**<br>

세션 키에 TTL을 설정하여 손쉽게 만료 관리 가능<br>

로그인·인증 서비스 등 다양한 애플리케이션에서 활용<br>
<br>
(출처: https://aws.amazon.com/ko/elasticache/what-is-redis/)<br>

**(3) 분산 시스템 구성**<br>

Cluster 모드를 통한 샤딩, 고가용성 구성 가능<br>

샤딩이란? : 데이터를 나눠서 여러 서버에 저장하는 구조, 서버 여러 대의 메모리를 합쳐 하나처럼 사용, 트래픽도 분산되어 훨씬 빠름<br>

**(4) 데이터 백업 및 복구**<br>

RDB : 메모리에 있는 데이터 전체에 대해 스냅샷을 저장하고 디스크에 저장함 -> 데이터 복원이 필요할 때 스냅샷만 로딩해오면 됨 <br>

AOF : 모든 쓰기 명령을 로그로 기록해둠(데이터 변경시에도 변경기록을 로그로 기록해두기 때문에 백업이 가능함) => RDB 방식보다 데이터 유실량이 적으나, 로딩 속도가 느리고 파일 크기가 급격히 커짐<br>
→ 두 방식을 함께 사용하면 안정성 증가<br>
<br>
(출처: https://hgggny.tistory.com/entry/Redis-%EC%B4%88%EB%A9%B4%EC%9E%85%EB%8B%88%EB%8B%A4-%EC%A0%95%EC%9D%98-%EA%B5%AC%EC%A1%B0-%EC%9E%A5%EB%8B%A8%EC%A0%90-%ED%99%9C%EC%9A%A9-%EC%82%AC%EB%A1%80)<br>

**(5) 복제(Replication)<br>**

마스터 서버의 데이터를 실시간으로 다른 Redis 서버(Slave)에 복사할 수 있음<br>

장애 발생 시 Slave를 승격해 서비스 지속 가능<br>

데이터 백업 및 읽기 부하 분산 효과<br>

RDB/AOF 파일 크기가 커도 실시간 복제로 복원 시간 최소화<br>

![image](https://github.com/seojaelynn04-lab/OPENSOURCE-Images/blob/main/image02.png)<br>
Master Redis Server : 실제 데이터를 저장하는 서버<br>
Slave Redis Server : 실제 데이터를 복사하여 전달받는 서버<br>
<br>
(출처 : https://server-talk.tistory.com/496)<br>

## 4. Redis의 Trade-off
**① 단일 스레드 기반의 한계**<br>

CPU 연산이 많은 작업에서는 병목 발생 가능<br>

멀티코어 활용이 어려움<br>

**② 데이터 유실 가능성**<br>

RDB 스냅샷 사이 구간은 유실 여지 있음<br>

AOF는 안전하지만 I/O 부담 증가<br>

**③ 메모리 기반 특성 → 높은 비용**<br>

RAM은 디스크보다 훨씬 비싸기 때문에 대규모 데이터 저장에는 비경제적<br>

실제로는 핵심 데이터만 Redis에 두고 나머지는 RDBMS에 저장하는 하이브리드 구조가 일반적으로 사용됨<br>
