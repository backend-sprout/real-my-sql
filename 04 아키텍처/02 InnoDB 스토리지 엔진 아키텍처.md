# InnoDB 스토리지 엔진 아키텍처

![[innodb-architecture.png]]

InnoDB는 MySQL에서 사용할 수 있는 스토리지 엔진중에서 거의 유일하게 레코드 기반의 잠금을 제공한다.  
이때문에 높은 동시성 처리가 가능하고 안정적이며 성능이 뛰어나다.

## 프라이머리 키에 의한 클러스터링
InnoDB의 모든 테이블은 기본적으로 프라이머리 키를 기준으로 클러스터링되어 저장된다.    
즉, 프라이머리 키 값의 순서대로 디스크에 저장된다는 뜻이며      
모든 세컨더리 인덱스는 레코드의 주소 대신 프라이머리 키의 값을 논리적인 주소로 사용한다.   

프라이머리키가 클러스터링 인덱스이기 때문에 프라이머리 키를 이용한 레인지 스캔은 상당히 빨리 처리된다.  
기본적으로 프라이머리키는 다른 인덱스에 비해 비중이 높게 설정되어 자주 사용될 가능성이 높다.  
 
InnoDB와 달리 MyISAM 스토리지 엔진에서는 클러스터링키를 지원하지 않는다.    
그래서 MyISAM 테이블에서는 프라이머리 키와 세컨더리 인덱스는 구조적으로 아무 차이가 없다.    
그리고 MyISAM 테이블의 프라이머리 키를 포함한 모든 인덱스는 물리적인 레코드의 주소 값을 가진다.  

## 외래 키 지원 
외래키에 대한 지원은 InnoDB 스토리지 엔진 레벨에서 지원하는 기능이다.
외래 키는 서비스의 불편함 때문에 서비스용 데이터베이스에서는 생성하지 않은 경우가 자주 있는데  
그렇다 하더라도 개발 환경의 데이터베이스에서는 좋은 가이드 역할을 할 수 있다.  

InnoDB 에서 외래키는 부모 테이블과 자식 테이블 모두 해당 칼럼에 인덱스 생성이 필요하고   
변경시에는 반드시 부모테이블이나 자식 테이블에 데이터가 있는지 체크하는 작업도 필요하다.      
즉 **잠금이 여러 테이블로 전파되고, 그로인해 데드락이 발생할 때가 많으니 주의가 필요하다.**    

수동으로 데이터를 적재학나 스키마 변경등의 관리 작업이 실패할 수 있다.   
물론 관계를 파악하고 있으면 괜찮지만, 복잡하게 얽힐 경우 그렇게 간단하지 않다.   
또한 서비스에 문제가 있어서 긴급하게 뭔가 조치를 해야하는데 이런 문제가 발생하면 더 조급할 수 있다.  

* `foreign_key_checks` 시스템 변수를 OFF로 설정하면 
  외래 키 관계에 대한 체크 작업을 일시적으로 멈출 수 있다.
* 외래 키 관계를 멈추고 작업을 진행했다면 데이터 정합성을 위해 연관된 테이블에도 신경을 쓰자
* 참고로 위 명령어를 사용하면 부모 테이블에 대한 작업
  (ON DELETE CASCADE  / ON UPDATE CASCADE) 도 무시하게 된다. 

## MVCC
레코드 레벨의 트랜잭션을 지원하는 DBMS가 제공하는 기능이다.  
MVCC의 가장 큰 목적은 **잠금을 사용하지 않는 일관된 읽기를 제공하는데 있다.**  

InnoDB는 `UnDO 로그`를 이용해 이 기능을 구현했다.    
MVCC의 멀티 버전은 하나의 레코드에 대해 여러개의 버전이 동시에 관리된다는 의미이다.  

이해를 위해 격리 수준이 READ_COMMITED 인 MySQL 서버에서
InnoDB 스터리지 엔진을 사용하는 테이블의 데이터 변경을 어떻게 처리하는지 살펴보자.   

![[images_fortice_post_f3b40a28-14d1-4ff6-85f6-1f1648c0ec77_image.png]]

UPDATE 문장이 실행되면   
커밋 실행 여부와 관계 없이 InnoDB의 버퍼 풀은 새로운 값으로 업데이트 된다.  
그리고 디스크의 데이터 파일에는 체크 포인트나 InnoDB 의 Write 쓰레드에 의해 
새로운 값으로 업데이트 될수도 있고 아닐 수도 있다.   

