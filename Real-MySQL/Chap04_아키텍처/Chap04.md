# Chapter 04 아키텍처
MySQL 서버는 MySQL 엔진(머리 역할)과 스토리지 엔진(손발 역할)로 구분된다.

스토리지 엔진은 핸들러 API를 만족하면 구현해서 MySQL 서버에 추가해 사용할 수 있다.
* MySQL 엔진과 기본 스토리지 엔진(InnoDB, MyISAM)을 알아보자
## 4.1 MySQL 엔진 아키텍처
MySQL 서버는 다른 DBMS에 비해 구조가 상당히 독특하다.
* 혜텍을 누릴수도, 문제가 되기도 한다.

### 4.1.1 MySQL의 전체 구조


1. 프로그래밍 API - Java, C++, .NET, php, python, Go, Ruby ,,,
2. 커넥션 핸들러 - 2,3 MYSQL 엔진 - 이 아래로 쭉 MySQL Server
3. SQL 인터페이스, SQL 파서, SQL 옵티마이저, 캐시 & 버퍼
4. InnoDB, MyISAM, Memory - 스토리지 엔진 API
5. 데이터 파일, 로그 파일, 디스크 - 운영체제, 하드웨어

MySQL은 일반 상용 RDBMS와 같이 대부분의 프로그래밍 언어로부터 접근 방법을 모두 지원

MySQL 서버는 크게 MySQL 엔진과 스토리지 엔진으로 구분 가능


### 4.1.1.1 MySQL 엔진
* 커넥션 핸들러 : 클라이언트로부터의 접속, 쿼리 요청 처리
* SQL 파서, 전처리기
* 옵티마이저 : 쿼리의 최적화된 실행을 위한
* ANSI SQL (표준 SQL 문법)을 지원하기 때문에 타 DBMS와 호환 가능

### 4.1.1.2 스토리지 엔진
디스크 스토리지로부터 데이터를 읽거나 저장하는 작업 수행
* 스토리지 엔진은 여러 개를 동시에 사용할 수 있다.

CREATE TABLE test_table (fd1 INT, fd2 INT) ENGINE=INNODB;
* 테이블을 생성할 때 스토리지 엔진을 등록하면 해당 테이블의 작업은 지정한 스토리지 엔진이 수행

### 4.1.1.3 핸들러 API
핸들러 요청 : MySQL 엔진의 쿼리 실행기에서 데이터를 쓰거나 읽어야할 때, 각 스토리지 엔진에 쓰기 또는 읽기를 요청하는 것
* 여기서 사용되는 API를 핸들러 API라고 한다.

얼마나 많은 데이터(레코드) 작업이 있었는지 확인하는 명령
~~~SQL
SHOW GLOBAL STATUS LIKE 'Handler%';
~~~

### 4.1.2 MySQL 스레딩 구조
MySQL 서버는 프로세스 기반이 아닌 스레드 기반으로 작동한다.
* 포그라운드 스레드와 백그라운드 스레드로 구분

실행 중인 스레드의 목록을 조회하는 명령
~~~SQL
SELECT thread_id, name, type, processlist_user, processlist_host FROM performance_schema.threads ORDER BY type, thread_id;

