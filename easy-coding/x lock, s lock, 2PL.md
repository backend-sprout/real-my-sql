# 투페이즈락(2PL)

DB에 값을 넣는 것은 단순한 작업이 아니다.  
만약 값을 바꾸는 대상의 컬럼이 인덱스에 잡혀있다면, 
리인덱싱 과정과 실제 물리적인 위치에 저장된 데이터의 파일을 바꿔야 하는등의 작업이 이루어진다.

# Write Lock
> Exclusive Lock

* read/wrtie(insert/update/delete) 할때 사용되는 락이다.
* 다른 tx(transaction)이 같은 데이터를 read/write 하는 것을 허용하지 않는다.


# Read Lock
> Shared Lock

* read 할때 사용한다.
* 다른 tx가 같은 데이터를 read 하는 것은 허용한다.
* 단, 다른 tx가 같은 데이터를 write 하는 것은 허용하지 않는다.


## Lock 을 써도 생기는 이상 현상
### 상황
* TransactionA : x와 y의 합을 x에 저장
* TransactionB : x와 y의 합을 y에 저장 
* Database : 초기값 x = 100, y = 200 
* 예상되는 결과 값 :
	* TransactionA 먼저 실행 : x = 300, y = 500
	* TransactionB 먼저 실행 : x = 400, y = 300
* Isolation Level : NonSerializable(Serializable 이 아닌 Isolation level)	

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 100, y = 200

Database -> TransactionB : read_lock(x)
TransactionB -> Database : read(x)
TransactionB -> Database : unlock(x)

Database -> TransactionA : read_lock(y)
Database -> TransactionB : write_lock(y)
Database <--> TransactionB : [ TransactionB write_lock(y) is block ]

TransactionA -> Database : read(y) = 200
TransactionA -> Database : unlock(y)
Database <--> TransactionB : [ TransactionB write_lock(y) is start ]

TransactionB -> Database : read(y)
TransactionB -> Database : write(y = x + y)
TransactionB -> Database : unlock(y)

Database -> TransactionA : write_lock(x)
TransactionA -> Database : read(x)
TransactionA -> Database : write(x = x + y)
TransactionA -> Database : unlock(x)

Database -> Database : Result : x = 300, y = 300
```

* TransactionA와 TransactionB 모두 서로 read_lock으로 얻은 값을 기준으로 작업을 수행한다.
	* TransactionA : y = 200
	* TransactionB : x = 100
* 위 과정대로 거치면 트랜잭션 종료시 값은 아래와 같다.
	* TransactionA : 
		* 200(read_lock 당시 y 조회 값) + 100(x값) == 300
	* TransactionB :
		* 100(read_lock 당시 x 조회 값) + 200(y값) == 300
* Lock 을 사용했지만, NonSerializable 한 형태로 동작을 했다.
* 이유는 업데이트 이전에, 미리 값을 읽어서 처리했기 때문이다. 

### 문제 해결

```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 100, y = 200

Database -> TransactionB : read_lock(x)
TransactionB -> Database : read(x)
Database -> TransactionB : write_lock(y)

Database -> TransactionA : read_lock(y)
Database <--> TransactionA : [ TransactionA read_lock(y) is block ]

TransactionB -> Database : unlock(x)
TransactionB -> Database : read(y) = 200
TransactionB -> Database : write(y = x + y) = 300
TransactionB -> Database : unlock(y)

Database <--> TransactionA : [ TransactionA read_lock(y) is start ]
TransactionA -> Database : read(y) = 300
Database -> TransactionA : write_lock(x)
TransactionA -> Database : unlock(y)
TransactionA -> Database : read(x) = 100
TransactionA -> Database : write(x = x + y) = 400
TransactionA -> Database : unlock(x)

Database -> Database : Result : x = 400, y = 300

```

* TransactionB의 write_lock(y)와 unlock(x)의 위치를 바꾸었다.
* 이로인해, TranSactionA는 read_lock 을 얻는데 있어서 대기상태가 된다.
* 즉, TransactionB의 모든 작업이 끝난후 TrasactionA의 작업이 실행된다.
* **중요** : TrasactionA의 unlock(y)와 write_lock(x) 의 순서도 바꾸었다.
	* 위 그래프는 TransactionB 가 먼저 실행된다 가정이었으므로, 반대 상황도 대비


```plantuml
participant TransactionA
database Database
participant TransactionB