아직 COMMIT 이나 ROLLBACK이 되지 않은 상태에서 다른 사용자가 작업중인 레코드를 조회하면?   
격리수준마다 다르게 결과값이 나온다.  

ReadUncommitted 는 버퍼풀이 현재 가지고 있는 값(변경 된 값)을 읽어서 반환한다.   
ReadCommitted 이상의 격리수준인 경으 언두 영역의 데이터를 반환한다.   
이러한 과정을 DBMS에서는 MVCC 라고 표현한다.  

즉 하나의 레코드에 대해서 2개의 버전이 유지되고,   
필요에 따라 어느 데이터가 보여지는지 여러가지 상황에 따라 달라지는 구조다.   
(정확히 말하면 write-lock/write-lock 구조가 락 구성시 대기를 최소화 하도록 하는것)

즉, 새로운 데이터로 변경이 이루어지면 
버퍼 풀에 있는 데이터가 언두 영역으로 옮겨지고 
COMMIT 시 InnoDB 버퍼풀에 데이터(변경된)가 영구 저장
ROLLBACK시 언두 영역 데이터로 InnoDB 버퍼풀로 복구 하는 것을 알 수 있다.   
  
하지만, 커밋이 된다고 언두 영역의 백업 데이터가 항상 바로 삭제되는 것은 아니다.   
이 언두 영역을 필요로하는 틀내잭션이 더는 없을 때 비로소 삭제된다.  

## 잠금 없는 일관된 읽기(Non-Locking Consistent Read)

InnoDB 스토리지 엔진은 MVCC 기술을 이용해 잠금을 걸지 않고 읽기 작업을 실행한다.    
즉, 다른 트랜잭션에서 락이 발생해도 `읽기`에 한하여 잠금 대기를 하지 않고 데이터를 읽어올 수 있는 형태다.    
  
격리수준이 SERIALIZABLE 이 아닌   
READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ 수준인 경우   
INSERT 와 연결되지 않은 순수한 읽기 작업은   
다른 트랜잭션의 변경 작업과 관계없이 잠금을 대기하지 않고 실행된다.     
(SERIALIZABLE 은 읽기시 Select for update 를 이용해 락을 걸기 때문에 MVCC 매커니즘과 다르다.)      
  
쓰기 입장에서도    
아직 커밋을 수행하지 않았다 하더라도 다른 사용자의 SELECT 작업을 방해하지 않는다.     
이를 `잠금 없는 일관된 읽기`라고 표현하며    
InnoDB에서는 변경되기 전의 데이터를 읽기 위해 언두로그를 사용한다.  

오랜 시간동안 호라성 상태인 트랜잭션으로 인해 MySQL 서버가 느려지거나 문제가 발생할 때가 있는데      
바로 이러한 일관된 읽기를 위해 언두 로그를 삭제하지 못하고 계속 유지해야하기 때문에 발생하는 문제이다.      
따라서 트랜잭션이 시작됐다면 가능한 한 빨리 롤백이나 커밋을 통해 트랜잭션을 완료하는 것이 좋다.   
 
## 자동 데드락 감지
  
InnoDB 스토리지 엔진은     
내부적으로 잠금이 교착상태에 빠지지 않았는지 체크하기 위해 잠금 대기 목록을 그래프(Wait-for List) 형태로 관리한다.          
      
InnoDB 스토리지 엔진은 별도의 데드락 감지 스레드를 가지고 있어        
데드락 감지 스레드가 주기적으로 잠금 대기 그래프를 검사해 교착 상태에 빠진 트랜잭션을 찾아서 그 중 하나를 종료시킨다.           
종료되는 트랜잭션 선정 기준은 언두 로그의 양이며, 언두 로그 레코드를 더 적게 가진 트랜잭션이 일반적으로 롤백 대상이 된다.   

참고로 InnoDB는 상위 엔진인 MySQL엔진에서 관리되는 테이블 잠금을 볼수 없어 데드락 감지가 불확실할 수 있는데      
innodb_table_lock 시스템 변수를 활성화하면, 테이블 레벨까지도 감지할 수 있어서 이를 설정해주자      

일반적인 서비스의 데드락 감지는 부담되지 않지만, 
동시 처리 스레드가 너무 많아지거나 각 틀내잭션이 가진 잠금의 개수가 많아지면 데드락 감지 스레드가 느려진다.  
    
잠금 감지 스레드는 잠금 리스트에 새로운 잠금을 걸고 데드락 검사를 진행한다.       
데드락 감지 스레드가 느려지면 서비스 처리 스레드는 더 작업을 진행하지 못하고 대기하면서 서비스에 악영향을 미친다.        
이렇게 동시 처리 스레드가 매우 많을 경우 데드락 감지 스레드는 더 많은 CPU 자원을 소모할 수 있다.    

