# MongoDB Chapter8 - MongoDB 성능 튜닝

## 8.1 디자인 튜닝
- 데이터를 저장하는 논리적 구조인 **컬렉션의 분석과 설계 작업 부족**

### 1) Extent 크기 조절

```bash
db.createCollection("employees", {capped:true, size:100000 });
# SIZE : Collection의 최초 크기
```
- Extent가 작으면 Extent가 증가될 때 불필요한 성능 지연이 발생
- 상황에 따른 Extent 크기 설정
	- Extent의 크기를 충분히 할당하는 경우
		- 대용량 데이터의 INSERT가 발생하는 컬렉션
		- 대용량 데이터의 Full Collection Scan이 자주 발생하는 컬렉션
	- Extent의 크기를 작게 설계하는 경우
		- Index Scan이 자주 발생하는 컬렉션
- Collection의 Extent 정보 확인

```bash
db.employees.validate(); # Collection의 현재 상태 및 정보 분석(WiredTiger 저장 엔진)
{
	"ns" : "test.employees",
	"nInvalidDocuments" : NumberLong(0),
	"nrecords" : 0,
	"nIndexes" : 1,
	"keysPerIndex" : {
		"test.employees.$_id_" : 0
	},
	"valid" : true,
	"warnings" : [
		"Some checks omitted for speed. use {full:true} option to do more thorough scan."
	],
	"errors" : [ ],
	"ok" : 1
}
```
```bash
db.employees.validate(); # Collection의 현재 상태 및 정보 분석(MMAP 저장 엔진)
{
    "ns" : "test.employees",
	"capped" : 1,
	"max" : 2147483647, # 해당 Extent의 최대 크기
	"firstExtent" : "0:4000 ns:test.employees", # 첫 번째 Extent 주소
	"lastExtent" : "0:4000 ns:test.employees", # 마지막 Extent 주소
	"extentCount" : 1, # 할당된 Extent 수
	"datasize" : 0,
	"nrecords" : 0,
	"lastExtentSize" : 102400, # 현재 Extent 크기
	"padding" : 1,
	"firstExtentDetails" : { # 첫 번째 Extent의 상세 정보
		"loc" : "0:4000",
		"xnext" : "null",
		"xprev" : "null",
		"nsdiag" : "test.employees",
		"size" : 102400",
		"firstRecord" : "null",
		"lastRecord" : "null"
	},
	...
	"ok" : 1
}
```

- 성능 향상 포인트
	- 해당 컬렉션에 초당 얼마나 많은 빅 데이터가 저장될 것인지 비즈니스 룰 분석
	- 하루, 일주일, 한달, 6개월, 1년 등 얼마나 많은 데이터를 저장/관리할 것인지 데이터 발생량을 파악

### 2) 설계방식

- **Embedded/Extend Document** 설계 : 컬렉션과 컬렉션 간의 업무적 관계가 밀접한 경우
	- 4장에서 소개한 "주문전표"예시
		- "주문전표"는 '주문공통'과 '주문상세' 2개의 테이블로 '주문번호'라는 공통 컬럼이 배치되는 관계 구조
		- "주문전표"를 통해서 '주문공통'과 '주문상세' 테이블이 만들어지므로 함께 저장되고 함께 참조되는 데이터
	- 이 두 테이블에 빅데이터가 저장된다면 논리적, 물리적으로 분리된 저장 구조에 동시 쓰기 작업을 수행해야 하므로 성능 지연
	- 이릭기 작업에서도 불필요한 결합을 유발시키기 떄문에 비 효율적
- **Link(DBRef) Document** 설계 : 컬렉션과 컬렉션 간의 업무적 관계가 밀접하지 않은 경우

```bash
db.ord.insert(
{ ord_id		 : "2012-09-012345", # 주문 공통 정보
  customer_name  : "Wonman & Sports",
  emp_name       : "Magee",
  total          : "601100",
  payment_type   : "Credit",
  order_filled   : "Y",
  item_id        : [ { item_id      : "1", # 주문 상세 정보
					   product_name : "Bunny Boots",
					   item_price   : "135",
					   qty          : "500",
					   price        : "67000" },
					 { item_id      : "2",
					   product_name : "Pro Ski Boots",
					   item_price   : "380",
					   qty          : "400",
					   price        : "152000" }
                   ]
} ) # 내장형 도큐멘트 구조
```

