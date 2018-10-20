# Data Modeling

### 네임 스페이스

MongoDB에서는 데이터 베이스의 이름과 컬렉션의 이름 조합을 네임스페이스라고 한다. 네임 스페이스와 컬렉션의 이름 사이에는 `.`으로 구분해야 한다. 

컬렉션 뿐만 아니라 인덱스도 자신만의 네임스페이스를 가질 수 있으며,  인덱스의 네임 스페이스는 컬렉션의 네임스페이스에 추가로 인덱스 이름을 붙인 형태이다. 

예시) 
shop.user.$ix_catalog_nvMid

MMAPv1 스토리지 엔진에서는 데이터베이스 별로 네임스페이스의 목록을 저장하는 네임 스페이스 파일이 생성 되는데, 이 파일의 최대 크기는 2GB이다. 
각 네임 스페이스는 120바이트를 넘어 설 수 없으며, 데이터 베이스 이름과 컬렉션 이름의 합이 120바이트 이상의 문자열로 생성 될 수 없다. 
기본으로 생성 되는 네임 스페이스 파일 크기는 16MB이다. 

WiredTiger 스토리지 엔진에는 별도 네임 스페이스 파일을 사용하지 않아서 제약이 없다. 

### 데이터 베이스 

DB는 주로 서비스나 데이터의 그룹을 만들기 위해서 사용하는 물리적인 개념이다. 몽고의 초창기 버전에서는 하나의 도큐먼트를 변경하기 위해서 MongoDB의 인스턴스 잠금을 사용했었다.  
버전 업을 해가면서 잠금의 영역이 줄어들었다. 

MongoDB 2.8부터 WiredTiger 스토리지 엔진이 도입 되면서, 새로운 변화가 생겨났고, 기존의 엔진은 컬렉션 수준의 잠금을 지원 하나, WiredTiger는 도큐먼트 수준의 잠금을 제공한다. 

### 컬렉션 

RDBMS에서 주로 테이블이라고 부르는 것을 MongoDB에서는 컬렉션이라고 부른다. 더불어서 3.2부터 Join을 지원 하기는 하지만, 실질적인 join이 아니다. 고로, 컬렉션은 가능하면 많은 데이터를 내장 하는 것을 권장한다. 

하지만 이는 모델링 측면에서나 맞는 이야기일 수 있다. 성능적인 측면에서는 어떤 문제가 발생할까? 

MMAPv1 스토리지 엔진을 사용하면, Mongodb는 쓰기 작업을 위해서 데이터베이스 단위의 잠금이나 컬렉션 단위의 잠금을 사용한다. 그래서 단일 컬렉션에 쓰기 작업이 많을 경우에는 컬렉션을 분리하는 것이 좋다. 
다만, WiredTiger나 RocksDB 스토리지 엔진을 사용한다면, 데이터 베이스나 컬렉션을 물리적으로 분리할 필요는 없다. 

MongoDB는 내부적으로 샤딩 기능을 가지고 있기 때문에 컬렉션 단위의 샤딩은 별도로 크게 고민 하지 않아도 된다. 

MongoDB도 내부적으로는  MySQL과 동일한 형태의 스토리지 엔진을 사용한다. WiredTiger는 실제 트랜잭션을 지원하는  B-Tree 기반의 스토리지 엔진이기 때문에 결론적으로 MySQL 서버의 InnoDB스토리지 엔진이나 다른 RDBMS의 작동방식과 유사한 최적화가 MongoDB에서도 도움이 되는 경우가 많다. 

- 하나의 컬렉션에 저장 되는 도큐먼트들의 Access Pattern이 많이 다를 때에는 컬렉션을 물리적으로 분리 하는 것이 좋다.
- 자주 읽히는 데이터 위주로 메모리 캐시를 활용하도록 하면 성능상 이점을 얻을 수 있다. 
- 컬렉션 단위의 샤딩을 지원 하기에 컬렉션을 너무 잘게 쪼개어도 성능상 이점을 얻을 수는 없다.
- WiredTiger 엔진을 사용시 컬렉션에서는 청크 이동이 발생 할 시점에 디스크 사용이 늘 수 있다. 
- 컬렉션의 설계에서 가장 주요한 것은 샤드 키의 선정이다. 

