



# MongoDB - 백업 및 복구

<br>

## mongodump와 mongorestore를 이용한 논리 백업 및 복구

### 백업

- **mongodump는 MongoDB에서 공식적으로 제공되는 유일한 백업 도구**이다.
- mongodump는 MongoDB 서버의 데이터 파일을 물리적으로 복사하는 것이 아니라 **MongoDB 서버에 로그인한 다음 도큐먼트를 한건 한건씩 덤프해서 BSON 파일로 저장하는 방식**이다.
- mongodump는 백업 시간 뿐만아니라 복구 시간도 상당히 오래 걸린다.
- 서비스 요건상 **백업 시간이나 복구 시간이 문제되지 않는다면 mongodump는 훌륭한 백업 도구**이다.
- 기본적으로  mongodump가 데이터를 덤프하는 동안 변경되는 데이터에 대해서는 백업이 일관된 상태를 유지하지 못한다.
- **--oplog 옵션을 이용해서 mongodump 명령을 실행해야만 덤프가 실행 중인 동안 변경되는 데이터의 OpLog 이벤트를 같이 백업**할 수 있다.

<br>

```
mongodump --authenticationDatabase admin --oplog --out ./data/backup
```

백업이 실행되면 mongodump는 **"--out" 파라미터에 지정한 디렉터리로 덤프한 BSON 도큐먼트를 기록**한다.

<br>

```tex
[irteam@dev-junghyuck2-ncl testMongoDB]$ docker exec -it mongoDB mongodump --authenticationDatabase admin --out ./data/backup
2018-11-19T20:09:26.933+0000	writing admin.system.version to
2018-11-19T20:09:26.934+0000	done dumping admin.system.version (1 document)
2018-11-19T20:09:26.934+0000	writing test.thing to
2018-11-19T20:09:26.935+0000	writing test.foo to
2018-11-19T20:09:26.935+0000	writing SALES.employees to
2018-11-19T20:09:26.935+0000	writing test.emp to
2018-11-19T20:09:26.936+0000	done dumping test.thing (2 documents)
2018-11-19T20:09:26.937+0000	done dumping test.foo (1 document)
2018-11-19T20:09:26.940+0000	done dumping test.emp (0 documents)
2018-11-19T20:09:26.940+0000	done dumping SALES.employees (0 documents)
```

> --oplog 옵션은 당연히 해당 MongoDB 서버의 OpLog가 활성화되있어야만 스냅샷을 백업할 수 있다.


<br>


### 복구

- **mongorestore를 이용해서 위에서 만든 백업용 BSON 파일들을 읽어들여 데이터베이스를 복구할 수 있다.**
- "--oplogReplay" 옵션은 덤프된 데이터 파일을 모두 적재한 후에 백업 디렉터리의 "oplog.bson" 파일을 적재하도록 해주는 옵션이다.
- "oplog.bson" 파일은 --oplog 옵션을 사용해서 mongodump를 실행한 경우에만 생성되는 파일이다. 그래서 --oplog 옵션 없이 백업을 했다면 "--oplogReplay" 옵션을 사용하면 안된다.

```
mongorestore --oplogReplay ./data/backup
```

<br>

> "--nsInclude"나 "--nsExclude" 옵션을 이용하여 mongodump로 백업된 데이터로부터 일부 데이터베이스나 컬렉션만 적재할 수 있다.
>
> "--nsFrom"와 "--nsTo" 옵션을 활용하여 백업 디렉터리에서 특정 데이터베이스의 데이터를 다른 데이터베이스나 컬렉션으로 적재할 수 있다.
>
> "--nsParallelCollections", "--numInsertionWorkersPerCollection" 옵션으로 쓰레드 수를 조절하여 mongorestore 적재를 더 빠르게 실행할 수도 있다.

<br>



## 물리 백업 및 복구

MongoDB에서는 공식적으로 물리 백업이나 복구에 대한 도구는 제공하지 않고 있다.

mongodump와 mongorestore와 같은 논리적인 백업 및 복구 방식보다 **물리적으로 데이터를 백업하는 방법으로 빠르게 데이터를 백업하고 복구**할 수 있다.

<br>

### 셧다운 상태의 백업

서비스에 투입되지 않은 세컨드리 멤버의 MongoDB 서버를 셧다운하고 데이터 파일을 복사하는 방법