이런 문제점을 해결하기 위해 MySQL 서버는 innodb_deadlock_detect 시스템 변수를 제공하며    
innodb_deadlock_detect 시스템 변수를 off로 설정하면 데드락 감지 스레드는 더는 작동하지 않는다.     

## 자동화된 장애 복구 

InnoDB는 손실이나 장애로부터 데이터를 보호하기 위한 여러가지 매커니즘이 탑재되어있다.  
이 매커니즘을 이용해 MySQL 서버가 시작될 때   
완료되지 못한 트랜잭션이나 디스크에 일부만 기록된 데이터 페이지 등에 대한 일련의 복구 작업이 자동으로 진행된다.    

InnoDB의 스토리지 엔진은 매우 견고해서 데이터 파일이 손상되거나 MySQL 서버가 시작되지 못하는 경우는 거의 발생하지 않는다.  
하지만 MySQL 서버와 무관하게 디스크나 서버 하드웨어 이슈로 InnoDB 스토리지 엔진이 자동으로 복구를 못하는 경우가 있다.   
InnoDB 데이터 파일은 기본적으로 MySQL 서버가 시작될 때 항상 자동 복구를 수행한다.   
이 단계에서 복구가 될 수 없는 손상이 있다면 자동 복구를 멈추고 MySQL 서버는 종료된다.  

이때는 MySQL 서버의 설정 파일에 innodb_force_recovery 시스템 변수를 설정해서 MySQL 서버를 시작해야한다.  

* InnoDB 로그 파일이 손상됐다면 6으로 설정하고 MySQL 서버를 가동한다.     
* InnoDB 테이블의 데이터 파일이 손상됐다면 1로 설정하고 MySQL 서버를 기동한다.  
* 어떤 부분이 문제인지 알 수 없다면 innodb_force_recovery 값을 1부터 6까지 변경하면서 재시작해본다.    

MySQL 서버가 기동되고 InnoDB 테이블이 인식된다면   
mysqldump 를 이용해 데이터를 가능한 만큼 백업하고 그 데이터로 MySQL 서버의 DB와 테이블을 다시 생성하는 것이 좋다.   
innodb_force_recovery 설정 가능한 값은 1~6 까지인데 이에 대한 설명은 아래와 같다.  

**1(SRV_FORCE_IGNORE_CORRUPT)**
* InnoDB 테이블스페이스의 데이터나 인덱스 페이지에서 손상된 부분이 발견돼도 무시하고 MySQL 서버를 시작한다.  
* 에러가 발생하면, mysqldump 나 SELECT INTO OUTFILE... 명령어를 이용해 덤프해서 DB를 다시 구축하는게 좋다.    

**2(SRV_FORCE_NO_BACKGROUND)**
* InnoDB는 쿼리의 처리를 위해 여러 종류의 백그라운드 스레드를 동시에 사용한다.  
* 이 복구 모드에서는 이러한 백그라운드 스레드 가운데 메인 스레드를 시작하지 않고 MySQL 서버를 시작한다.  
* InnoDB는 트랜잭션의 롤백을 위해 언두 데이터를 관리하는데, 
  트랜잭션이 커밋되어 불필요한 언두 데이터는 InnoDB 의 메인스레드에 의해 주기적으로 삭제된다.(Undo Purge)
* InnoDB의 메인 스레드가 언두 데이터를 삭제하는 과정에서 장애가 발생한다면 이 모드로 복구하면 된다.  

**3(SRV_FORCE_NO_TRX_UNDO)**  
* InnoDB 에서 트랜잭션이 실행되면 롤백에 대비해 변경 전의 데이터를 언두 영역에 기록한다.   
* 일반적으로 MySQL 서버는 다시 시작하면서 언두 영역의 데이터를 먼저 데이터 파일에 적용하고   
  그 다음에 리두 로그의 내용을 다시 덮어써서 장애 시점의 데이터 상태를 만들어낸다.   
* 그리고 정상적인 MySQL 서버의 시작에서는 최종적으로 커밋되지 않은 트랜잭션은 롤백을 수행하지만   
  3 으로 설정되면 커밋되지 않은 트랜잭션의 작업을 롤백하지 않고 그대로 놔둔다.   