### 제한 사항 

- 하나의 도큐먼트는 반드시 `{` `}`로 시작하고 종료 되어야 한다. 
- 도큐먼트의 모든 원소는 반드시 키와 값의 쌍으로 구성 되어야 함.
- 중첩된 도큐먼트의 깊이는 100 레벨 까지 지원한다.
- 도큐먼트의 전체 크기는 16MB까지만 지원한다.

------

### Data Model

1. 개념적 데이터 모델링: 비즈니스 영역으로부터 데이터를 수집하는 단계
2. 논리적 데이터 모델링: Document 구조에 맞게 분석, 설계하는 단계
3. 물리적 데이터 모델링: MongoDB 물리적 구조에 맞게 설계하는 단계

개념적 DB 모델링 

- Collection 추출
- Field 추출
- Document & Tree Document 추출
- Link/Embeded 추출
- Collection Diagram 

논리적 DB 모델링 

- Data Type의 결정
- Validator 설계

 물리적 DB 모델링 

- Database 설계
- 사용자 계정 설계
- Collection 타입/크기 설계
- Index 타입/크기 설계
- DB storage-Engine & HW Spec 설계
- Sharding & Replicaion 설계



## 4.2 설계 주요 특징

File system - DBMS - NoSQL로 이동하는 수순을 볼 수 있는데, 우선 File system은 process의 중심으로 흘러 갔다. 그리고 DBMS를 주로 많이 사용하던 C-S 구조에서 Data중심의 설계를 지향했다. 그런데, NoSQL의 환경은 Data와 Process 두 가지의 초점에 맞춰서 설계 해야 한다. 



### 4.2.2 Rich Document Structure

관계형 DB는 정규화를 통해 데이터 중복을 제거 하며 무결성을 보장하는 설계 기법을 지향하지만, NoSQL은 데이터의 중복을 허용하며 역정규화된 설계를 지향한다. 

### 4.2.3 중첩 구조

관계형 DB는 Entity간의 Relationship을 중심으로 데이터의 무결성을 보장하지만, 불필요한 join을 유발시킴으로써 코딩양을 증가하게 하고, 검색 성능을 저하시키는 원인이 된다. 

NoSQL은 도큐먼트 내에 도큐먼트를 가질 수 있는 중첩 구조를 허용함으로 join을 줄일 수 있다. 결론적으로 Relationship을 통한 데이터 무결성을 보장하지 않는다. 

### 4.2.4 Non-Scheme structure

기본적으로 MongoDB는 schemeless로 스키마 설계를 하지 않는다. RDBMS에서는 하나의 테이블을 생성하면, 테이블명은 사용자명과 함께 결합되어 생성된다. MongoDB의 경우 사용자 계정은 오로지 인증의 의미만 가지고 있다. 

### 4.2.5 비정형 데이터 구조 

사진, 동영상, 음성 파일과 같은 바이너리를 RDBMS구조에서는 파일 시스템으로 지니고, 해당 파일 시스템의 경로만 가지고 있기 마련이다. 그런데, MongDB와 같은 NoSQL 기술은 비정형 데이터들을 보다 쉽고, 간편하게 저장, 관리 할 수 있는 기능들을 제공한다. 

**HOW?에 대한 부분이 누락됨.**

### 4.2.6 유연한 서버 구조

RDBMS는 서버에 설치 할 때, DBMS만을 위한 전용 메모리 영역을 할당 해준다. 이 영역은 시스템 메모리의 일부를 할당 받아 제한된 크기만 사용할 수 있다. 

**HOW?에 대한 부분이 누락됨.**

### 4.3 MongDB 설계 기준

