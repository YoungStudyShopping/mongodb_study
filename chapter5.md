# mongoDB CHAPTER 5 논리적 구조 & 물리적 구조



## mongodb?

- database > collections(=table) > extents > data records > documents(=row)
- schemaless
- c++로 짜여진 오픈소스 데이터베이스
- 다양한 인덱싱과 다양한 쿼리문을 지원함
- 로그, 세션 등 쌓아놓고 삭제가 없는 경우 적합
- 정합성이 떨어지므로 트랜잭션이 필요한 경우에는 부적합
- b트리 인덱스를 사용해서 인덱스를 생성하는데, b트리는 크기가 커질수록 새로운 데이터를 입력하거나 삭제할 때 성능이 저하됨. 그니까 변하지 않는 정보를 조회하는데에 적합하다고
- join이 없으니까 join안해도 되도록 데이터 구조화를 잘해야함



- extent는 데이터가 저장되는 논리적 단위
  - 빅 데이터가 저장되는 컬렉션의 익스텐트 크기를 너무 작게 설정하면 입력 도중에 새로운 익스텐트를 활성화 시키는 횟수가 빈번해지기 때문에 빠른 쓰기 작업의 성능을 지연시킬 수 있음
- 하나의 data recode는 하나의 document 정보와 이전, 이후 document가 저장되어 있는 주소 정보를 함께 저장
- 하나의 extent는 여러 개의 data recode 정보와 이전, 이후 extent에 대한 주소 정보를 함께 저장



1) 프로세스 영역 : mongod.exe, mongo.exe를 통해 서버를 활성화하는 영역

2) 메모리 영역 : MongoDB만을 위한 전용 메모리 영역

3) 파일 영역 : 사용자 데이터, 데이터 Dictionary를 저장하는 디스크 영역



## Storage Engine의 종류

1. MMAPv1 (3.0 이전)
2. wiredTiger (3.0 이후)
3. In-Memory (3.2 이후)



새로운 저장 구조가 이전 저장 구조에 비해 상대적으로 데이터 저장 및 관리에 보다 효율적인 장점을 가지고 있는 것은 맞지만 기업의 데이터는 매우 다양하기 때문에 최신 버전의 저장 엔진을 단순히 선택하여 사용하는 것보다 가장 적절한 저장 엔진을 선택하여 사용하는 것이 무엇보다 중요합니다.



### 1) MMAPv1 스토리지 엔진