+-----------+---------------------------------------------+------------+------------------+------------------+
| thread_id | name                                        | type       | processlist_user | processlist_host |
+-----------+---------------------------------------------+------------+------------------+------------------+
|         1 | thread/sql/main                             | BACKGROUND | NULL             | NULL             |
|         2 | thread/mysys/thread_timer_notifier          | BACKGROUND | NULL             | NULL             |
|         4 | thread/innodb/io_ibuf_thread                | BACKGROUND | NULL             | NULL             |
|         5 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
|         6 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
|         7 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
|         8 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
|         9 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
|        10 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
|        11 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
|        12 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
|        13 | thread/innodb/page_flush_coordinator_thread | BACKGROUND | NULL             | NULL             |
|        14 | thread/innodb/log_checkpointer_thread       | BACKGROUND | NULL             | NULL             |
|        15 | thread/innodb/log_flush_notifier_thread     | BACKGROUND | NULL             | NULL             |
|        16 | thread/innodb/log_flusher_thread            | BACKGROUND | NULL             | NULL             |
|        17 | thread/innodb/log_write_notifier_thread     | BACKGROUND | NULL             | NULL             |
|        18 | thread/innodb/log_writer_thread             | BACKGROUND | NULL             | NULL             |
|        19 | thread/innodb/log_files_governor_thread     | BACKGROUND | NULL             | NULL             |
|        24 | thread/innodb/srv_lock_timeout_thread       | BACKGROUND | NULL             | NULL             |
|        25 | thread/innodb/srv_error_monitor_thread      | BACKGROUND | NULL             | NULL             |
|        26 | thread/innodb/srv_monitor_thread            | BACKGROUND | NULL             | NULL             |
|        27 | thread/innodb/buf_resize_thread             | BACKGROUND | NULL             | NULL             |
|        28 | thread/innodb/srv_master_thread             | BACKGROUND | NULL             | NULL             |
|        29 | thread/innodb/dict_stats_thread             | BACKGROUND | NULL             | NULL             |
|        30 | thread/innodb/fts_optimize_thread           | BACKGROUND | NULL             | NULL             |
|        31 | thread/mysqlx/worker                        | BACKGROUND | NULL             | NULL             |
|        32 | thread/mysqlx/worker                        | BACKGROUND | NULL             | NULL             |
|        33 | thread/mysqlx/acceptor_network              | BACKGROUND | NULL             | NULL             |
|        37 | thread/innodb/buf_dump_thread               | BACKGROUND | NULL             | NULL             |
|        38 | thread/innodb/clone_gtid_thread             | BACKGROUND | NULL             | NULL             |
|        39 | thread/innodb/srv_purge_thread              | BACKGROUND | NULL             | NULL             |
|        40 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
|        41 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
|        42 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
|        44 | thread/mysqlx/acceptor_network              | BACKGROUND | NULL             | NULL             |
|        47 | thread/sql/con_sockets                      | BACKGROUND | NULL             | NULL             |
|        43 | thread/sql/event_scheduler                  | FOREGROUND | event_scheduler  | localhost        |
|        46 | thread/sql/compress_gtid_table              | FOREGROUND | NULL             | NULL             |
|        51 | thread/sql/one_connection                   | FOREGROUND | root             | localhost        |
+-----------+---------------------------------------------+------------+------------------+------------------+
~~~
* 마지막 3개만 포그라운드 스레드
* 백그라운드 스레드의 개수는 MySQL 서버의 설정과 내용에 따라 가변적일 수 있다.
* 동일한 이름의 스레드가 2개 이상 보이는 것은 여러 스레드가 동일 작업을 병렬로 처리하는 경우

전통적인 스레드 모델 : 커넥션과 포그라운드 스레드가 1:1 관계

스레드 풀 : 하나의 스레드가 여러 개의 커넥션 요청을 전담


### 4.1.2.1 포그라운드 스레드(클라이언트 스레드)
포그라운드 스레드는 최소한 MySQL 서버에 접속된 클라이언트 수만큼 존재
* 각 사용자가 요청하는 쿼리 문장 처리
* 사용자가 작업을 마치고 커넥션을 종료하면 해당 커넥션을 담당하던 스레드는 스레드 캐시로 되돌아간다.
* 이 때 일정 개수 이상의 대기 중인 스레드가 있다면 스레드 캐시에 넣지 않고
* 스레드를 종료시켜 일정 개수의 스레드만 스레드 캐시에 존재하게 된다.
* thread_cahe_size 시스템 변수로 설정
* 내 MySQL 서버에서는 9개

~~~SQL
mysql> SHOW GLOBAL VARIABLES LIKE 'thread_cache%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| thread_cache_size | 9     |
+-------------------+-------+
1 row in set, 1 warning (0.00 sec)
~~~

포그라운드 스레드는 데이터를 MySQL의 데이터 버퍼나 캐시로부터 가져오며, 버퍼나 캐시에 없는 경우에는 직접 디스크의 데이터나 인덱스 파일로부터 데이터를 읽어와서 작업을 처리한다.

MyISAM 테이블은 디스크 쓰기 작업까지 포그라운드 스레드가 처리

InnoDB 테이블은 데이터 버퍼나 캐시까지만 포그라운드 스레드가 처리, 나머지는 백그라운드

사용자 스레드와 포그라운드 스레드는 같은 의미
* DBMS 앞단에서 사용자와 통신하므로 포그라운드 스레드라 한다.

### 4.1.2.2 백그라운드 스레드
MyISAM의 경우에는 별로 해당 사항이 없다.