* 즉, 커밋되지 않고 종료된 트랜잭션은 계속 그 상태로 남아있게 MySQL 서버를 시작하는 모드다.  
* 이때도 우선 MySQL 서버가 시작되면 mysqldump를 이용해 데이터를 백업해서 다시 데이터베이스를 구축하는 것이 좋다. 

**4(SRV_FORCE_NO_IBUF_MERGE)**    
* InnoDB는 INSERT, UPDATE, DELETE 등의 데이터 변경으로 인한 인덱스 변경 작업을   
  상황에 따라 즉시 수행할 수 있고 인서트 버퍼에 저장해두고 니증에 처리할 수 있다.    
* 이렇게 인서트 버퍼에 기록된 내용은 언제 데이터 파일에 병홥(Merge)될지 알 수 없다.  
* MySQL 을 종료해도 병합되지 않을 수 있는데, 만약 MySQL이 재시작되면서 인서트 버퍼의 손상을 감지하면   
  InnoDB는 에러를 발생시키고 MySQL 서버는 시작하지 못한다.    
* 4 로 설정하면 InnoDB 스토리지 엔진이 인서트 버퍼의 내용을 물시하고 강제로 MySQL을 시작되게 한다.   
* 인서트 버퍼는 실제 데이터와 관련된 부분이 아니라 인덱스에 관련된 부분이므로 테이블을 덤프한 후 다시 데이터베이스의 손실없이 복구할 수 있다.

**5(SRV_FORCE_NO_UNDO_LOG_SCAN)**.    
* MySQL 서버가 장애나 정상적으로 종료되는 시점에 진행중인 트랜잭션이 있었다면     
  MySQL 은 그냥 단순히 그 컼넥션을 강제로 끊어 버리고 별도의 정리 작업 없이 종료한다.     
* MySQL이 다시 시작하면 InnoDB 엔진은 언두 레코드를 이용해서 데이터 페이지를 복구하고 
  리두 로그를 적용해 종료 시점이나 장애 발생 시점의 상태를 재현해낸다.    
* 그리고 InnoDB는 마지막으로 커밋되지 않은 트랜잭션에 변경한 작업은 모두 롤백 처리한다.   
  그런데 InnoDB의 언두 로그를 사용할 수 없다면 InnoDB 엔진의 에러로 MySQL 서버를 시작할 수 없다.  
* 5로 설정하면 InnoDB 엔진이 언두 로그를 모두 무시하고 MySQL 을 시작할 수 있다.  
* 하지만, 이 모드로 복구되면 MySQL 서버가 종료되던 시점에 커밋되지 않았던 작업도 모두 커밋된 것처럼 처리되므로 
  실제로는 잘못된 데이터가 데이터베이스에 남는 것이라고 볼 수 있다.  
  이때도 mysqldump 를 이용해 데이터를 백업하고, 데이터베이스를 새로 구축해야한다.  
  
**6(SRV_FORCE_NO_LOG_READ)**       
* InnoDB 스토리지 엔진의 리두 로그가 손상되면 MySQL 서버가 시작되지 못한다.     
* 이 복구 모드로 시작하면 InnoDB 엔진은 리두 로그를 모두 무시한채로 MySQL 서버가 시작된다.    
* 또한 커밋됐다 하더라도 리두 로그에만 기록되고 데이터 파일에 기록되지 않은 데이터는 모두 무시된다.     
* 즉, 마지막 체크 포인트 시점의 데이터만 남게 된다.  
* 이때는 기존 InnoDB의 리두 로그는 모두 삭제하고 MySQL 서버를 시작하는 것이 좋다.    
* MySQL 서버가 시작하면서 리두 로그가 없으면 새로 생성하므로 별도로 파일을 만들 필요는 없다.     
* 이때도 mysqldump 를 이용해 데이터를 모두 백업해서 MySQL 서버를 새로 구축하는 것이 좋다.   
  
6가지 복구 프로세스를 이용해도 서버가 시작되지 않으면 백업을 이용해 다시 구축하는 방법밖에 없다.    

## InnoDB 버퍼 풀 

InnoDB 스토리지 엔진에서 가장 핵심적인 부분으로    
디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해두는 공간이다.       
**쓰기 작업을 지연시켜 일괄 작업으로 처리할 수 있게 해주는 버퍼 역활과 같이한다.**   

일반적인 INSERT, UPDATE, DELETE 처럼 데이터를 변경하는 쿼리는   
데이터 파일의 이곳저곳에 위치한 레코드를 변경하 때문에 랜덤한 디스크 작업읇 발생시킨다.  
하지만 버퍼풀이 이러한 변경된 데이터를 모아서 처리하면 랜덤한 디스크 작업의 횟수를 줄일 수 있다.   

