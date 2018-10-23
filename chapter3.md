# 3-1 인덱스의 생성과 관리



## 목차

1. 인덱스란?
2. MongoDB 인덱스의 특징
3. 인덱스 생성
4. 인덱스 조회
5. 인덱스 삭제
6. 인덱스 재구성
7. 실행계획 확인



## 1. 인덱스란?

>**인덱스**([영어](https://ko.wikipedia.org/wiki/%EC%98%81%EC%96%B4): index)는 [데이터베이스](https://ko.wikipedia.org/wiki/%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4) 분야에 있어서 [테이블](https://ko.wikipedia.org/wiki/%ED%85%8C%EC%9D%B4%EB%B8%94_(%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4))에 대한 동작의 속도를 높여주는 [자료 구조](https://ko.wikipedia.org/wiki/%EC%9E%90%EB%A3%8C_%EA%B5%AC%EC%A1%B0)를 일컫는다.
>(출처: [wikipedia](https://ko.wikipedia.org/wiki/%EC%9D%B8%EB%8D%B1%EC%8A%A4_(%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4)))





## 2. MongoDB 인덱스의 특징

1. Index 명은 Case Sensitive (대소문자 구별)
2. 문서 Update 시 해당 Index Key 만 변경되지만, 
   변경되는 문서의 크기가 기존 EXTENT 공간 크기보다 크면, 
   더 큰 EXTENT 공간으로 마이그레이션 될 수 있고 성능 저하 발생 가능
   => 충분한 EXTENT 크기로 설계 필요  (5장 - 논리 & 물리 구조 설계에서 다룸)
3. sort() 와 limit() 같이 사용해 불필요한 검색 줄여 성능 향상시킬 것



## 3. 인덱스 생성

#### 명령어: `createIndex()`



````javascript
db.employees.createIndex(
    {eno: 1}, {unique: true} // 1: Asc, -1: Desc
);

db.employees.createIndex({job: -1});
````



<img src="https://user-images.githubusercontent.com/20942871/46903409-5908ad00-cf0f-11e8-87c4-82e16b513aee.png" width="70%" />



* 기본 인덱스는 _id 값으로 생성된다.





## 4. 인덱스 조회

#### 명령어: `getIndexes()`



````javascript
db.employees.getIndexes()
````



<img src="https://user-images.githubusercontent.com/20942871/46903418-70e03100-cf0f-11e8-9ae8-8083c4dd48dd.png" width="70%"/>





## 5. 인덱스 삭제

#### 명령어: `dropIndex()`, `dropIndexes()`



````javascript
db.employees.dropIndex({eno: -1})
````

<img src="https://user-images.githubusercontent.com/20942871/46903427-a84edd80-cf0f-11e8-8bd2-b4760d1597a3.png" width="70%" />



````javascript
db.employees.dropIndexes()
````

<img src="https://user-images.githubusercontent.com/20942871/46903441-e21fe400-cf0f-11e8-9be0-dfac0deae747.png" width="70%"/>





## 6. 인덱스 재구성

### 왜 재구성을 해야하나?

MongoDB에선 B* Tree 인덱스를 기본으로 사용하여 빠른 검색이 가능 ([B Tree 참고](http://blog.naver.com/eng_jisikin/220889188747), [B* Tree 참고](http://egloos.zum.com/sweeper/v/899699)) 

그러나 사용 과정에서 **입력 & 삭제가 빈번**히 발생하면 **인덱스 구조의 변형으로 인한 불균형이 발생**해 **빠른 성능이 보장되지 않을** 수 있음.

이런 경우 **유일한 해결책은 인덱스 구조의 재 구성**



#### 명령어: `reIndex()`



````javascript
db.employees.reIndex()
````

<img src="https://user-images.githubusercontent.com/20942871/46903748-5066a580-cf14-11e8-85d8-9f0902a0b2ee.png" width="50%" />







## 7. 실행계획 확인

#### 명령어: `explain()`

````javascript
db.employees.find({empno: 10}).explain()
````

<img src="https://user-images.githubusercontent.com/20942871/46904176-22389400-cf1b-11e8-9768-6735685d126f.png"/>


---


# 3-2 인덱스의 종류

### 인덱스 종류 및 옵션

1. Single-Key / Compound Key
2. Unique / Non-Unique
3. Sparse
4. Partial
5. Background (인덱스 생성 옵션)
6. Covered (검색방식)
7. TTL (인덱스 관리 옵션)
8. GeoSpartial
9. GeoMetry



## 1. Single-Key / Compound Key

말 그대로 인덱스의 키가 1개 or 여러개 의 차이



#### 싱글 키 인덱스

````javascript
db.employees.createIndex({empno: 1}) // 1: Asc, -1: Desc
````



#### 복합 키 인덱스

````javascript
db.employees.createIndex({empno: 1, deptno: -1})
````





## 2. Unique / Non-Unique

인덱스의 값들이 유일값(Unique) 인지 중복값이 있는지(Non-Unique)의 차이

기본값: false (Non-Unique)



#### Unique Index

````javascript
db.employees.createIndex({empno: 1}, {unique: true})
````

* 인덱스 생성 전 이미 대상에 중복값이 있었다면 에러 발생



#### Non-Unique Index

````javascript
db.employees.createIndex({empno: 1}) // 기본값: {unique: false}
````





## 3. Sparse

**검색 대상 필드가 전체에서 차지하는 비중이 낮은 경우**에 유리

(전체 대상 Full Collection 검색보다 해당 조건을 만족하는 Document 로 만 Sparse 인덱스 생성 & 검색하기 때문에 더 빠름)



아래와 같은 경우 prize 필드는 일부의 소수에게만 있으므로 sparse 인덱스로 구성하기 좋음



#### Example

* 90%의 일반적인 employees 필드: empno, ename, deptno
* 10%의 소수의 수상내역 있는 employess 필드: empno, ename, deptno, <font color="red">**prize**</font>

````javascript
db.employees.createIndex({prize: 1}, {sparse: true}) // prize 필드가 있는 doc만 인덱스 생성
````





## 4. Partial

특정 컬럼에서 **조건을 만족하는 데이터에서 추가로 조건을 또 만족하는 값을 검색**할 때 유리

즉, **조건을 만족하는 일부 doc 에만 인덱스 적용**



#### Example

* 1번 부서에서 연봉(salary)이 1억 넘는 사원들만 인덱스 적용

````javascript
db.employees.createIndex(
	{deptno: 1},
    {
        partialFilterExpression: {
        	sal: {$gt: 10000000}	// 1번 부서에서 연봉이 1억 넘는 사원들만 인덱스 생성
    	}
    }
)
````

<img src="https://user-images.githubusercontent.com/20942871/46904667-f3beb700-cf22-11e8-8bc5-4a1e9d446a2f.png" />





## 5. Background

인덱스 생성을 백그라운드에서 진행함

시스템 자원 부족한 환경 또는 빅 데이터 인덱싱의 상황에서 유용함



#### Example

````javascript
db.employees.createIndex({eno: 1}, {background: true})
````





## 6. Covered

여러 필드로 생성된 인덱스 검색 시 Index 만의 검색 만으로도 조건을 만족하는 doc을 추출하는 경우

인덱스 스캔의 경우 **인덱스를 통해 조건 만족하는 데이터 검색** 후, **컬렉션에 대한 추가 검색**을 통해 데이터 추출하는데, **인덱스 검색만으로 해결할 수 있는 경우** Covered 인덱스 생성하여 해당 인덱스 검색만으로 빠르게 처리 가능



#### Example

* employees 에서 _id는 제외하고 **부서번호와 이름만** 검색할 때

````javascript
db.employees.createIndex({deptno: 1, ename: 1}); // deptno와 ename에 인덱스
db.employees.find(
    {deptno: 3, ename: "Kim"}, // 검색 조건 = 3번 부서의 Kim 씨
    {_id: 0, ename: 1} // 노출 대상(실제 검색 대상)에서 _id 를 제외(값이 0)
);
````

* 실행계획 확인

  * Covered 인 경우

    ````javas
    db.employees.find({deptno: 3, ename: "Kim"}, {_id: 0, ename: 1}).explain() // Covered
    ````

    <img src="https://user-images.githubusercontent.com/20942871/46904952-7ba6c000-cf27-11e8-911d-ca4aaab645b0.png" width="50%"/>

  * Covered 가 아닌 경우

    ````javascript
    db.employees.find({deptno: 3, ename: "Kim"}).explain() // Not Covered
    ````

    <img src="https://user-images.githubusercontent.com/20942871/46904969-caecf080-cf27-11e8-9d42-8c2863e4caa9.png" width="50%"/>





## 7. TTL

일정 시점이 지난 인덱스 데이터 자동 삭제



#### Example

````javascript
db.employees.createIndex({"dailyWorkLog": 1}, {expireAfterSeconds: 3600})
````







## 8. GeoSpatial

좌표로 구성되는 2차원 구조로 하나의 컬렉션에 하나의 2D 인덱스 생성 가능.

내 위치 기준 반경 1Km 내 맛집 검색 등의 경우에 유용



#### Example

````javascript
db.spatial.ensureIndex({pos: "2d"}) // 2D 인덱스 생성 (안 하면 검색 시 에러)
````





### 연산자 종류

* **$near** : 해당 좌표를 기준으로 가장 가까이 있는 n 개의 좌표 검색 </br>
  <img src="https://user-images.githubusercontent.com/20942871/46913874-96813f00-cfd0-11e8-9725-60768da8c083.png" width="50%" />

  ````javascript
  db.spatial.find({
      pos: {
          $near: [5, 5]	// (5,5) 기준으로
      }
  }).limit(5)	// 가장 가까이 있는 5개 추출
  ````

* **$center** : 해당 좌표를 기준으로 가장 가까운 원형 좌표 검색 </br>
  <img src="https://user-images.githubusercontent.com/20942871/46913909-3a6aea80-cfd1-11e8-8a2e-02a7787521ff.png" width="50%" />

  ````javascript
  db.spatial.find({
      pos: {
          $geoWithin: {	// $within 은 2.4 버전에서 Deprecated 됨
              $center: [
                  [5, 5], 2	// (5, 5) 를 중점으로 하는 반지름 2인 원 내에 속하는 좌표 검색
              ]
          }
      }
  }, {_id: 0})
  ````

* **$box** : 해당 좌표를 기준으로 가장 가까운 Box 형 좌표를 검색 </br>
  <img src="https://user-images.githubusercontent.com/20942871/46913972-e3b1e080-cfd1-11e8-94e8-99b69f485b9a.png" width="50%" />

  ````javascript
  db.spatial.find({
      pos: {
          $geoWithin: {
              $box: [
                  [5, 5], [6, 6]	// (5, 5), (6, 6)을 끝점으로 하는 Box 내에 속하는 좌표 검색
              ]
          }
      }
  }, {_id: 0})
  ````

* **$polygon** : 해당 좌표를 기준으로 가장 가까운 다면체 좌표 검색 </br>
  <img src="https://user-images.githubusercontent.com/20942871/46914006-88342280-cfd2-11e8-9d6e-9878c2198ae1.png" width="50%" />

  ````javascript
  db.spatial.find({
      pos: {
          $geoWithin: {
              $polygon: [
                  [3, 4], [5, 7], [7, 4]	// 각 좌표를 끝점으로 하는 삼각형에 속하는 좌표 검색
              ]
          }
      }
  }, {_id: 0})
  ````

* **$centerSphere** : 중점을 기준으로 지정한 반경 내에 속하는 좌표(경도, 위도)를 반환.  </br>
  (centerSphere는 2차원 원이 아닌 3차원 구체 내에 속하는 값 검색)

  ````javascript
  db.tel_pos.find({
      last_pos: {
          $geoWithin: {
              $centerSphere : [
                  [경도, 위도], 반경(radius) // (경도, 위도)를 기준으로 radius 에 포함되는 좌표 검색
              ]
          }
      }
  })
  ````

* **$nearSphere** : 중점을 기준으로 가장 가까운 좌표(경도, 위도) n 개 반환

  ````javascript
  db.tel_pos.find({
      last_pos: {
          $nearSphere: [경도, 위도]	// (경도, 위도)를 기준으로
      }
  }).limit(2)	// 가장 가까이 있는 2개 좌표 반환
  ````




## 9. GeoMetry

직선 또는 곡선의 교차에 의해 이뤄지는 추상적인 구조나 다각형과 같은 기하학적 구조

3가지 타입을 제공하며, 인덱스 타입을 "2dsphere" 로 지정해야 한다.

````javascript
db.position.ensureIndex({loc: "2dsphere"}) // 2dsphere 로 지정 필수
````





### 타입 종류

* **Point Type** : 좌표와 같음

  ````javascript
  /* Point Type 좌표 생성 */
  db.position.insert({
      "loc": {
          "type": "Point",	// Point 타입
          "coordinates": [127.0980748, 37.5301218]	// (경도, 위도)
      },
      "name": [
          "name=동서울 터미널"
      ]
  })
  
  /* Point 좌표를 기준으로 반경 검색 */
  db.position.find({
      loc: {
          $near: {
              $geometry: {
                  type: "Point",	// Point 타입
                  coordinates: [127.1058431, 37.5164113]	// (경도, 위도)
              },
              $maxDistance: 2000,	// 미터 값
  	        $minDistance: 10 	// 미터 값
          }        
      }
  })
  ````

* **LineString Type** : 경로 등의 선분 위에 위치한 좌표를 검색할 때 사용

  ````javascript
  /* 경로 (선분)에 속하는 값 생성 */
  db.position.insert({
      loc: {
          type: "LineString",
          coordinates: [
              [127.1058431, 37.5164113],	// 잠실역
              [127.0846600, 37.5120906],	// 신천역
              [127.0740075, 37.5133497],	// 종합운동장역
              [127.0847829, 37.5105344]	// 삼성역
          ]
      },
      pos_name: [
          "start_name=SamSung Station",
          "route1=Complex Stadium Station",
          "route2=SinCheon Station",
          "end_name=JamSil Station"
      ]
  })
  
  /* 경로상에 있는 지하철 역 검색 */
  db.position.find({
      loc: {
          $geoIntersects: {
              $geometry: {
                  type: "LineString",
                  coordinates: [
                      [127.1058431, 37.5164113],
                      [127.0846600, 37.5120906],
                      [127.0740075, 37.5133497],
                      [127.0847829, 37.5105344]
                  ]
              }
          }
      }
  }).pretty()
  ````

* **Polygon Type** : 다면체 내에 속하는 좌표를 검색

  ````javascript
  /* 올림픽 공원 영역 저장 */
  db.position.insert({
      loc: {
          type: "Polygon",
          coordinates: [
              [127.1261076, 37.5191452], // 몽촌토성역
              [127.1220412, 37.5221428],
              [127.1224733, 37.5239739],
              [127.1269535, 37.5231093],
              [127.1290333, 37.5179105],
              [127.1239271, 37.5116750],
              [127.1261076, 37.5191452] // 몽촌토성역
          ]
      },
      pos_name: [
          "add_name=올림픽 공원",
          "add_type=공원"
      ]
  })
  
  /* 올림픽 공원 내 수영장 위치 저장 */
  db.position.insert({
      loc: {
          type: "Point",
          coordinates: [127.126553, 37.520952]
      },
      pos_name: [
          "add_name=올림픽 수영장",
          "add_tag=Public Sport"
      ]
  })
  
  
  /* 올림픽 공원 영역 내 어떤 시설물들이 있는지 검색 */
  db.position.find({
      loc: {
          $geoWithin: {
              $geometry: {
                  type: "Polygon",
                  coordinates: [
                      [
                          [127.1261076, 37.5191452], // 몽촌토성역
                          [127.1220412, 37.5221428],
                          [127.1224733, 37.5239739],
                          [127.1269535, 37.5231093],
                          [127.1290333, 37.5179105],
                          [127.1239271, 37.5116750],
                          [127.1261076, 37.5191452] // 몽촌토성역
                      ]
                  ]
              }
          }
      }
  }).pretty()
  ````
  
---

# 3-3. 사용자 생성과 관리

### 목차

1. OS 인증방식
2. DB 인증방식
3. 사용자 권한 룰



## 1. OS 인증방식

MongoDB 인스턴스 실행 시 `--bind_ip` 옵션을 추가하여 접속가능한 클라이언트 IP 를 지정



#### Example

````bash
mongod --bind_ip=127.0.0.1  # localhost 로만 접근 가능
````





## 2. DB 인증방식

Role 기능을 통해 사용자 계정이 DB에서 어떤 권한으로 어느 범위의 작업들을 수행할 수 있는지 제어

참고: [built-in-roles](https://docs.mongodb.com/manual/reference/built-in-roles/)



#### 1. admin DB 선택 & 유저 확인

````javascript
use admin	// admin 데이터베이스 선택
show users	// user 확인 -> 처음엔 아무런 유저도 없다.
````

<img src="https://user-images.githubusercontent.com/20942871/46949012-e76d6200-d0ba-11e8-9118-b948648d71e0.png" width="30%"/>



#### 2. 유저 & 권한 생성

````javascript
db.createUser({
	user: "system", // ID: system
    pwd: "1234", // PWD: 1234
    roles: [
        "dbAdminAnyDatabase"
    ]
})
````

<img src="https://user-images.githubusercontent.com/20942871/46949347-16d09e80-d0bc-11e8-8509-08028fb9ad3b.png" width="70%" />



````javascript
db.dropUser("system")
db = db.getSiblingDB('admin')
````

<img src="https://user-images.githubusercontent.com/20942871/46949421-5a2b0d00-d0bc-11e8-87ab-1772459424dc.png" width="50%" />



````javascript
db.createUser({
    user: "system",
    pwd: "manager",
    roles: [
        {role: "readWrite", db: "admin"},
        {role: "userAdmin", db: "admin"},
        {role: "dbAdmin", db: "admin"},
        {role: "clusterAdmin", db: "admin"},
        {role: "dbAdminAnyDatabase", db: "admin"},
        {role: "readWrite", db: "test"} // Test DB R/W 도 추가
    ]
})
````

<img src="https://user-images.githubusercontent.com/20942871/46949538-afffb500-d0bc-11e8-87e5-1ef7f87e14b4.png" width="50%" />



#### 3. 인증 적용 및 재시작

* mongod 실행 시 반드시 `--auth`를 붙여야 한다!

````javascript
/* 종료 후 재접속 */
mongod --bind_ip=0.0.0.0 --dbpath=/data/db --auth // 반드시 --auth 를 붙여야 함
use test
show collections
````

<img src="https://user-images.githubusercontent.com/20942871/46950296-36b59180-d0bf-11e8-9f8e-b146a768ae2b.png"/>



#### 4. 인증 방법

* 참고: [enable-authentication](https://docs.mongodb.com/manual/tutorial/enable-authentication/)

````javascript
/* db.auth를 통한 인증 */
use admin
db.auth("system", "manager")

/* mongo 클라이언트 실행 시 인증 */
mongo -u system -p manager admin // -u 유저 -p 비밀번호 
````

<img src="https://user-images.githubusercontent.com/20942871/46950736-b8f28580-d0c0-11e8-950a-d853e9a3a797.png"/>





## 3. 사용자 권한 롤

#### 1. Database User Role

가장 기초 권한으로 MongoDB 접속 및 해제, CRUD 작업 수행 가능



#### 2. DB Administration Role

MongoDB 를 기본적으로 관리할 수 있는 권한 부여



#### 3. Administrative Role

DB ADministraion Role 과  더불어 Shards 시스템과 Replication 시스템을 구축 및 관리할 수 있는 권한 부여



#### 4. Any Database Role

MongoDB 에서 제공하는 최상위 권한 부여



#### 참고: [built-in-roles](https://docs.mongodb.com/manual/reference/built-in-roles/)