1. 데이터 발생량과 Access pattern 분석은 되었나?
   - Data양과 보관 기한은 어떤지?
2. 컬렉션과 인덱스에 대한 충분한 설계가 고려 되었나?
   - 적절한 Collection Type을 결정했나?
   - Field, Data Type, Validator는 결정 되었나?
   - 저장할 Document의 크기는 ?
   - Link & Embeded Document를 적절히 설계 했나?
   - Secondary Indexes의 사용여부?
   - 적절한 Index Type을 선택했는가?
3. 적절한 시스템 환경은 준비 되었나?
   - 데이터가 발생하는 비즈니스 환경에 맞는 적절한 스토리지 엔진을 선택했나?
   - 해당 Collection의 sharding 여부는 결정 되었나?
   - 효율적인 처리를 위한 적절한 시스템 환경은 준비 되었나?

### Embeded Document(Rich Document)

Join을 쓰지 않고, 빠른 성능의 쓰기와 읽기가 가능한데, 이것을 가능하게 해주는 설계 구조를 Rich Document 구조라고 한다. 

```json
{ 
	order_id: "2018-10-091122",
	customer_name: "seung ho",
	emp_name: "miji",
	total_price: "600000",
	payment: "Credit card",
	order_filled: "Y",
	item_id: [
	{
		item_id: "1",
		product_name: "Pizza",
		item_price: "55550",
		qty: "1"
	},
	{
		item_id: "2",
		product_name: "Milk Pizza",
		item_price: "50000",
		qty: "2"
	}
	]
}

```

실습) ord collection에 데이터 넣기 

### Extent Document

앞에서 저장한 방식과는 다르게 확장형 도큐먼트는 주문 정보를 먼저 Collection에 저장 한 뒤에 추가로 주문 상세 정보를 UPDATE문으로 추가 시키는 방법이다. 
결론적으로는 동일한 데이터 구조를 가지게 된다. 

Rich Document의 장점 

1. Query가 단순해지고, Join문을 실행할 필요가 없어 도큐먼트 단위의 데이터 저장에 효과적이며, 빠른 성능이 보장된다. 
2. 데이터 보안에 효과적이다. 

주문 정보 

```json
{
	order_id: "2018-10-091122",
	customer_name: "seung ho",
	emp_name: "miji",
	total_price: "600000",
	payment: "Credit card",
	order_filled: "Y"
}
```

주문 상세 정보 데이터 

```json
{
order_id : "2018-10-091122",
item_id: [
	{
		item_id: "1",
		product_name: "Pizza",
		item_price: "55550",
		qty: "1"
	},
		{
		item_id: "2",
		product_name: "Milk Pizza",
		item_price: "50000",
		qty: "2"
	}
	]
}
```

실습하기) 

1. ord collection에 주문 정보 데이터 넣기
2. update문을 이용해 상세 주문 정보를 주문 정보 데이터 내에 넣기 

Rich Document의 단점

1. Embeded 되는 도큐먼트의 크기는 최대 16MB 범위에서만 가능하다. 
2. Embeded 도큐먼트를 저장하기 위해서는 기존에 이미 Collection이 있거나, 같이 저장 해야 하는 구조에서만 쓸모가 있다. 

### LINK

관계형 데이터베이스처럼 MongoDB에도 유사한 관계를 지정하는 것이 가능한데,  ObjectId를 이용하여, 두 정보를 연결 지을 수있다. 

#### 수동 LINK