InnoDB가 백그라운드에서 작업하는 것
* 인서트 버퍼를 병합
* 로그를 디스크로 기록
* InnoDB 버퍼 풀의 데이터를 디스크에 기록
* 데이터를 버퍼로 읽어오는 작업
* 잠금이나 데드락을 모니터링

이중 가장 중요한 것
* 로그 스레드
* 쓰기 스레드

MySQL 5.5 버전부터 데이터 읽기 쓰기 스레드를 2개 이상 지정할 수 있다.
* innodb_write_io_threads, innodb_read_io_threads 시스템 변수로 스레드 개수 설정

데이터의 쓰기 작업은 지연(버퍼링)되어 처리될 수 있지만
* 읽기 작업은 절대 지연될 수 없다.
* 일반적인 상용 DBMS는 대부분 쓰기 작업을 버퍼링해서 일괄 처리
* InnoDB 또한 이렇게 처리

MyISAM의 경우 사용자 스레드가 쓰기 작업까지 함께 처리
* INSERT, UPDATE, DELETE 쿼리로 데이터가 변경되는 경우
* 완전히 저장될 때까지 기다리지 않아도 된다.
* MyISAM에서 일반적인 쿼리는 쓰기 버퍼링 기능을 사용할 수 없다.

### 4.1.3 메모리 할당 및 사용 구조
글로벌 메모리 영역과 로컬 메모리 영역으로 구분된다.
* 글로벌 메모리 영역 : MySQL 서버가 시작되면서 운영체제로부터 할당
  
글로벌 메모리 영역과 로컬 메모리 영역은 MySQL 서버 내에 존재하는 많은 스레드가 공유해서 사용하는 공간인지 여부에 따라 구분


### 4.1.3.1 글로벌 메모리 영역
클라이언트 스레드 수와 무관하게 하나의 메모리 공간만 할당
* 필요에 따라 2개 이상을 할당받을 수도 있지만 클라이언트 스레드 수와는 무관
* N개가 생성되더라도 모든 스레드에 의해 공유

대표적인 글로벌 메모리 영역
* 테이블 캐시
* InnoDB 버퍼 풀
* InnoDB 어댑티브 해시 인덱스
* InnoDB 리두 로그 버퍼

### 4.1.3.2 로컬 메모리 영역
MySQL 서버상에 존재하는 클라이언트 스레드가 쿼리를 처리하는데 사용하는 메모리 영역
* 클라이언트가 접속하면 요청을 처리하기 위한 스레드를 하나씩 생성 - 클라이언트 메모리 영역이라고도 함
* 세션 메모리 영역이라고도 함

로컬 메모리는 각 클라이언트 스레드별로 독립적으로 할당되며 절대 공유되어 사용되지 않는다.
* 각 쿼리의 용도별로 필요할 때만 공간이 할당되고, 필요하지 않으면 할당조차 안할 수도 있다. ex 소트 버퍼, 조인 버퍼
* 커넥션이 열려 있는 동안 계속 할당된 상태로 남아있는 공간도 있고
* 쿼리를 실행하는 순간에만 할당했다가 다시 해제하는 공간도 있다.

로컬 메모리 영역은 다음과 같다.
* 정렬 버퍼
* 조인 버퍼
* 바이너리 로그 캐시
* 네트워크 버퍼

### 4.1.4 플러그인 스토리지 엔진 모델
MySQL의 독특한 구조 중 대표적인 것 - 플러그인 모델
* 플러그인해서 사용할 수 있는 것이 스토리지 엔진만 있는 것은 아니다.
* 검색어 파서, 사용자 인증을 위한 것들도 플러그인으로 제공
* 사용자가 직접 개발해서 사용하는 것도 가능


MySQL에서 쿼리가 실행되는 과정

MySQL 엔진의 처리 영역
1. SQL 파서
2. SQL 옵티마이저
3. SQL 실행기

스토리지 엔진의 처리 영역
4. 데이터 읽고/쓰기

5. 디스크


데이터 읽기/쓰기 작업은 대부분 1건의 레코드 단위로 처리된다.


핸들러
* MySQL 엔진은 사람 역할
* 각 스토리지 엔진은 자동차 역할
* MySQL 엔진이 스토리지 엔진을 조정하기 위해 핸들러를 사용

MySQL 엔진이 각 스토리지 엔진에서 데이터를 읽어오거나 저장하도록 명령하려면 반드시 핸들러를 통해야 한다.
* MySQL 상태 변수 중 Handler_ 로 시작하는 것이 많다.
* Handler_가 붙은 변수는 MySQL 엔진이 각 스토리지 엔진에게 보낸 명령의 횟수를 의미하는 변수
* 실질적인 GROUP BY나 ORDER BY 등 복잡한 처리는 MySQL 엔진의 쿼리 실행기에서 처리