Database -> TransactionB : **read_lock(x)**
Database -> TransactionB : **write_lock(y)**
Database -> TransactionA : **read_lock(y)**
TransactionB -> Database : unlock(x)
TransactionB -> Database : unlock(y)
Database -> TransactionA : **write_lock(x)**
TransactionA -> Database : unlock(y)
TransactionA -> Database : unlock(x)
```

* Lock을 획득하고 해제하는 과정만 놓고 본다면 위와 같은 시퀀스가 나온다.
* **특이점**은 `락 획득` 과정을 먼저 수행하고 언락을 하는 것을 확인할 수 있다. 

# 2PL Protocol
* tx(transaction) 에서 
  모든 locking operation이 최초의 unlock operation보다 먼저 수행되도록 하는 것이다.
* Serializability 보장  

## Expanding Phase(growing phase)
```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> TransactionB : **read_lock(x)**
Database -> TransactionB : **write_lock(y)**
Database -> TransactionA : **read_lock(y)**
Database -> TransactionA : **write_lock(x)**
```
* Lock 을 취득하기만 하고 반환하지 않는 phase


## Shrinking Phase(contracting phase)
```plantuml
participant TransactionA
database Database
participant TransactionB


TransactionB -> Database : unlock(x)
TransactionB -> Database : unlock(y)
TransactionA -> Database : unlock(y)
TransactionA -> Database : unlock(x)
```
* Lock 을 반환만 하고 취득하지 않는 phase

## 2PL의 문제점
```plantuml
participant TransactionA
database Database
participant TransactionB

Database -> Database : x = 100, y = 200

Database -> TransactionB : read_lock(x)
Database -> TransactionA : read_lock(y)
TransactionA -> Database : read(y) = 200
Database -> TransactionA : write_lock(x)
Database <--> TransactionA : [ TransactionA write_lock(y) is block(read_lock(x) using) ]


TransactionB -> Database : read(x)
Database -> TransactionB : write_lock(y)
Database <--> TransactionB : [ TransactionA write_lock(x) is block(read_lock(y) using) ]
TransactionA <-> TransactionB : DeadLock
```
* Lock을 획득하고 사용하는 로직에 따라서 상황이 바뀔 수 있다.
* 2PL 은 Lock 을 먼저 획득하는 작업으로 인하여 데드락이 발생할 수 있다.  
* 데드락 해결방식은 OS 의 데드락 해결방식과 비슷하다.

## 2PL Protocol 종류
### 시나리오

* 트랜잭션 : 
	* x와 y와 z를 더해서 y에 쓴다.
	* x에 2를 곱해서 z를 쓴다.
* 순서 : 
	* read_lock(x)
		* read(x)
	* write_lock(y)
		* read(y)
	* write_lock(z)
	* unlock(x)
		* read(z)
		* write(y = x + y + z)
	* unlock(y)
		* write(z = 2 * x)
	* unlock(z)


###  convervative 2PL
* 순서 : 
	* read_lock(x)
	* write_lock(y)
	* write_lock(z)
		* read(x)
	* unlock(x)
		* read(y)
		* read(z)
		* write(y = x + y + z)
	* unlock(y)
		* write(z = 2 * x)
	* unlock(z)

* 특징 : 
	 * 모든 lock 을 취득한 뒤 transaction 을 시작한다.
	 * deadlock-free
	 * 실용적이지는 않다.

### strict 2PL(S2PL)
* 순서 : 
	* read_lock(x)
		* read(x)
	* write_lock(y)
		* read(y)
	* write_lock(z)
	* unlock(x)
		* read(z)
		* write(y = x + y + z)
		* write(z = 2 * x)
	* **commit**
	* unlock(y)
	* unlock(z)

* 특징 :
	* strict schedule 을 보장하는 2PL
	* recoverability 보장
	* write-lcok 을 commit / rollback 될 때 반환

### strong strict 2PL(SS2PL or Rigorous 2PL)
* 순서 : 
	* read_lock(x)
		* read(x)
	* write_lock(y)
		* read(y)
	* write_lock(z)
		* read(z)
		* write(y = x + y + z)
		* write(z = 2 * x)
	* **commit**
	* unlock(x)
	* unlock(y)
	* unlock(z)
* 특징 :
	* strict schedule 을 보장하는 2PL
	* recoverability 보장
	* read-lock 및 write-lcok 모두 commit / rollback 될 때 반환
	* S2PL 보다 구현이 쉽다.
	* 단, Lock 점유 시간이 길어지는 문제는 있다. 

## 2PL Protocol 단점
* read-read  트랜잭션을 제외하고는,
  한쪽이 block 이 되니까 전체 처리량이 좋지 않다.
* 그래서 고안해낸게, write-write 를 제외한 Lock 조합에 대한 블락을 해결 하자 
  => MVCC(Mutliversion Concurrency control)