```
// MongoDB 서버 종료
systemctl stop mongod.service

// MongoDB 서버의 데이터 디렉터리를 백업 디렉터리로 복사
cp -r /mongodb_data /data/backup
```

<br>

이렇게 세컨드리 멤버를 셧다운하는 경우 **MongoDB 레플리카 셋의 고가용성을 고려**해야 한다. 즉, 백업 위해 세컨드리 멤버가 셧다운 상태이므로 백업을 실행하는 동안 다른 멤버에서 장애가 발생했을 때 새로운 프라이머리 멤버가 선출되지 못하면서 서비스 할 수 없는 상태가 될 수 있는 것이다.

<br>

### 복제 중지 상태의 백업

db.fsyncLock() 명령을 이용해서 데이터 파일을 동기화하고, 글로벌 잠금을 거는 방법을 복제 동기화를 멈추는 방법으로 사용할 수 있다.

```
// 글로벌 잠금 확득(복제 쓰기 멈춤)
db.fsyncLock({fsync: 1, lock: true})

// MongoDB 서버의 데이터 디렉터리를 백업 디렉터리로 복사
cp -r /mongodb_data /data/backup

// 데이터 디렉터리 복사가 완료되면 글로벌 잠금 해제
db.fsyncUnlock()
```

> db.fsyncLock() 명령을 사용하는 경우에는 데이터 디렉터리의 복사가 완료될 때까지 db.fsyncLock() 명령을 실행했던 커넥션을 닫지 않고 유지해야 한다.

위의 "셧다운 상태의 백업" 처럼 이 백업 방법도 **MongoDB 레플리카 셋의 고가용성을 고려**해야 한다.

<br>

### Percona 온라인 백업

위에 언급했던 "셧다운 상태의 백업", "복제 중지 상태의 백업" 방법은 고가용성에 영향을 미친다.

Percona에서는 이런 번거로움과 어려움을 해결하기 위해서 **MongoDB서버에 온라인 백업기능을 구현**했다.

> Percona는 MongoDB, MySQL 등에 대한 오픈 소스 데이터베이스 솔루션을 제공 및 기술 지원을 하는 업체. 
>
> https://www.percona.com/

온라인 백업기능을 사용하려면 Percona에서 배포하는 "Percona Server for MongoDB" 배포판을 사용해야 한다.

```
> db.runCommand({createBackup: 1, backupDir: "/data/backup"})
{ "ok" : 1 }
```

<br>



### 물리 백업 복구

- 복구도 간단하다. **복구하고자 하는 MongoDB 서버의 데이터 디렉터리에 백업된 데이터 파일을 그대로 복사**하기만 하면 된다.
- 만약 **샤딩된 MongoDB 클러스터에서 백업된 데이터 파일을 이용해서 복구하는 경우**에는 백업된 데이터 파일을 사용해서 시작되는 MongoDB 서버가 아무런 응답도 없이 **무한정 대기 상태로 빠지는 경우**도 있다. **원본 MongoDB 서버가 소속된 클러스터의 구조를 확인하기 위해서 컨피그 서버로 접속하려고 하기 때문**
  ==> **기존 샤드 클러스터와 무관하게 백업된 데이터 파일을 이용해서 복구를 하기 위해서는** MongoDB 서버의 시작 옵션에서 **recoverShardingState 옵션을 false 로 설정**하면 기존 샤드 클러스터의 컨피그 서버 정보를 무시하고 MongoDB 서버를 시작하게 된다. 

> recoverShardingState 옵션은 MongoDB 3.4 버전에서 사용가능, 3.6부터는 MongoDB 서버 시작시 레플리케이션 관련 옵션을 모두 제거하고 실행하면 recoverShardingState 옵션을 false 로 설정한 것과 동일한 효과를 얻을 수 있다.

<br>

```
// MongoDB 서버 시작 시 옵션 사용
mongod --setParameter=recoverShardingState=false -f mongod.conf

or

// MongoDB 설정파일에 옵션 적용
...
setParameter:
  recoverShardingState: false
...
```

<br>

### PIT(Point-In-Time) 복구

사용자나 운영자의 실수로 데이터가 삭제되거나 하드웨어 폴트로 인해 데이터를 잃게 된 경우 복구를 하려면 백업된 데이터가 필수이다.

**잘못 실행된 명령 직전까지, 즉 특정 시점까지 데이터를 복구하는 것을 PIT(Point-In-Time) 복구**라고 한다.