### 3) 인덱스 사용

- 인덱스의 가장 큰 목적은 빅데이터로부터 조건을 만족하는 데이터를 빠르게 검색하기 위함
- MongoDB의 밸런스-트리 인덱스는 기존 관계형 데이터베이스에서 제공되던 밸런스-트리 인덱스와 구조, 생성, 관리 방법 등이 동일

```bash
db.employees.createIndex({empno : 1, deptno : -1, sal})
# EMPNO를 기준으로 생성하는 복합 인덱스
db.employees.createIndex({empno : -1, deptno : 1, sal})
# DEPTNOfmf rlwnsdmfh 생성하는 복합 인덱스
```

- MongoDB의 인덱스 키 종류
	- 싱글 키 인덱스 : 하나의 키로 만든 인덱스
	- 복합 키 인덱스 : 2개 이상의 키로 만든 인덱스
		- 필드의 우선순위에 따라 인덱스의 성능을 좌우

```bash
db.employees.find({empno:7369}).forEach(printjson) # 동등 조건
db.employees.find({deptno:{$in: [10, 30]}}, {empno:"", deptno:""}) # 비 동등 조건
db.employees.find({sal : {$gte: 1500, $lte: 5000}}, {empno:"", ename:"", sal:""}) # 비 동등 조건
```

- 복합 인덱스를 생성할 때 필드의 우선 순위를 결정하는 기준
	1. 자주 검색 조건으로 사용되는 필드를 선행 필드로 결정
	2. 모든 필드가 자주 사용될 경우 분포도가 좋은 필드를 선행 필드로 선택
		- 분포도가 좋다 : 데이터가 전체 데이터 중에 약 10%내외로 저장되어 있는 상태
	3. 1번과 2번으로 결정할 수 없다면 데이터 양이 적은 필드를 선행 필드로 선택
	4. 모든 기준으로 결정이 불가능하다면 동등 조건(Equal)으로 검색되는 필드를 선택

- 빅데이터를 저장하는 컬렉션의 경우에는 백그라운드 인덱스의 사용을 적극적으로 고려(3장 참조)

---

## 8.2 문장 튜닝
- 개발자가 작성한 **Query가 적절하지 않은 경우**

### 1) Profiling System
- MongoDB는 서버와 사용자에 의해 실행된 Query 문장의 실행 결과에 대해 Profiling System을 제공
- 결과는 db.system.profile Collection에 저장(시간을 기반으로 하는 Historical 데이터가 저장)
- Query 문장이 실행될 때마다 모든 문장들의 로그 정보는 MongoDB의 Profiling System에 저장됨
- 로그 정보 중에 일정시간이 초과된 문장들을 추출하여 성능 튜닝을 수행해야 함

```bash
db.commandHelp("profile") # Profiler 최초 설정을 위한 Help
help for: profile controls the behaviour of the performance profiler, the fraction of eligible operations which are sampled for logging/profiling, and the threshold duration at which ops become eligible. See http://docs.mongodb.org/manual/reference/command/profile
db.setProfilingLevel(2) # was: 이전 설정 정보
{ "was" : 0, "slowms" : 100, "sampleRate" : 1, "ok" : 1 }
# 0: Off, 1: default > 100ms인 정보, 2: System에서 발생한 모든 정보
db.getProfilingLevel() # 현재 설정되어 있는 정보
2
```
```bash
mongod --help # 인스턴스를 최초 시작할 때 설정도 가능
```
#### 1-1. Profiler 환경 분석 결과 및 상태 확인
- 컬렉션을 검색하는 문장을 실행한 뒤 새로운 세션에서 명령어 실행하면 모든 로그 정보를 출력