### 버퍼 풀의 크기 설정
  
일반적으로 전체 물리 메모리의 80% 정도를 InnoDB의 버퍼풀로 설정하라는 내용의 게시물도 있는데     
그렇게 단순하게 설정되는 값은 아니며 운영체제와 각 클라이언트 스레드가 사용하는 메모리도 충분히 고려해야한다.   
  
MySQL 서버내에서 메모리를 필요로 하는 부분은 크게 없지만 간혹 레코드 버퍼가 상당한 메모리를 사용하기도 한다.      
레코드 버퍼는 각 클라이언트 세션에서 테이블의 레코드를 읽고 쓸때 버퍼로 사용하는 공간을 말하는데     
커넥션이 많고 사용하는 사용하는 테이블도 많다면 레코드 버퍼 용도로 사용되는 메모리 공간이 꽤 많이 필요해질 수 있다.    

MySQL 서버가 사용하는 레코드 버퍼 공간은 별도로 설정할 수 없으며     
전체 커넥션 개수와 각 커넥션에 읽고 쓰는 테이블의 개수에 따라서 결정된다.
또한 이 버퍼 공간은 동적으로 해제되기도 하므로 정확히 필요한 메모리 공간의 크기를 게산할 수는 없다.  

다행히 MySQL 5.7버전부터는 InnoDB 버퍼 풀의 크기를 동적으로 조절할 수 있게 개선되었다.  
그래서 처음에는 적절히 작은값부터 시작해서 점진적으로 증가시키는 방법이 최적이다.   

윤영체제의 전체 메모리 공간이 8GB 미만이라면 50%정도만 InnoDB 버퍼풀로 설정하고  
나머지 메모리 공간은 MySQL 서버와 운영체제, 그리고 다른 프로그램을 위해 메모리를 확보해두는 것이 좋다.  
이후 앞서 말한대로 점진적으로 증가시키는 방향으로 작업을 진행하면 된다.  

InnoDB의 버퍼풀은 innodb_buffer_pool_size 시스템 변수로 크기를 설정할 수 있으며 동적 변경도 허용한다.  
동적 변경의 경우 크리티컬한 변경이므로 한가한 시간대를 잡고 변경하는 것이 좋다.   
InnoDB 버퍼풀의 크기를 줄이는 작업은 서비스 영향도가 매우 크므로 왠만하면 진행하지 말자    

InnoDB 버퍼풀은 내부적으로 128MB 청크 단위로 쪼개 관리하는데       
이는 버퍼풀의 크기를 줄이거나 늘리기 위한 단위 크기로 사용한다.        
그래서 버퍼 풀의 크기를 줄이거나 늘릴때는 128MB 단위로 처리된다.     

InnoDB 버퍼풀은 전통적으로 버퍼풀 전체를 관리하는 잠금(세마포어)으로 인해 내부 잠금 경합을 많이 유발해왔는데     
이런 경합을 줄이기 위해 버퍼 풀을 여러개로 쪼개어 관리할 수 있도록 개선되었다.     
버퍼풀이 여러 개의 작은 버퍼풀로 쪼개면서 
개별 버퍼 풀 전체를 관리하는 잠금(세마포어) 자체도 경합이 분산되는 효과를 내게 된 것이다.    
  
각 버퍼풀을 버퍼풀 인스턴스라고 표현한다.    
기본적으로 버퍼풀 인스턴스의 개수는 8개로 초기화되지만      
전체 버퍼풀을 위한 메모리 크기가 1GB 미만이면 버퍼풀 인스턴스는 1개만 생성된다.      
  
버퍼 풀로 할당할 수 있는 메모리 공간이 40GB 이하 수준이라면 기본 값인 8을 유지하고    
메모리가 크다면 버퍼 풀 인스턴스당 5GB 정도가 되게 인스턴스 개수를 설정하는 것이 좋다.  

### 버퍼 풀의 구조 
InnoDB 스토리지 엔진은 버퍼풀이라는 거대한 메모리 공간을   
페이지 크기의 조각으로 쪼개어 InnoDB 스토리지 엔진이 데이터를 필요로 할 때 해당 페이지를 읽어서 각 조각에 저장한다.    
버퍼풀의 페이지 크기 조각을 관리하기 위해 InnoDB 스토리지 엔진은 크게 3가지 자료구조를 사용한다.   
  
* LRU 리스트
* 플러시 리스트 
* 프리 리스트 






  
  
  
  






   
