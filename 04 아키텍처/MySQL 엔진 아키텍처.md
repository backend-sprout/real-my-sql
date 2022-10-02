# MySQL 엔진 아키텍처
      
MySQL 서버는 머리 역할을 담당하는 MySQL 엔진과         
손과 발의 역할을 담당하는 스토리지 엔진으로 구분할 수 있다.      
  
스토리지 엔진은 핸들러 API를 만족하면 누구든지 스토리지 엔진을 구현해서 MySQL 서버에 추가해서 사용할 수 있다.        
이번 챕터에서는 MySQL 엔진과 InnoDB 스토리지 엔진, 그리고 MyISAM 스토리지 엔진을 구분해서 살펴보겠습니다.    
    
# MySQL 엔진 아키텍처      
## MySQL의 전체 구조

![images_jsj3282_post_aa130a72-12de-4d7a-b32e-52be236ab026_image](https://user-images.githubusercontent.com/50267433/193463535-1ef89ac4-abda-43f6-b832-eebeabbe4b15.png)

MySQL 은 일반 상용 RDBMS 와 같이 대부분의 프로그래밍 언어로부터 접근 방법을 모두 지원한다.       
MySQL 서버는 크게 MySQL 엔진과 스토리지 엔진으로 구분할 수 있다.         
이 책에서는 MySQL의 쿼리파서 옵티마이저 등과 같은 기능을 스토리지 엔진과 구분하고자 'MySQL 엔진'과 '스토리지 엔진'으로 구분했다.      
그리고 이 둘을 모두 합쳐서 그냥 MySQL 또는 MySQL 서버라고 표현하겠다.        

### MySQL 엔진  
### 스토리지 엔진  
### 핸들러 API   

## MySQL 스레딩 구조   
## 메모리 할당 및 사용 구조   
### 글로벌 메모리 영역  
### 로컬 메모리 영역 