MongoDB에서 PIT 복구는 물리 또는 논리 풀 백업( MongoDB 덤프)에 최근의 OpLog 를 복구하는 방식으로 진행된다.

<br>

1. 물리 백업 복구
2. 복구된 백업 시점부터 가장 최근 시점까지의  OpLog 백업(가용한 레플리카 셋 멤버로부터)
3. 실수로 실행된 명령의  OpLog 이벤트의 타임스탬프 위치 확인
4. 물리 백업 복구된 서버에 백업 시점부터 실수로 실행된 명령 직전까지의 OpLog 적용

<br>

**2번 OpLog 백업**은 레플리카 셋 중에서 **마지막 백업 시점부터의 OpLog를 가진 멤버가 있어야만 가능**하다.

> 아래 그림과 같이 백업을 2016-10-28 04시에 하고(백업-1), 2016-10-30 04시에(백업-2) 했다고 한 상황에서
>
> 현재 레플리카 셋의 모든 멤버들이 2016-10-29 10시 이후부터의 OpLog를가진 상태라면.....
>
> 백업-1은 OpLog 동기화할 수 없고, 백업-2는 OpLog 동기화할 수 있다.

![test](https://user-images.githubusercontent.com/10356931/48893825-93595880-ee84-11e8-8940-4d87295f6211.jpeg)

<br>

**OpLog의 용량을 크게 해서 OpLog의 보관 주기를 길게 해두면 데이터 복구의 가능성이나 백업 주기를 길게 설정할 수 있다.**

<br>

**복구한 백업의  OpLog 시점 확인하기 위해 local 데이터 베이스의 oplog.rs 컬렉션의 가장 마지막 이벤트를 확인**한다.

OpLog의 이벤트별로 시점은 "ts" 필드에 저장된  Timestamp 값을 기준으로 한다.

```
> db.oplog.rs.find({}, {ts: 1,}).sort({ts: -1}).limit(1)
{"ts" : Timestamp(1459850401, 11)}
```

<br>

이제 **가장 최근까지의 OpLog를 가진 다른 MongoDB 서버에서 OpLog를 덤프**하면 된다.

```
> mongodbdump --db local --collection oplog.rs \
--query '{"ts": { "$gt": { "$timemstamp": {"t": 1459850401, "i": 11}}}}' \
--out backup_oplog
```

> OpLog는 Timestamp(1459850401, 11) 이전부터 덤프해도 무관하다.

<br>

이제 **덤프된 OpLog에서 실수로 데이터를 삭제한 이벤트의 시점을 찾아야 한다.**	

덤프된 OpLog는 BSON 파일이므로 확인을 위해 OpLog 덤프 파일을 JSON 파일로 변환한다.

```
bsondump backup_oplog/local/oplog.rs.bson > oplog.json
```

<br>

OpLog는 사용자가 실행한 명령을 로깅하는 것이 아니라 변경된 데이터의 결과를 저장한다.

그래서 db.users.remove({name:"matt"}) 와 같은 명령으로 삭제했을 때, **OpLog는 이 삭제 명령을 삭제된 도큐먼트의 프라이머리 키를 기록**한다. 그래서 name필드가 "matt"인 사용자가 10건 있었다면 10건의 도큐먼트를 기록하게 된다.

이 점을 인지하고 **삭제된 시점을 JSON 파일로 변환된 OpLog 덤프 파일을 확인하고 그 시점의 Timestamp을 확인**한다.

<br>

**Timestamp(1459852199,1) --> 삭제된 시점의 Timestamp**

<br>

--oplogLimit 옵션을 이용해서 이 시점의 명령은 재생되지 않도록 한다. 

아래처럼 **"타임스탬프:시퀀스" 형식으로 타임스탬프 지점을 지정하면 해당 이밴트 지점 직전까지 OpLog를 재생하고 mongorestore 명령이 멈춘다.**

```
// mongorestore 명령이 OpLog 덤프 파일을 인식할 수 있도록 디렉터리 경로와 파일명 변경
> mv backup_oplog/local/oplog.rs.bson backup_oplog/oplog/bson backup_oplog/oplog.bson

// mongorestore 명령을 이용해서 OpLog 이벤트 재생
> mongorestore --oplogReplay --oplogLimit 1459852199:1 backup_oplog
```

<br>