중요한 것
* 하나의 쿼리 작업은 여러 하위 작업으로 나뉜다.
* 여기서 각 하위 영역이 MySQL 엔진 영역 혹은 스토리지 엔진 영역에서 처리되는지 구분할 줄 알아야 한다.

MySQL 서버에서 지원하는 스토리지 엔진 확인
~~~SQL
mysql> SHOW ENGINES;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| ndbinfo            | NO      | MySQL Cluster system information storage engine                | NULL         | NULL | NULL       |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| ndbcluster         | NO      | Clustered, fault-tolerant tables                               | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
~~~
Support 컬럼에 표시 될 수 있는 값
* YES : MySQL 서버에 포함되어 있고, 사용 가능으로 활성화된 상태
* DEFAULT : YES와 동일한 상태이지만 필수 스토리지 엔진임을 의미
* NO : 현재 MySQL 서버에 포함되지 않았음을 의미
* DISABLED : 현재 MySQL 서버에는 포함됐지만 파라미터에 의해 비활성 상태

NO로 표지되는 스토리지 엔진을 사용하려면 MySQL 서버를 다시 빌드해야 한다.
* SHOW PLUGINS; 명령으로 플러그인의 내용을 확인할 수 있다.
* 따로 다운로드해서 끼워 넣기만 해도 사용할 수 있다.


### 4.1.5 컴포넌트
MySQL 8.0부터는 기존의 플러그인 아키텍처를 대체하기 위해 컴포넌트 아키텍처가 지원된다.

다음의 플러그인의 단점을 보완하여 구현
* 플러그인은 오직 MySQL 서버와 인터페이스할 수 있고, 플러그인끼리는 통신할 수 없음
* 플러그인은 MySQL 서버의 변수나 함수를 직접 호출하기 때문에 안전하지 않음(캡슐화 안됨)
* 플러그인은 상호 의존 관계를 설정할 수 없어서 초기화가 어려움

### 4.1.6 쿼리 실행 구조

### 4.1.6.1 쿼리 파서
사용자 요청으로 들어온 쿼리 문장을 토큰(MySQL이 인식할 수 있는 최소 단위의 어휘나 기호)로 분리해 트리 형태의 구조로 만들어 내는 작업을 의미
* 쿼리 문장의 기본 문법 오류는 이 과정에서 발견되고 사용자에게 오류 메시지 전달

### 4.1.6.2 전처리기
파서 과정에서 만들어진 파서 트리를 기반으로 쿼리 문장에 구조적인 문제점이 있는지 확인
* 각 토큰을 테이블 이름이나 컬럼 이름, 또는 내장 함수와 같은 개체를 매핑
* 해당 객체의 존재 여부와 객체의 접근 권한 등을 확인하는 과정 수행
* 실제 존재하지 않거나 권한상 사용할 수 없는 개체의 토큰은 이 단계에서 걸러짐

### 4.1.6.3 옵티마이저
사용자의 요청으로 들어온 쿼리 문장을 저렴한 비용으로 가장 빠르게 처리할지를 결정하는 역할을 담당
* 즉, DBMS의 두뇌에 해당

### 4.1.6.4 실행 엔진
옵티마이저가 두뇌라면 실행 엔진과 핸들러는 손과 발에 비유할 수 있다.
* 옵티마이저 : 회사의 경영진
* 실행 엔진 : 중간 관리자
* 핸들러 : 각 업무의 실무자

옵티마이저가 GROUP BY를 처리하는 과정
1. 실행 엔진이 핸들러에게 임시 테이블을 만들라고 요청
2. 다시 실행 엔진은 WHERE 절에 일치하는 레코드를 읽어오라고 핸들러에게 요청
3. 읽어온 레코드들을 1번에서 준비한 임시 테이블로 저장하라고 다시 핸들러에게 요청
4. 데이터가 준비된 임시 테이블에서 필요한 방식으로 데이터를 읽어 오라고 핸들러에게 다시 요청
5. 최종적으로 실행 엔진은 결과를 사용자나 다른 모듈로 넘김

즉, 실행 엔진은 만들어진 계획대로 각 핸들러에게 요청해서 받은 결과를 또 다른 핸들러의 요청으로 연결하는 역할을 수행