```bash
db.system.profile.find() # 모든 상태 정보 출력
db.system.profile.find({millis: {$gt:5}}) # 조건 검색된 상태 정보(5초 이상 소요된 Query만 추출)
```

#### 1-2. Profile Collection의 재생성 및 관리
- db.system.profile Collection에 저장되는 공간을 관리
	- 컬렉션에 너무 많은 로그 정보를 저장해 두는 것은 공간 낭비/관리의 비효율 문제 야기
	- 주기적으로 재생성해주는 것이 좋음

```bash
db.setProfilingLevel(0) # 재생성을 위해 환경을 0으로 설정
{ "was" : 2, "slowms" : 100, "sampleRate" : 1, "ok" : 1 }
db.system.profile.drop() # Profile Collection 삭제
true
db.createCollection("system.profile", {capped:true, size:4000000}) # profile 컬렉션 재 생성
{ "ok" : 1 }
db.system.profile.stats() # profilling 컬렉션의 상태 확인
```
#### 1-3. Profiling 분석 결과
- Profiling 결과 분석 방법
	1. 문장 실행 날짜 정보
	2. 실행된 문장 타입(Update, Find, Delete, Insert)
	3. DB명 & Collection명
	4. 실행된 문장에 부여된 고유 ID
	5. SCAN된 Index-Key 수
	6. SCAN된 DOC 수
	7. Update가 요구되는 Index 수
	8. Lock Yield 수
	9. 실제로 Return된 Document 수
	10. 리턴된 Document 크기
	11. 소요된 시간
	12. Index를 통해 리턴된 Document tn
	13. 검색된 Index 명

### 2) Hint 함수 / Explain 함수

- Hint() : INDEX 사용이 가능할 수 있도록 실행계획을 정의할 수 있음
	- 힌트절은 관계형 DBMS보다는 제한적
- Explain() : Query Plan을 모니터링할 수 있는 Method
	- Explain()함수를 통해 Full Collection Scan되는 문장들을 분석(튜닝 대상)
	- Query 문장의 성능 저하 원인 중 하나는 인덱스가 없거나, 인덱스를 통한 검색이 수행되지 못하는 경우

---

## 8.3 아키텍처 튜닝
- 샤딩 시스템, ReplicaSets을 구축하면서 발생하는 문제

### 1) ReplicaSets 분리
- ReplicaSets 영역과 사용자의 비즈니스 Area 영역은 반드시 분리해서 구축
	- ReplicaSets은 Primary 서버에 장애가 발생하는 경우 대비하여 복제되는 클론 DB이기 때문에 실시간으로 지속적인 데이터의 복제가 수행됨
	- 시스템의 유연성/확장성/복구성을 고려한다면 반드시 분리 운영
	- 하나의 서버에 다양한 기능 설정을 하면 성능이 저하될 수밖에 없음

### 2) 여러대의 서버 구축
- 하나의 싱글 노드로 운영하는 것은 빠른 쓰기 작업이 불가능
	- 하드웨어 사양이 좋아도 디스크에 쓰는 데에는 한계가 있음(이럴 때 MongoDB 샤드 시스템 이용)
	- 여러대의 서버로 시스템을 구축/운영하면 빠른 쓰기 및 Load Balancing을 통한 시스템 성능 향상 가능
	- 6장 샤드 서버의 추가와 삭제 참조

### 3) Slave 서버에서 백업/데이터 분석 하지 않기
- ReplicaSet의 Slave 서버에서는 백업 및 데이터 분석 작업 지양
	- 슬레이브에서는 실시간으로 마스터 서버의 데이터가 복제되므로 과부하가 발생할 수 있음
	- 필요하다면 마스터 데이터베이스를 통해 작업 수행

### 4) 하나의 서버에는 하나의 mongod.exe 인스턴스 활성화
- MongoDB는 많은 Memory Mapped 영역을 요구하므로 불필요하게 여러 MongoDB.exe를 수행하개 되면 메모리 부족으로 인한 성능 지연 발생
- 피크타임에 페이지폴트가 지속적으로 증가하면 메모리 부족으로 성능 지연 문제 발생
	- 충분한 시스템 메모리 영역을 추가 할당