```bash
> db.ord.insert({
... order_id: "2018-10-091122",
... customer_name: "seung ho",
... emp_name: "miji",
... total_price: "600000",
... payment: "Credit card",
... order_filled: "Y"
... })
> o = db.ord.findOne( {"order_id" : "2018-10-091122" })
{
	"_id" : ObjectId("5bc7207501d63feedf2eabfe"),
	"order_id" : "2018-10-091122",
	"customer_name" : "seung ho",
	"emp_name" : "miji",
	"total_price" : "600000",
	"payment" : "Credit card",
	"order_filled" : "Y"
}
> db.ord_detail.insert({ order_id : "2018-10-091122",
... item_id: [
... {
... item_id: "1",
... product_name: "Pizza",
... item_price: "55550",
... qty: "1"
... },
... {
... item_id: "2",
... product_name: "Milk Pizza",
... item_price: "50000",
... qty: "2"
... }
... ],
... ord_id: ObjectId("5bc7207501d63feedf2eabfe") 
... })
> db.ord_detail.findOne({ord_id: o._id})
{
	"_id" : ObjectId("5bc721a501d63feedf2eabff"),
	"order_id" : "2018-10-091122",
	"item_id" : [
		{
			"item_id" : "1",
			"product_name" : "Pizza",
			"item_price" : "55550",
			"qty" : "1"
		},
		{
			"item_id" : "2",
			"product_name" : "Milk Pizza",
			"item_price" : "50000",
			"qty" : "2"
		}
	],
	"ord_id" : ObjectId("5bc7207501d63feedf2eabfe")
}
```

#### DBRef 함수를 이용한 LINK

```bash
> x = {
... order_id: "2018-10-091122",
... customer_name: "seung ho",
... emp_name: "miji",
... total_price: "600000",
... payment: "Credit card",
... order_filled: "Y"
... }
{
	"order_id" : "2018-10-091122",
	"customer_name" : "seung ho",
	"emp_name" : "miji",
	"total_price" : "600000",
	"payment" : "Credit card",
	"order_filled" : "Y"
}
> db.order.save(x)
WriteResult({ "nInserted" : 1 })
> db.order_detail.save({
... order_id : "2018-10-091122",
... item_id: [
... {
... item_id: "1",
... product_name: "Pizza",
... item_price: "55550",
... qty: "1"
... },
... {
... item_id: "2",
... product_name: "Milk Pizza",
... item_price: "50000",
... qty: "2"
... }
... ],
... ref_orderId: [new DBRef ("ord", x._id)]
... })
> db.order_detail.find().pretty()
{
	"_id" : ObjectId("5bc7231301d63feedf2eac01"),
	"order_id" : "2018-10-091122",
	"item_id" : [
		{
			"item_id" : "1",
			"product_name" : "Pizza",
			"item_price" : "55550",
			"qty" : "1"
		},
		{
			"item_id" : "2",
			"product_name" : "Milk Pizza",
			"item_price" : "50000",
			"qty" : "2"
		}
	],
	"ref_orderId" : [
		DBRef("ord", ObjectId("5bc722be01d63feedf2eac00"))
	]
}
```

LINK의 장점 

1. 별도의 논리적 구조로 저장되기 때문에 도큐먼트 크기에 제한 받지 않는다.
2. 비지니스 룰 상 별도로 처리되는 데이터 구조에 적합하다

LINK의 단점

1. 매번 논리적 구조간 Link를 해야 하기에 Embeded보다 성능이 떨어진다. 
2. 컬렉션 개수가 증가하며, 관리 비용이 증가한다.

------

### 일반 필드 도큐먼트 vs 서브 도큐먼트 필드 

```json
{
	_id: 1,
	user_name: 'seungdols',
	user_real_name: 'seungho choi',
	contact_office_number: '010-xxxx-xxxx',
	contact_house_number: '070-777-7777',
	address_zip_code: "00000",
	address_detail: "ABC Mart",
	address_city: "Seoul",
	address_country: "Korea"
}
```

```json
{
	_id: 1,
	user: {
		name: 'Eni',
		real_name: 'Eniii'
	},
	contact: {
		office_number: '010-xxxx-xxxx',
		house_number: '070-777-7777'
	},
	address: {
		zip_code: "00000",
		detail: "ABC Mart",
		city: "Seoul",
		country: "Korea"
	}
}
```

실제로 두 방법 모두 인덱스 생성이나, 검색에서 기능적인 차이는 없다. 다만, 성능 차이는 존재한다. 

