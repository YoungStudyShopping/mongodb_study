# ch7. Replica & ReplicaSet



## 7.1 Master & Slave 서버

- 데이터의 저장과 관리를 안전하게 하기위해선 백업솔루션이 필요함.

- replica와 replicaSet은 **백업을 통해 안정성을 보장하기위한 솔루션**

  ![](https://user-images.githubusercontent.com/8342133/29204563-6f800658-7e95-11e7-934b-76102411878e.png)






- Replica를 이용하기 위해서는 **master node**와 **slave node**가 필요하다.
- master 서버는 데이터가 입력, 수정, 삭제되면 동일한 구조의 slave 서버에 복제를 해놓고 장애가 나도 slave를 이용해 복구할수있다.
- master/slave 노드를 활성화한다

```bash
$ mongod --dbpath ./master --port 10000 --master #마스터
$ mongod --dbpath ./slave1 --port 10001 --slave --source localhost:10000 #슬레이브1
$ mongod --dbpath ./slave1 --port 10002 --slave --source localhost:10000 #슬레이브2
```





### 7.1.1 master & slave 서버 환경설정

- replica 기능은 기본적으로 최소 2개이상의 노드로 구성되야한다.
- 하나의 node에 샤드서버와 컨피그 서버를 구축할땐 별도의 port를 지정해야한다.



- replica가 정상적으로 동작하는지 테스트 한다.

1. 마스터에 컬랙션 생성

```bash
$ mongo localhost:10000
MongoDB shell version: 3.0.15
connecting to: localhost:10000/test
Welcome to the MongoDB shell.
> show dbs
local  0.328GB 
> use test
switched to db test
> db.things.insert({empno:1000, ename:'james', dept:'account'}) # 컬랙션 생성
WriteResult({ "nInserted" : 1 })
> 
> db.things.find();
{ "_id" : ObjectId("5be97ec635e0177a4a618c73"), "empno" : 1000, "ename" : "james", "dept" : "account" }
> exit
bye

```



2. 슬레이브1,2 접속

```bash
$ mongo localhost:10001
MongoDB shell version: 3.0.15
connecting to: localhost:10001/test
Server has startup warnings: 
> 
> rs.slaveOk() # 슬레이브 db를 read할수있도록 설정한다.
> show dbs
local  0.078GB
test   0.078GB # Replication 에의해 생성된 DB
> 
> use test
switched to db test
> db.things.find()
{ "_id" : ObjectId("5be97ec635e0177a4a618c73"), "empno" : 1000, "ename" : "james", "dept" : "account" }
> 
> exit
bye
```





### 7.1.2 master & slave 복구

```bash
1. 
$ mongo localhost:10000
> use admin
> db.shutdownServer() # 마스터를 내린다.


2. 
$ mongo localhost:10001
> rs.slaveOk() # 슬레이브에 읽기 설정한다.
> 
> db.things.insert({empno:1002, ename:'tete', dept:'information'}) # 컬랙션추가
WriteResult({ "writeError" : { "code" : undefined, "errmsg" : "not master" } })
>  # 슬레이브서버에는 입력불가
> use admin
switched to db admin
> db.shutdownServer() # 슬레이브를 내린다.

3.
$ rm -rf master/ # 마스터를 날린다.
$ cp slave1/* master/ # 슬레이브에 있는 내용을 마스터로 복사

4. 
$ mongod --dbpath ./master --port 10000 --master
$ mongod --dbpath ./slave1 --port 10001 --slave --source localhost:10000
# 마스터 슬레이브 재시작

5. 
$ mongo localhost:10000 # 마스터 접속
> show dbs
local  0.328GB
test   0.078GB 
>  db.things.find() # 마스터에서 생성했던 컬랙션 존재하는거 확인 (복구)
{ "_id" : ObjectId("5be97ec635e0177a4a618c73"), "empno" : 1000, "ename" : "james", "dept" : "account" }
```





## 7.2 ReplicaSets

위에서 살펴봤던 마스터 슬레이브는 실시간으로 복구작업을 수행할수없다.

이런 문제점을 보완한 기능이 **ReplicatSet**이다.



![](http://linux.systemv.pe.kr/wp-content/uploads/2016/06/replica-set-primary-with-secondary-and-arbiter.png)



- 메인 서버를 primary 서버라고 한다. 
- 사용자는 primary에서 입력, 수정, 삭제, 조회를 한다.
- 메인서버의 장애를 대비한 서버를 Secondary서버라고한다.
- 장애가 나면 primary의 마지막 작업부터 연속적으로 수행해준다.
- 이때부터 secondary가 primary가 되고 primary가 복구되면 secondary가 된다.
- hearybeat으로 매 2초마다 secondary의 상태를 체크한다.
- arbiter는 장애시 어떤 서버를 primary로 지정할지 투표하는 서버이다.



### 7.2.1 priority (우선순위)

- replica set으로 서버가 구성되면 장애가났을때 어떤 서버를 primary로 할지 vote가 필요하다.
- 이떄 **Arbiter**가 있으면 우선순위에 따라 primary가 된다.
- 우선순위는 0~1000까지있다.

```js
conf ={
 _id: "rptmongo",
 members: [
     {_id:0, host:"server_a:10001", priority:3},
     {_id:1, host:"server_b:10002", priority:2},
     {_id:2, host:"server_c:10004", priority:1},
     {_id:3, host:"server_d:10005", hidden:true, slaveDelay:1800},
 ]
}
```

- 마지막은 장애발생시 투표에는 참여하지 않고 복제데이터는 1800초 이후부터 저장되는 secondary서버이다.



### 7.2.3 멤버의 유형

- replica set을 구성하는 멤버의 유형



#### 1. secondary only member

- 데이터는 저장하고 있지만 primary 서버는 될수없는 멤버이다.

```js
cfg = rs.conf()
cfg.members[0].priority = 0
cfg.members[1].priority = 0.5
cfg.members[2].priority = 1
cfg.members[3].priority = 2
rs.reconfig(cfg)
```



#### 2. hidden member

- Hidden 멤버는 primary 데이터 셋을 복사해놓지만 클라이언트 애플리케이션에 보이지 않는다.
- 클라이언트에게 보이지않으므로 트래픽을 받지 않는다. 그러므로 Hidden members는 reporting이나 backups와 같은 헌신적인 일들을 하는데 주로 쓰인다.
- arbiter 서버가 프라이머리 서버를 선출하기위해 투표에는 참여한다.
- 우선순위는 항상 0이어야한다.

```js
cfg = rs.conf()
cfg.members[0].priority = 0
cfg.members[0].hidden = true

rs.reconfig(cfg)
```



#### 3. arbiter Member

- 사용자 데이터는 저장하지 않고 장애가났을때 primary를 선출하는데에만 사용한다.

```js
db.runCommand({
    "replSetInitiate":{
        "_id": "rptmongo",
        "members":[
            {_id:0, "host":"localhost:10001"},
            {_id:1, "host":"localhost:10002"},
            {_id:2, "host":"localhost:10003", arbiterOnly:true},
        ]
    }
})
```



### 7.2.3 replica set 환경설정 

```bash
$ mongod --dbpath . --port 10001 --replSet rptmongo/localhost:10002 # primary db
$ mongod --dbpath . --port 10002 --replSet rptmongo/localhost:10001 # secondary db
$ mongod --dbpath . --port 10003 --replSet rptmongo/localhost:10001 # secondary db for backup
```



```bash
> rs.add("192.168.0.10:10002") # ip address로 레플리카셋 설정
> rs.add("192.168.0.10:10003")

rptmongo:PRIMARY> rs.status()

{
	"set" : "rptmongo",
	"date" : ISODate("2018-11-12T14:50:37.115Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "choetaeeun-ui-MacBook-Pro.local:10001",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 1184,
			"optime" : Timestamp(1542033095, 1),
			"optimeDate" : ISODate("2018-11-12T14:31:35Z"),
			"electionTime" : Timestamp(1542033095, 2),
			"electionDate" : ISODate("2018-11-12T14:31:35Z"),
			"configVersion" : 1,
			"self" : true
		},
		{
			"_id" : 0,
			"name" : "choetaeeun-ui-MacBook-Pro.local:10002",
			"health" : 1,
			"state" : 1,
			"stateStr" : "SECONDARY",
			"uptime" : 1184,
			"optime" : Timestamp(1542033095, 1),
			"optimeDate" : ISODate("2018-11-12T14:31:35Z"),
			"electionTime" : Timestamp(1542033095, 2),
			"electionDate" : ISODate("2018-11-12T14:31:35Z"),
			"configVersion" : 1,
			"self" : true
		},
		{
			"_id" : 0,
			"name" : "choetaeeun-ui-MacBook-Pro.local:10003",
			"health" : 1,
			"state" : 1,
			"stateStr" : "ARBITER",
			"uptime" : 1184,
			"optime" : Timestamp(1542033095, 1),
			"optimeDate" : ISODate("2018-11-12T14:31:35Z"),
			"electionTime" : Timestamp(1542033095, 2),
			"electionDate" : ISODate("2018-11-12T14:31:35Z"),
			"configVersion" : 1,
			"self" : true
		}
	],
	"ok" : 1
}

```



### 7.2.4 Fail over

내부동작 참고 강대명님 블로그

https://charsyam.wordpress.com/2012/03/24/mongodb-%EC%86%8C%EC%8A%A4%EB%A1%9C-%EC%82%B4%ED%8E%B4%EB%B3%B8-replica-set-%EB%B0%A9%EC%8B%9D%EC%97%90%EC%84%9C%EC%9D%98-failover-1/