![mongodb_internal_structure](http://nicewoong.github.io/assets/mongodb_internal_structure.png)

#### 특징

- 파일 시스템 기반의 저장 엔진
- x86_64 only
- mongodb 4.0때 deprecated 됨
- 초당 10만 건 이상의 빅 데이터에 대해 빠른 쓰기/읽기 작업 처리에 적합
- 자동 복구 작업이 가능하도록 Journal 로그를 제공
  - 서버 장애 발생시 빠른 복구가 보장됨
  - dafault 설정일 경우, 데이터 파일을 60초마다 디스크에 쓰고, 대략 0.1초마다 journal 파일에 씀 (data 파일보다 journal 파일에 더 자주 씀)
  - 데이터 파일에 쓰는 간격을 변경하려면 storage.syncPeriodSecs를 저널 파일에 쓰는 간격을 변경하려면 storage.journal.commitIntervalMs 설정을 참조!
- 싱글 CPU 기반이므로 CPU 개수보다는 충분한 크기의 메모리 자원이 더 요구됨
- collection level concurrency
- mongod --dbpath e:/dev/mongodb/test --storageEngine mmapv1



#### 내부 구조

- Recode : BSON 객체를 저장하는 노드를 레코드로 정의
  - mongodb는 BSON객체를 데이터 저장 단위로 사용
  - BSON 객체의 이중연결리스트 구조로 구성
- Bucket : 인덱스는 레코드에 저장된 데이터를 빠르게 찾기 위해 b-tree 형태로 저장된 노드 구조를 가짐
  - b-tree 노드를 버켓(bucket)이라고 정의



- Extent : mongodb는 대용량 데이터를 HDD에 쉽게 저장할 수 있는 단위로 레코드들을 그룹핑한다.
  - 이를 extent라고 한다.
  - 익스텐드들을 이용하여 mongodb는 HDD에 저장된 파일과 삭제된 레코드를 관리한다.
  - 자료 구조 관점에서 보면 연결되어 있는 레코드들의 헤더 역할을 수행하는 것



#### Recode 저장 특성

- 3.0.0 이전에는 모든 레코드는 디스크 상에 연속적으로 위치했고, document가 할당된 record보다 커지면 새로운 record를 할당해야만 했다. 이럴 경우, document를 이동시켜야 했고 document를 참조하는 모든 색인을 업데이트해야 했다. storage 단편화 발생.

- 3.0.0 이후에는 디폴트로 Power of 2 sized Allocations를 사용한다. 모든 document는 한 record에 저장된다. record가 꽉차면 2의 n제곱의 크기로 recode의 크기를 늘려나간다. document의 이동이 줄어들고, 해제된 record를 효율적으로 재사용하여 단편화를 줄일 수 있다.



### 2) wiredTiger 스토리지 엔진

![wiredtiger_internal](http://nicewoong.github.io/assets/wiredtiger_internal.jpg)



- 파일시스템 기반의 저장 엔진
- 트랜잭션 위주의 데이터를 처리하는데 최적화
- 압축과 암호화 기능 제공
- Point in Time Recovery 복구 기능이 제공됩니다.
- 다중 CPU 기반의 트랜잭션 위주 데이터 처리에 적합하기 때문에 충분한 개수의 CPU와 메모리 자원이 요구됨
- mongod --dbpath e:/dev/mongodb/test --storageEngine wiredTiger
- MMAP 엔진에 비해 적은 시스템 메모리와 디스크 저장 장치로도 구현 가능
- MMAP 엔진와 wiredTiger 엔진은 모든 데이터를 디스크 영역의 파일에 저장 관리할 수 밖에 없기 때문에 디스크 I/O 문제로부터 발생하는 성능 지연 문제가 있음
- b tree구조와 LSM구조를 모두 지원
  - mongodb는 b tree 만 지원. read workloads에 최적화 되어있음



#### Document Level Concurrency

- document 레벨로 concurrency가 제어됨
- 여러 클라이언트가 동시에 컬렉션으 다른 문서를 수정할 수 있다
- 두 작업 간의 충돌을 감지하면 write가 막히고, 해당 작업을 retry한다



####  Snapshots and Checkpoints

- 작업 시작시 wiredTiger는 트랜잭션에 대한 데이터의 특정 시점 스냅 샷을 제공함 (이 체크포인트는 recovery 포인트가 될 수 있다.)
- 3.6 버전부터 60초 간격으로 체크포인트를 생성(스냅샷 데이터를 디스크에 기록)하도록 변경됨
- 새 체크포인트를 쓰는 동안 이전 체크 포인트는 여전히 유효하기 때문에 mongodb가 갑자기 종료되거나 새로운 체크 포인트를 작성하는 동안 오류가 발생하더라도 다시 시작할 때 마지막 유효한 체크 포인트로 복구가 가능함
- 새로운 체크 포인트가 모두 업데이트되어 그 체크 포인트를 참조하게 되면 이전 체크 포인트를 해제하고 새로운 체크포인트가 유효해진다.
- wiredTiger에서는 저널링이 없어도 복구가 가능함. 마지막 체크 포인트 이후의 변경사항을 복구하고 싶으면 저널링을 사용해야함



#### Journal

- 체크 포인트 사이에 저널이 유지되고, mongodb가 저널 데이터를 디스크에 쓰는 빈도는 3.2버전에서는 50 밀리세컨드, 3.6버전 이후는 60초. 저널 파일의 최대 사이즈는 100MB 이고, 이 크기가 넘으면 새 파일이 생성됨
- wiredTiger journal는 snappy 압축 라이브러리를 사용하여 압축된다. 최소 로그 레코드 사이즈는 128bytes 이고, 이거보다 작으면 압축안함.



#### Compression

- gzip과 snappy방식의 압축을 제공
  - snappy : 기본 압축 기능, 압축율이 좋고 오버헤드가 적게 발생
  - gzip : snappy에 비해 매우 높은 압축율을 제공하지만 cpu 오버헤드가 발생

- wiredTiger에서 mongodb는 모든 collection와 인덱스에 대한 압축을 지원함

- dafault로 wiredTiger는 모든 collection에 snappy 압축을 사용하고, 모든 인덱스에 대해 prefix 압축을 사용함. 인덱스의 경우, prefix 압축을 비활성화하려면 storage.wiredTiger.indexConfig.prefixComporession 설정을 사용하면 됨

- 압축 세팅은 collection과 index가 생성되는 동안 각 collection, index 별로 구성할 수도 있다. 

- collection의 사이즈가 더 작아져서 저장공간을 효율적으로 사용 가능하다.

- index에 대해서도 압축 기능을 제공하여 디스크에서도, 메모리에서도 공간 효율이 좋아짐
 