- 시스템 사양 확인하기

```bash
db.serverStatus().mem
{
	"bits" : 64, # 시스템 사양
	"resident" : 35, # 물리적 메모리 영역 크기
	"virtual" : 4984, # Virtual 메모리 현재 크기
	"supported" : true,
	"mapped" : 0, # Mapped File 현재 크기
	"mappedWithJournal" : 0 # Journal 메모리 현재 크기
}
db.serverStatus().extra_info
{ "note" : "fields vary by platform", "page_faults" : 0 }
```

- MongoDB 잠금(Lock) 상태를 모니터링하기
	- 지속적으로 증가하면 Lock 문제로 인한 성능 지연 현상이 발생하고 있음을 의미

```bash
db.serverStatus().globalLock
{
	"totalTime" : NumberLong("24898450000"), # 전체 Lock time
	"currentQueue" : { # 최근에 발생한 Lock time
		"total" : 0,
		"readers" : 0,
		"writers" : 0
	},
	"activeClients" : {
		"total" : 12,
		"readers" : 0,
		"writers" : 0
	}
}
```

---

## 8.4 인스턴스 튜닝
- **충분한 메모리 영역**이 할당되지 못한 경우(MongoDB는 메모리 매핑 기술을 이용한 데이터 처리 기법을 활용)

### 1) 문장 튜닝 먼저 수행
- Explain Plan과 Profiler 기능을 사용
	- 데이터베이스 튜닝은 가장 빠른 시간 내에 가장 효과적인 방법으로 성능을 개선시킬 수 있는 방법으로 수행해야 함
	- 일반적인 성능 지연 문제중 가장 큰 문제는 Query 때문
	- 인스턴스 튜닝을 아무리 수행해도 성능 개선엔 한계가 존재하기 때문에 Query 문장부터 튜닝 후 인스턴스 영역 튜닝 진행
- MongoDB의 Resident 영역이 시스템 전체 메모리 영역의 90% 이상을 점유하게 되는 경우 성능 지연 현상을 해결하기 위해 시스템 메모리를 추가로 할당해야 함
	-  peak time 때 resident 영역은 시스템 전체 메모리 영역의 70~80%를 유지하는 것이 가장 이상적인 성능 보장

### 2) 최적화된 Read/Write가 가능하도록 문장 작성
- 컬렉션에 Document를 저장할 때 사용할 수 있는 문법
	- SAVE : 데이터를 저장할 때 하나의 도큐멘트 단위로 데이터를 저장
	- UPDATE : SET 절에 정의된 필드 단위로 데이터를 저장
	- SAVE는 Document 전체 필드를 변경할 때는 유리하지만, 특정 필드만 변경할 때는 불필요하게 다른 필드까지 함께 저장해야 하기 때문에 성능 지연 발생
- 자주 검색되는 필드에 대해서는 적절한 인덱스를 생성하여 불필요한 필드에 대해 읽기 작업이 발생하는 것을 피해야 함
- Resident 영역을 최소화시켜 인스턴스 영역에 대한 성능 향상 고려

### 3) Data Locality(인접성)이 높아지도록 컬렉션을 설계하고 저장
- 빅데이터를 효과적으로 저장하기 위해서는 유사한 성격의 데이터들을 인접한 데이터 영역에 저장하는 것이 좋음
	- 읽기 작업에서도 빠른 성능을 보장해 줄 수 있음
	- 불필요한 데이터에 대한 쓰기/읽기 작업을 피할 수 있어 인스턴스 영역의 효율성을 향상시킬 수 있음

---

## 8.5 하드웨어 튜닝
- 운영체제 및 HW 환경에 대해 최적화가 수행되어야 좋은 성능을 기대 할 수 있음
- 책 한번씩 읽어보기
  1. 운영체계 환경에 대한 고려
  2. 하드디스크 환경에 대한 고려
  3. File System 환경에 대한 고려
  4. 저장엔진에 대한 고려
  5. Replication 서버 환경에 대한 고려
  6. Sharding 서버 환경에 대한 고려