### 4.1.6.5 핸들러(스토리지 엔진)
MySQL 서버의 가장 밑단에서 MySQL 실행 엔진의 요청에 따라 데이터를 디스크로 저장하고 디스크로부터 읽어오는 역할을 담당
* 결국 핸들러는 스토리지 엔진을 의미하고
* MyISAM 테이블을 조작하는 경우에는 핸들러가 MyISAM 스토리지 엔진이되고
* InnoDB 테이블을 조작하는 경우에는 핸들러가 InnoDB 스토리지 엔진이 된다.

### 4.1.7 복제
16장에서 살펴봄


### 4.1.8 쿼리 캐시
쿼리 캐시(Query Cache)는 빠른 응답을 필요로 하는 웹 기반의 응용 프로그램에서 매우 중요한 역할을 담당했다.

쿼리 캐시는 SQL의 실행 결과를 메모리에 캐시하고, 동일 SQL 쿼리가 실행되면 테이블을 읽지않고 즉시 결과를 반환하기 때문에 매우 빠른 성능을 보인다.
* 쿼리 캐시는 테이블의 데이터가 변경되면 캐시에 저장된 결과 중 변경된 테이블과 관련된 것들은 모두 삭제해야 했다.
* 이는 심각한 동시 처리 성능 저하를 유발한다.
* 많은 버그의 원인이 되기도 한다.

결국 MySQL 8.0으로 올라오면서 쿼리 캐시는 MySQL 서버의 기능에서 완전히 제거되었고, 관련 시스템 변수도 모두 제거
* 특정 환경에서는 도움이 되었지만, 이런 서비스는 흔치 않다.
* 큰 도움이 되었던 서비스도 없었다. - 수많은 버그의 원인으로 지목되는 경우가 더 많았다.

### 4.1.9 스레드 풀
스레드 풀의 목적은 내부적으로 사용자의 요청을 처리하는 스레드의 개수를 줄여 동시 처리되는 요청이 많더라도 MySQL 서버의 CPU가 제한된 개수의 스레드 처리에만 집중할 수 있게 해서 서버의 자원 소모를 줄이는 것이 목적
* 스레드 풀은 앞에서 소개한 것처럼 동시에 실행 중인 스레드들을 CPU가 최대한 잘 처리해낼 수 있는 수준으로 줄여서 빨리 처리하게 하는 기능
* 스케줄링 과정에서 CPU 시간을 제대로 확보하지 못하는 경우에는 쿼리 처리가 더 느려지는 사례도 발생할 수 있다.
* 물론 CPU가 적절히 처리하도록 유도한다면 CPU의 프로세서 친화도도 높이고 불필요한 컨텍스트 스위치를 줄여서 오버헤드를 낮출 수 있다.

### 4.1.10 트랜젝션 지원 메타데이터
데이터베이스 서버에서 테이블의 구조 정보와 스토어드 프로그램 등의 정보를 데이터 딕셔너리 혹은 메타데이터라 한다.
* MySQL 서버는 5.7 버전까지 테이블의 구조를 FRM 파일에 저장하고 일부 스토어드 프로그램 또한 파일 기반으로 관리했다.
* 파일 기반의 메타데이터는 생성 및 변경 작업이 트랜잭션을 지원하지 않기 때문에
* 테이블의 생성 또는 변경 도중에 MySQL 서버가 비정상적으로 종료되면 일관되지 않은 상태로 남는 문제가 있었다.

MySQL 8.0 버전부터는 이런 관련 정보 모두 InnoDB 테이블에 저장하도록 개선되었다.
* 시스템 테이블 : MySQL 서버가 작동하는데 기본적으로 필요한 테이블
* 이런 시스템 테이블을 모두 InnoDB 스토리지 엔진을 사용하도록 개선
* 시스템 테이블과 데이터 딕셔너리 정보를 모두 모아 mysql DB에 저장하고 있다.
* mysql DB는 통째로 mysql.ibd라는 이름의 테이블스페이스에 저장된다.
* 따라서 다른 *ibd 파일과는 주의해야 한다.

MySQL 8.0 버전부터 스키마 변경 작업 중간에 MySQL 서버가 비정상으로 종료된다 하더라도 스키마 변경이 완전한 성공 또는 완전한 실패로 정리된다.
* 기존의 파일 기반 메타데이터를 사용할 때와 같이 작업 진행 중인 상태로 남으며 문제를 유발하지 않게 개선