### 도큐먼트의 크기 증가 

- 도큐먼트 내의 배열 사용에 있어서 배열의 아이템이 제한 없이 증가 할 수 있는 구조라면, 배열을 쓰지 말것.
- 도큐먼트의 크기가 계속 증가할 경우, 엔진 마다 미치는 영향이 다르다. 
  - MMAVPv1 스토리지 엔진을 사용하면, 도큐먼트의 크기가 증가하고, 그때마다 해당 도큐먼트를 새로운 디스크 공간으로 이동시켜야 할 수도 있다. 이러면, 도큐먼트를 가리키는 인덱스의 주소 정보를 모두 변경해야 하는데, 이는 매우 치명적인 성능 저하를 유발 할 수 있다. 
  - WiredTiger를 사용할때, 가져온 도큐먼트 데이터 보다 읽을 데이터가 워낙 적은 구조를 유지 한다면, 리소스 낭비가 심해진다. 
  - 더군다나, 해당 엔진은 트랜잭션을 처리 할 수 있는 구조이기에, WAL로그, Undo로그도 가지고 있다. 그래서 새로운 도큐먼트를 읽을 때마다, 메모리를 할당 받아 변경된 버전을 저장 하는데, 계속 변경 될 경우 변경 히스토리를 위한 메모리 공간의 낭비가 심해진다. 
  - 그렇게 되면, 변경 히스토리를 병합하는 작업인 Reconciliation을 수행해야 한다. 

### Join 

3.2버전부터 나왔고, 엔터프라이즈에만 추가 한다고 했다가 몰매 맞고, 무료 버전에도 추가하여 배포 되었다. 
조인의 기능을 하는 것은 `$lookup`이며, Aggregation 기능의 일부로 제공되고 있다. 

```bash
> db.orders.insert({
... order_id: "999",
... order_user: "nari",
... product_id: "123",
... order_date: "2018-10-17"
... })
WriteResult({ "nInserted" : 1 })
> db.products.insert({
... product_id: "123",
... product_name: "Computer",
... price: "1200000"
... })
WriteResult({ "nInserted" : 1 })
> db.orders.aggregate([
... {
... $lookup:
... {
... from: "products",
... localField: "product_id",
... foreignField: "product_id",
... as: "order_product"
... }
... }
... ])
```

```json
{ "_id" : ObjectId("5bc7369901d63feedf2eac02"), "order_id" : "999", "order_user" : "nari", "product_id" : "123", "order_date" : "2018-10-17", "order_product" : [ { "_id" : ObjectId("5bc736c301d63feedf2eac03"), "product_id" : "123", "product_name" : "Computer", "price" : "1200000" } ] }
```

MongoDB의 Aggregation은 여러 개의 스테이지를 가질 수 있기 때문에 `$lookup` 스테이지를 여러번 나열하여 여러 컬렉션을 한 번에 조인 할 수 있다. 

#### Join의 제약 

- INNER JOIN은 지원하지 않으며, OUTER JOIN만 지원한다. 
  - MongoDB의 Aggregation에 필터링 스테이지를 추가해서 드리븐 컬렉션에서 일치하는 도큐먼트를 찾지 못한 경우에 회피할 수 있다. 
- 조인되는 대상 컬렉션은 같은 데이터베이스에 있어야 한다.
  - 컬렉션 모델링 시점에 충분히 회피가 가능하다.
- 샤딩되지 않은 컬렉션만 `$lookup` 오퍼레이션을 사용할 수 있다. 
  - 몽고 라우터에서 처리 되는 것이 아니라  MongoDB 샤드 서버 단위로 처리 되기 때문이다. 이 제약 사항은 같은 샤드 키로 샤딩된 컬렉션이라 해도 피할 수 없다. 같은 샤드키로 샤딩 됐다고 하더라도 컬렉션이 다르면, 청크가 서로 다르게 분산 되기 때문이다. 
