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
