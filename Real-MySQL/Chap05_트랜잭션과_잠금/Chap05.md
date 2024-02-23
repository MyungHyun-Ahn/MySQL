# 05 트랜잭션과 잠금
이번 장에서 알아볼 것
* 잠금(Lock), 트랜잭션, 트랜잭션의 격리 수준(Isolation level)

트랜잭션
* 작업의 완전성을 보장해주는 것
* 논리적인 작업 셋을 모두 완벽하게 처리
* 처리하지 못한 경우에는 원 상태로 복구(Partial Update 방지)

잠금 : 동시성을 제어하기 위한 기능\
트랜잭션 : 데이터의 정합성을 보장하기 위한 기능\
격리수준 : 트랜잭션 내에서 작업 내용을 어떻게 공유하고 차단할 것인지 결정하는 레벨


## 5.1 트랜잭션
트랜잭션을 지원하지 않는 MyISAM과 트랜잭션을 지원하는 InnoDB의 처리 방식 차이를 살펴보자.\
그리고 트랜잭션을 사용하는 경우 주의해야할 사항도 함께 살펴보자.

### 5.1.1 MySQL에서의 트랜잭션
트랜잭션은 꼭 여러 개의 변경 작업을 수행하는 쿼리가 조합되어야 의미 있는 개념은 아니다.
* 쿼리가 하나든 두개든 관계 없이 논리적인 작업 셋 자체가 100% 적용되거나(COMMIT) 아무것도 적용되지 않아야(ROLLBACK 또는 오류) 함을 보장하는 것

트랜잭션 관점에서 InnoDB 테이블과 MyISAM 테이블의 차이
~~~sql
mysql> CREATE TABLE tab_myisam (fdpk INT NOT NULL, PRIMARY KEY (fdpk)) ENGINE=MyISAM;
mysql> INSERT INTO tab_myisam (fdpk) VALUES (3);

mysql> CREATE TABLE tab_innodb (fdpk INT NOT NULL, PRIMARY KEY (fdpk)) ENGINE=INNODB;
mysql> INSERT INTO tab_innodb (fdpk) VALUES (3);
~~~

테스트용 테이블에 각각 레코드 1건 씩 저장

AUTO-COMMIT 모드에서 다음 쿼리 문장을 각각 실행
~~~sql
--// AUTO-COMMIT 활성화
mysql> SET autocommit=ON;

mysql> INSERT INTO tab_myisam (fdpk) VALUES (1), (2), (3);
mysql> INSERT INTO tab_innodb (fdpk) VALUES (1), (2), (3);
~~~

결과
~~~sql
mysql> INSERT INTO tab_myisam (fdpk) VALUES (1), (2), (3);
ERROR 1062 (23000): Duplicate entry '3' for key 'tab_myisam.PRIMARY'

mysql> INSERT INTO tab_innodb (fdpk) VALUES (1), (2), (3);
ERROR 1062 (23000): Duplicate entry '3' for key 'tab_innodb.PRIMARY'

mysql> SELECT * FROM tab_myisam;
+------+
| fdpk |
+------+
|    1 |
|    2 |
|    3 |
+------+

mysql> SELECT * FROM tab_innodb;
+------+
| fdpk |
+------+
|    3 |
+------+
~~~

두 INSERT 문장 모두 프라이머리 키 중복 오류로 쿼리가 실패

MyISAM
* 오류가 발생했음에도 1, 2 가 INSERT 된 상태로 남아있다.
* 1, 2 차례대로 저장하고 3 에서 키 중복 오류가 발생했기 때문
* MyISAM에서는 이미 INSERT 된 것을 그대로 두고 쿼리 실행을 종료해 버린다.
* MEMORY 스토리지 엔진을 사용하는 테이블도 동일하게 동작

InnoDB
* 쿼리 중 일부라도 오류가 발생하면 전체를 원 상태로 만든다는 트랜잭션의 원칙대로 실행 전 상태로 복구

MyISAM 에서 발생한 현상을 부분 업데이트 (Partial Update)라고 표현
* 테이블 데이터의 정합성을 맞추는데 상당히 어려운 문제를 만들어 냄

2개 이상의 쿼리가 실행되는 경우 실패에 대한 재처리 작업
~~~sql
INSERT INTO tab_a ...;
IF(_isinsert1_succeed){
    INSERT INTO tab_b ...;
    IF (_is_insert2_succeed) {
        // 처리 완료
    } ELSE {
        DELETE FROM tab_a WHERE ...;
        IF (_is_delete_succeed) {
            // 처리 실패 및 tab_a, tab_b 모두 원상 복구 완료
        } ELSE {
            // 해결 불가능한 심각한 상황 발생
            // 이제 어떻게 해야 하나?
            // tab_b에 INSERT는 안 되고, 하지만 tab_a에는 INSERT 되어 버렸는데, 삭제는 안 되고...
        }
    }
}
~~~
트랜잭션이 지원되지 않는다면 위처럼 복잡해진다.


트랜잭션이 지원되는 InnoDB라면 깔끔
~~~sql
try {
    START TRANSACTION;
    INSERT INTO tab_a ...;
    INSERT INTO tab_b ...;
    COMMIT;
} catch (exception) {
    ROLLBACK;
}
~~~

### 5.1.2 주의사항
프로그램 코드에서 트랜잭션의 범위를 최소화하자.

사용자가 게시판에 게시물을 작성한 후 저장 버튼을 클릭했을 때의 예시
~~~
1) 처리 시작
    ==> 데이터베이스 커넥션 생성
    ==> 트랜잭션 시작
2) 사용자의 로그인 여부 확인
3) 사용자의 글쓰기 내용의 오류 여부 확인
4) 첨부로 업로드된 파일 확인 및 저장
5) 사용자의 입력 내용을 DBMS에 저장
6) 첨부 파일 정보를 DBMS에 저장
7) 저장된 내용 또는 기타 정보를 DBMS에서 조회
8) 게시물 등록에 대한 알림 메일 발송
9) 알림 메일 발송 이력을 DBMS에 저장
    <== 트랜잭션 종료(COMMIT)
    <== 데이터베이스 커넥션 반납
10) 처리 완료
~~~
위 처리 절차 중 트랜잭션 처리에 좋지 않은 영향을 미치는 부분

1번
* 많은 개발자들이 1,2 사이에서 트랜잭션을 시작, 9,10 사이에서 COMMIT하고 커넥션을 종료
* 실제로 DBMS에 데이터를 저장하는 작업은 5번부터 - 2, 3, 4번이 아무리 빨리 끝난다 해도 트랜잭션에 포함시킬 필요는 없다.
* 데이터 베이스의 커넥션은 개수가 제한적이어서 각 단위 프로그램이 커넥션을 소유하는 시간이 길어질수록 사용 가능한 여유 커넥션은 줄어든다. - 어느 순간에는 각 단위 프로그램에서 커넥션을 기다리는 상황이 생길 수 있다.

2번
* 8번 작업, 네트워크를 통한 통신 작업은 어떻게 해서든 트랜잭션 내에서 제거하는 것이 좋다.
* 프로그램이 실행되는 동안 통신이 불가능한 상황이 생기면 DBMS 서버까지 위험해지는 상황이 발생할 수 있다.

3번
* 위 처리 절차에는 DBMS 작업이 크게 4개가 있다.
* 사용자 입력 정보를 저장하는 5, 6번 작업은 반드시 트랜잭션으로 묶고
* 7번은 단순 읽기 작업이므로 트랜잭션에 포함할 필요가 없다.
* 9번 작업은 성격이 다르기 때문에 이전 트랜잭션과 묶지 않아도 무방 해보인다. - 별도 트랜잭션으로 분리하는 것 추천
* 7번 작업은 단순 조회 작업이므로 트랜잭션을 사용하지 않아도 된다.

위 작업을 보완
~~~
1) 처리 시작
2) 사용자의 로그인 여부 확인
3) 사용자의 글쓰기 내용의 오류 여부 확인
4) 첨부로 업로드된 파일 확인 및 저장
    ==> 데이터베이스 커넥션 생성(또는 커넥션 풀에서 가져오기)
    ==> 트랜잭션 시작
5) 사용자의 입력 내용을 DBMS에 저장
6) 첨부 파일 정보를 DBMS에 저장
    <== 트랜잭션 종료(COMMIT)
7) 저장된 내용 또는 기타 정보를 DBMS에서 조회
8) 게시물 등록에 대한 알림 메일 발송
    ==> 트랜잭션 시작
9) 알림 메일 발송 이력을 DBMS에 저장
    <== 트랜잭션 종료(COMMIT)
    <== 데이터베이스 커넥션 종료(또는 커넥션 풀에 반납)
10) 처리 완료
~~~

결론은 프로그램의 코드가 데이터베이스 커넥션을 가지고 있는 범위와 트랜잭션이 활성화되어 있는 프로그램의 범위를 최소화해야 한다는 것이다.\
또한 네트워크 작업이 있는 경우에는 반드시 트랜잭션에서 배재해야 한다.
* 이런 실수로 인해 DBMS 서버가 높은 부하 상태 또는 위험한 상태에 빠질 수 있다.

## 5.2 MySQL 엔진의 잠금
잠금은 크게 스토리지 엔진 레벨과 MySQL 엔진 레벨로 나눌 수 있다.
* MySQL 엔진은 스토리지 엔진을 제외한 나머지 부분

MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치고, 스토리지 엔진 레벨의 잠금은 스토리지 간 상호 영향을 미치지 않는다.

MySQL 엔진에서 제공하는 잠금 종류
* 테이블 락(Table Lock) : 테이블 데이터 동기화를 위한
* 메타데이터 락(Metadata Lock) : 테이블 구조를 잠그는
* 네임드 락(Named Lock) : 사용자의 필요에 맞게 사용할 수 있는

잠금의 특징과 어떤 경우에 사용되는지 알아보자.

### 5.2.1 글로벌 락
글로벌 락(GLOBAL LOCK)
* FLUSH TABLES WITH READ LOCK 명령으로 획득 가능
* MySQL에서 제공하는 잠금 가운데 가장 범위가 크다.

한 세션에서 글로벌 락을 획득하면
* 다른 세션에서 SELECT를 제외한 대부분의 DLL 문장이나 DML 문장을 실행하는 경우
* 해제될 때까지 대기 상태로 남는다.

글로벌 락이 영향을 미치는 범위는 MySQL 서버 전체
* 작업 대상 테이블이나 데이터베이스가 다르더라도 영향

여러 데이터베이스에 존재하는 MyISAM이나 MEMORY 테이블에 대해 mysqldump로 일관된 백업을 받아야할 때 글로벌 락을 사용


주의
* FLUSH TABLES WITH READ LOCK 명령은 실행과 동시에 MySQL 서버에 존재하는 모든 테이블을 닫고 잠금을 건다.
* 실행 전 테이블이나 레코드에 쓰기 잠금을 거는 SQL이 실행됐다면 먼저 실행된 것이 완료될 때까지 대기
* 글로벌 락 이전에 먼저 테이블을 플러시 해야하기 때문에 실행중인 모든 쿼리가 완료되어야 한다.
* 따라서 장시간 SELECT 쿼리가 실행되는 동안 글로벌 락 명령은 대기하게 된다.
* 이 경우 MySQL 서버의 모든 테이블에 대한 INSERT, UPDATE, DELETE 쿼리가 장시간 대기하게 될 수 있다.
* 따라서 서비스 MySQL 서버에서는 가급적 사용하지 않는 것이 좋다.
* mysqldump와 같은 백업 프로그램은 내부적으로 글로벌 락을 걸 수도 있다. - (백업 작업 시 어떤 잠금을 거는지 확인 필요)


InnoDB의 경우 트랜잭션을 지원하기 때문에 모든 데이터 변경 작업을 멈출 필요는 없다.
* 조금 더 가벼운 글로벌 락의 필요성
* MySQL 8.0 버전부터는 백업 툴들의 안정적인 실행을 위해 백업 락 도입
~~~sql
mysql> LOCK INSTANCE FOR BACKUP;
--// 백업 실행
mysql> UNLOCK INSTANCE;
~~~

특정 세션에서 백업 락을 획득하면 모든 세션에서 다음의 작업이 불가능하다.
* 데이터베이스 및 테이블 등 모든 객체 생성 및 변경, 삭제
* REPAIR TABLE과 OPTIMIZE TABLE 명령
* 사용자 관리 및 비밀번호 변경

백업 락은 일반적인 테이블의 데이터 변경은 허용된다.

일반적인 MySQL 서버의 구성은 소스 서버(Source server)와 레플리카 서버(Replica server)로 구성
* 백업 락은 레플리카 서버에서 실행

하지만 백업이 글로벌 락을 획득하면 복제는 백업 시간만큼 지연될 수밖에 없다.
* 레플리카 서버에서 백업을 실행하는 도중에 소스 서버에 문제가 생기면 레플리카 서버의 데이터가 최신 상태가 될 때까지 서비스를 멈춰야할 수 있음

XtraBackup, Enterprise Backup 툴들은 모두 복제가 진행되는 상태에서도 일관된 백업을 만들 수 있다.
* 백업 툴이 실행되는 도중 스키마 변경이 실행되면 백업은 실패하게 된다.
* 6~7시간 백업 도중 DDL 명령 하나로 백업 실패 -> 다시 처음부터 백업

MySQL 백업 락은 정상적으로 복제는 실행되지만 백업의 실패를 막기 위해 DLL 명령이 실행되면 복제를 일시 중지하는 역할을 한다.


### 5.2.2 테이블 락
테이블 락(Table Lock)
* 개별 테이블 단위로 설정되는 잠금
* 명시적 또는 묵시적으로 획득 가능

명시적으로 획득 / 해제
* LOCK TABLES table_name [ READ | WRITE ] : 획득
* UNLOCK TABLES : 해제

명시적 테이블 락은 특별한 상황이 아니면 거의 사용하지 않는다.
* 글로벌 락과 마찬가지로 온라인 작업에 상당한 영향을 미치기 때문

묵시적인 테이블 락
* MyISAM이나 MEMORY 테이블에 데이터를 변경하는 쿼리를 실행하면 발생
* 쿼리가 실행되는 동안 자동 획득, 완료된 후 자동 해제
* InnoDB에서는 레코드 기반의 잠금을 제공하기 때문에 묵시적 테이블 락이 설정되지 않음
* 정확히는 대부분의 DML 쿼리에서는 무시되고 DLL 쿼리에서 테이블 락이 영향을 미친다.


### 5.2.3 네임드 락
네임드 락(Named Lock)
* GET_LOCK() 함수를 이용해 임의의 문자열에 대해 잠금을 설정할 수 있다.
* 잠금의 대상은 사용자가 지정한 문자열(string)이다.
* 여러 클라이언트가 상호 동기화를 처리해야 할 때 네임드 락을 이용하면 쉽게 해결 가능

~~~sql
--// "mylock" 이라는 문자열에 대해 잠금을 획득한다.
--// 이미 잠금을 사용 중이면 2초 동안만 대기한다. (2초 이후 자동 잠금 해제됨)
mysql> SELECT GET_LOCK('mylock', 2);

--// "mylock"이라는 문자열에 대해 잠금이 설정돼 있는지 확인한다.
mysql> SELECT IS_FREE_LOCK('mylock');

--// "mylock"이라는 문자열에 대해 획득했던 잠금을 반납(해제)한다.
mysql> SELECT RELEASE_LOCK('mylock');

--// 3개 함수 모두 정상적으로 락을 획득하거나 해제한 경우에는 1을,
--// 아니면 NULL이나 0을 반환한다.
~~~

네임드 락은 많은 레코드에 대해서 복잡한 요건으로 레코드를 변경하는 트랜잭션에 유용하게 사용할 수 있다.

ex) 배치 프로그램처럼 한꺼번에 많은 레코드를 변경하는 쿼리 -> 데드락의 원인
* 이러한 경우 동일 데이터를 변경하거나 참조하는 프로그램끼리 분류해서 네임드 락을 걸고 쿼리를 실행하면 간단히 해결 가능

MySQL 8.0 버전부터는 네임드 락을 중첩해서 사용할 수 있다.
~~~sql
mysql> SELECT GET_LOCK('mylock_1', 10);
--// mylock_1에 대한 작업 실행
mysql> SELECT GET_LOCK('mylock_2', 10);
--// mylock_1과 mylock_2에 대한 작업 실행

mysql> SELECT RELEASE_LOCK('mylock_2');
mysql> SELECT RELEASE_LOCK('mylock_1');

--// mylock_1과 mylock_2를 동시에 모두 해제하고자 한다면
mysql> SELECT RELEASE_ALL_LOCKS();
~~~


### 5.2.4 메타데이터 락
메타데이터 락(Metadata Lock)
* 데이터베이스 객체(테이블이나 뷰 등)의 이름이나 구조를 변경하는 경우에 획득
* 명시적으로 획득, 해제가 불가능하다.
* RENAME TABLE 등의 명령을 사용하는 경우 자동으로 획득하는 잠금
* RENAME TABLE의 경우 원본 이름과 변경될 이름 모두 잠금 설정

실시간으로 테이블을 바꿔야 하는 요건이 배치 프로그램에서 자주 발생
~~~sql
--// 배치 프로그램에서 별도의 임시 테이블(rank_new)에 서비스용 랭킹 데이터를 생성

--// 랭킹 배치가 완료되면 현재 서비스용 랭킹 테이블(rank)을 rank_backup으로 백업하고
--// 새로 만들어진 랭킹 테이블(rank_new)을 서비스용으로 대체하고자 하는 경우
mysql> RENAME TABLE rank TO rank_backup, rank_new TO rank;
~~~
* RENAME TABLE 명령문에 두 개의 RENAME 작업을 한꺼번에 실행하면 "Table not found 'rank'" 같은 상황을 발생시키지 않고 적용 가능
* 2개로 나눠서 실행 시 아주 짧은 rank 테이블이 존재하지 않는 순간이 생긴다.
* 그 순간 실행되는 쿼리는 "Table not found 'rank'" 오류 발생
~~~sql
RENAME TABLE rank TO rank_backup;
RENAME TABLE rank_new TO rank;
~~~

메타데이터 잠금과 InnoDB 트랜잭션을 동시에 사용해야 하는 경우
* 예를들어 INSERT만 실행되는 로그 테이블 - 저장만 하기 때문에 UPDATE와 DELETE가 없다.

~~~sql
mysql> CREATE TABLE access_log (
        id BIGINT NOT NULL AUTO_INCREMENT,
        client_ip INT UNSIGNED,
        access_dttm TIMESTAMP,
        ...
        PRIMARY KEY (id)
       );
~~~

그런데 테이블의 구조를 변경해야 할 요건이 발생
* Online DDL을 이용하여 변경 - 언두 로그의 증가와, Online DDL이 실행되는 동안 누적된 버퍼의 크기 문제
* DDL은 단일 스레드로 작동 - 많은 시간 소모

이때는 새로운 구조의 테이블을 생성하고 먼저 최근(1시간 직전 또는 하루 전)의 데이터까지는 프라이머리 키인 id 값을 범위별로 나눠 여러 개의 스레드로 빠르게 복사
~~~sql
--// 테이블의 압축을 적용하기 위해 KEY_BLOCK_SIZE=4 옵션을 사용해 신규 테이블 생성
mysql> CREATE TABLE access_log_new (
        id BIGINT NOT NULL AUTO_INCREMENT,
        client_ip INT UNSIGNED,
        access_dttm TIMESTAMP,
        ...
        PRIMARY KEY (id)
       ) KEY_BLOCK_SIZE=4;

--// 4개의 스레드를 이용해 id 범위별로 레코드를 신규 테이블로 복사
mysql_thread1> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=0 AND id<10000;
mysql_thread2> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=10000 AND id<20000;
mysql_thread3> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=20000 AND id<30000;
mysql_thread4> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=30000 AND id<40000;
~~~

나머지 데이터는 다음과 같이 트랜잭션과 테이블 잠금, RENAME TABLE 명령으로 응용 프로그램의 중단 없이 실행할 수 있다.
* 이때 "남은 데이터를 복사"하는 시간 동안은 테이블 잠금 - INSERT 불가
* 가능하면 아주 최근 데이터까지 미리 복사해둬야 잠금 시간을 최소화해 서비스에 미치는 영향을 줄일 수 있다.
~~~sql
--// 트랜잭션을 autocommit으로 실행(BEGIN이나 START TRANSACTION으로 실행하면 안 됨)
mysql> SET autocommit=0;

--// 작업 대상 테이블 2개에 대해 테이블 쓰기 락을 획득
mysql> LOCK TABLES access_log WRITE, access_log_new WRITE;

--// 남은 데이터를 복사
mysql> SELECT MAX(id) as @MAX_ID FROM access_log_new;
mysql> INSERT INTO access_log_new SELECT * FROM access_log WHERE pk>@MAX_ID;
mysql> COMMIT;

--// 새로운 테이블로 데이터 복사가 완료되면 RENAME 명령으로 새로운 테이블을 서비스로 투입
mysql> RENAME TABLE access_log TO access_log_old, access_log_new TO access_log;
mysql> UNLOCK TABLES;

--// 불필요한 테이블 삭제
mysql> DROP TABLE access_log_old;
~~~


## 5.3 InnoDB 스토리지 엔진 잠금
InnoDB 스토리지 엔진은 MySQL에서 잠금과는 별개로 레코드 기반의 잠금 방식을 탑재
* 이것으로 훨씬 뛰어난 동시성 처리 제공
* 하지만 이원화된 잠금 처리 탓에 MySQL 명령을 이용해 접근하기가 까다롭다.

예전 버전 MySQL 서버에서 InnoDB 잠금 정보를 진단할 수 있는 도구
* innodb_lock_monitor라는 이름의 테이블을 생성해서 잠금 정보를 덤프하는 방법
* SHOW ENGINE INNODB STATUS 명령이 전부
* 위 내용은 분석하기 상당히 어려웠다.

최근 버전에서는 InnoDB의 트랜잭션과 잠금, 그리고 잠금 대기 중인 트랜잭션의 목록을 조회할 수 있는 방법이 도입됐다.
* information_schema 데이터베이스에 존재하는 INNODB_TRX, INNODB_LOCK, INNODB_LOCK_WAITS 테이블을 조인에서 조회
* Performance Schema를 이용해 InnoDB 스토리지 엔진의 내부 잠금(세마포어)에 대한 모니터링 방법도 추가

### 5.3.1 InnoDB 스토리지 엔진의 잠금
InnoDB 스토리지 엔진은 레코드 기반의 잠금 기능을 제공
* 상당히 작은 공간으로 관리 - 레코드 락 -> 페이지 락 -> 테이블 락으로 레벨업 되는 경우는 없다.(락 에스컬레이션)
* 레코드와 레코드 사이의 간격을 잠그는 갭(GAP) 락또 한 존재한다.

![InnoDB_Lock](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/ec43b4e2-ea27-4356-959e-be0181446c61)


### 5.3.1.1 레코드 락
레코드 락(Record lock, Record only lock)
* 레코드 자체만을 잠그는 것 - 다른 DBMS와 동일한 역할
* 한가지 차이 : InnoDB 스토리지 엔진은 레코드 자체가 아닌 인덱스의 레코드를 잠금
  * 인덱스가 없더라도 자동 생성된 클러스터 인덱스을 이용해 설정
  * 레코드를 잠그느냐와 인덱스를 잠그느냐는 중요한 차이를 만들어 냄(이후 다시 예제)

InnoDB에서는 대부분 보조 인덱스를 이용한 변경 작업은
* 넥스트 키 락 또는 갭 락 사용

프라이머리 키 또는 유니크 인덱스에 의한 변경 작업은
* 레코드 자체에 대해서만 락


### 5.3.1.2 갭 락
갭 락(Gap lock)
* 레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만 잠그는 것
* 중간에 INSERT되는 것을 제어하는 것
* 넥스트 키 락의 일부로 자주 사용된다.

### 5.3.1.3 넥스트 키 락
넥스트 키 락(Next key lock)
* 레코드 락과 갭 락을 합쳐 놓은 형태의 잠금

STATEMENT 포맷의 바이너리 로그를 사용하는 MySQL 서버에서는 REPEATABLE READ 격리수준을 사용\
또한 innodb_locks_unsafe_for_binlog 시스템 변수가 비활성화되면(0 설정) 변경을 위해 검색하는 레코드에는 넥스트 키 락 방식으로 잠금이 걸린다.

InnoDB의 갭 락과 넥스트 키 락의 목적
* 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것

그런데 의외로 넥스트 키 락과 갭락으로 인해 데드락이 발생하거나 다른 트랜잭션이 대기하게 만드는 일이 자주 발생
* 가능하면 바이너리 로그 포맷을 ROW 형태로 바꿔서 넥스트 키락과 갭 락을 줄이는 것이 좋다.

MySQL8.0에서는 ROW 포맷의 바이너리 로그가 기본 설정

### 5.3.1.4 자동 증가 락
AUTO_INCREMENT라는 칼럼 속성을 지정하면 중복되지 않고 저장된 순서대로 증가하는 일련번호 값을 가지게 된다.
* 이를 위해 내부적으로 자동 증가 락이라고 하는 테이블 수준 잠금을 사용

AUTO_INCREMENT 락 특징
* 새로운 레코드를 저장하는 쿼리에서만 필요(INSERT, REPLACE)
* InnoDB의 다른 잠금과는 달리 AUTO_INCREMENT 값을 가져오는 순간에만 락이 걸린다.
  * AUTO_INCREMENT의 값을 명시적으로 변경해도 걸린다.
* AUTO_INCREMENT 락을 명시적으로 획득하고 해제하는 방법은 없다.
* 아주 짧은 시간 적용되기 때문에 특별히 문제 생기지 않는다.

지금까지의 설명은 MySQL5.0 이하 버전에서 사용되던 방식

MySQL 5.1부터는 innodb_autoinc_lock_mod 시스템 변수로 작동 방식 변경 가능
* =0 : 5.0과 동일한 잠금 방식
* =1 : INSERT 되는 레코드 건수를 정확히 예측할 수 있으면 훨씬 가볍고 빠른 래치(뮤텍스)를 이용해 처리
  * 래치는 자동 증가 값을 가져오는 즉시 잠금이 해제
  * 대량의 INSERT가 수행될 때는 여러개의 자동 증가 값을 한 번에 할당받아 INSERT되는 레코드에 사용
  * 하지만 한 번에 할당받은 값이 남는다면 폐기되고 다음 INSERT되는 레코드는 연속되지 않은 값을 가질 수 있다.
  * 최소한 하나의 INSERT 문장으로 INSERT 되는 레코드는 연속된 자동 증가 값을 가지게 된다.
  * 연속 모드(Consecutive mode)라고도 한다.
* =2 : 무조건 래치(뮤텍스) 사용
  * 연속된 자동 증가 값을 보장하진 않는다.
  * 인터리빙 모드(Interleaced mode)라고도 한다.
  * 대량 INSERT 문장이 실행되는 중에도 다른 커넥션에서 INSERT를 수행할 수 있다.(동시 처리 성능 높아짐)
  * 하지만 이 설정에서 작동하는 자동 증가 기능은 유니크 값이 생성된다는 것만 보장

### 5.3.2 인덱스와 잠금
InnoDB의 잠금은 레코드를 잠그는 것이 아닌 인덱스를 잠그는 방식

~~~sql
--// 예제 데이터베이스의 employees 테이블에는 아래와 같이 first_name 칼럼만
--// 멤버로 담긴 ix_firstname이라는 인덱스가 준비돼 있다.
--//    KEY ix_firstname (first_name)
--// employees 테이블에서 first_name='Georgi'인 사원은 전체 253명이 있으며,
--// first_name='Georgi'이고 last_name='Klassen'인 사원은 딱 1명만 있는 것을
--// 아래 쿼리로 확인할 수 있다.
mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi';
+----------+
|       253|
+----------+

mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi' AND last_name='Klassen';
+----------+
|         1|
+----------+

--// employees 테이블에서 first_name='Georgi'이고 last_name='Klassen'인 사원의
--// 입사 일자를 오늘로 변경하는 쿼리를 실행해보자.
mysql> UPDATE employees SET hire_data=NOW() WHERE first_name='Georgi' AND last_name='Klassen';
~~~
위 1건의 업데이트를 위해 몇 개의 레코드에 락을 걸어야 할까?
* 인덱스를 이용할 수 있는 조건은 first_name='Georgi'이므로 253건의 레코드에 대해 락이 걸린다.
* 이 예제에서는 몇 건 안되는 레코드만 잠그지만, 적절한 인덱스가 준비되어 있지 않다면 동시성이 상당히 떨어질 것

![index_lock](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/b36d12d3-1fe4-4e2d-84a1-749ad6da3d15)

테이블에 인덱스가 하나도 없다면?
* 모든 레코드를 잠그게 된다.
* InnoDB에서 인덱스 설계가 중요한 이유가 바로 이것

### 5.3.3 레코드 수준의 잠금 확인 및 해제
레코드 수준 잠금은 테이블 수준 잠금보다는 조금 더 복잡
* 테이블 잠금에서는 문제의 원인이 쉽게 발견되고 해결될 수 있다.
* 레코드 잠금에서는 그 레코드가 자주 사용되지 않는다면 오랜 시간 잠겨진 상태로 남아 있어도 발견하기 쉽지 않다.

예전 버전 MySQL 서버에서는 레코드 잠금에 대한 메타 정보를 제공하지 않기 때문에 더욱 어려웠다.

MySQL5.1 부터는 레코드 잠금과 잠금 대기에 대한 조회가 가능하므로 쿼리 하나로 확인할 수 있다.
* 강제로 잠금을 해제하려면 KILL 명령을 이용해 MySQL 서버의 프로세스를 강제로 종료하면 된다.

잠금 시나리오 가정

![LS](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/c48a5013-4329-4d52-8e6c-7cc68e16bda6)

각 트랜잭션이 잠금 대기, 어떤 트랜잭션이 하고 있는지 쉽게 조회할 수 있다.
* information_schema 라는 DB에 INNODB_TRX, INNODB_LOCKS, INNODB_LOCK_WAITS 테이블로 확인 가능 - 5.1 버전
* 8.0 버전부터 이런 정보는 조금씩 제거되고 있다.
* performance_schema의 data_locks와 data_lock_waits 테이블로 대체

performance_schema의 data_locks 테이블과 data_lock_waits 테이블 조회하여 대기순서 조회
~~~sql
mysql> SELECT
        r.trx_id waiting_trx_id,
        r.trx_mysql_thread_id waiting_thread,
        r.trx_query waiting_query,
        b.trx_id blocking_trx_id,
        b.trx_mysql_thread_id blocking_thread,
        b.trx_query blocking_query
       FROM performance_schema.data_lock_waits w
       INNER JOIN information_schema.innodb_trx b
        ON b.trx_id = w.blocking_engine_transaction_id
       INNER JOIN information_schema.innodb_trx r
        ON r.trx_id = w.blocking_engine_transaction_id;
~~~

위 쿼리의 실행결과를 보면 몇 번 스레드가 현재 대기 중인지 알 수 있다.\
추가로 몇 번 스레드에 의해 대기 중인지 정보도 표시된다.

만약 19번 스레드가 17번 스레드의 잠금을 기다리고 있을 때\
17번 스레드가 어떤 잠금을 가지고 있는지 상세히 확인하고 싶다면 performance_schema의 data_locks 테이블이 가진 컬럼을 모두 확인하면 된다.
~~~sql
mysql> SELECT * FROM performance_schema.data_locks\G
~~~

![pl1](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/ae18477f-2754-4b37-a57e-29e05e9d0dad)

![pr2](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/13f1de0f-787c-4d03-bd2a-dcde80554891)

위 결과에서 employees 테이블에 대해 IX잠금(Intentional Exclusive)을 가지고 있으며 특정 레코드에 대해 쓰기 잠금을 가지고 있다는 것을 확인 가능
* REC_NOT_GAP 표시로 갭이 포함되지 않은 순수 레코드에 대한 잠금을 가지고 있음을 알 수 있다.

만약 이 상황에서 17번 스레드가 상당히 오래 멈춰 있었다면 강제로 종료시키면 된다.
~~~sql
mysql> KILL 17;
~~~

## 5.4 MySQL의 격리 수준
트랜잭션의 격리 수준(isolation level)이란?
* 여러 트랜잭션이 동시에 처리될 때 특정 트랜잭션이 다른 트랜잭션에게 변경하거나 조회하는 데이터를 볼 수 있게 허용 여부를 결정하는 것

격리 수준 4가지
* READ UNCOMMITTED
  * "DIRTY READ"라고도 불린다.
  * 일반적인 데이터베이스에서는 거의 사용하지 않는다.
* READ COMMITTED
* REPEATABLE READ
* SERIALIZABLE
  * 동시성이 중요한 데이터베이스에서는 사용되지 않는다.

순서대로 뒤로 갈수록 격리 정도가 높아지며, 동시 처리 성능도 떨어지는 것이 일반적
* SERIALIZABLE 격리 수준만 아니라면 크게 성능의 개선이나 저하는 발생하지 않는다.

부정합의 문제 발생 여부

![isolation_level01](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/1a5fa5d4-563e-4ec3-94d2-6e1b8eab7354)

일반적인 온라인 서비스 용도의 데이터베이스는 아래의 둘 중 하나를 사용
* READ COMMITTED : 오라클에서 주로 사용
* REPEATABLE READ : MySQL에서 주로 사용

여기서 설명하는 모든 예제는 AUTOCOMMIT이 OFF인 상태에서만 테스트할 수 있다.
* SET autocommit=OFF

### 5.4.1 READ UNCOMMITTED

READ UNCOMMITTED 격리 수준에서는 각 트랜잭션에서의 변경 내용이 COMMIT이나 ROLLBACK 여부에 상관없이 다른 트랜잭션에서 보인다.

다른 트랜잭션이 사용자 B가 실행하는 SELECT 쿼리의 결과에 어떤 영향을 미치는지 보여주는 예제

![READ_UNCOMMITTED](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/c370b737-a417-4b4b-90d0-8cad05be6504)

사용자 B는 사용자 A가 변경한 내용을 커밋하지 않은 상태에서도 조회가 가능하다.
* 문제는 사용자 A가 처리 중 롤백한다해도 여전히 사용자 B는 정상적인 줄 알고 계속 처리할 것

더티 리드(Dirty Read)
* 이처럼 어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데 다른 트랜잭션에서 볼 수 있는 현상
* 이것이 허용되는 격리 수준 READ UNCOMMITTED
* 데이터가 나타났다가 사라졌다 하는 현상을 초래 : 혼란스럽게 만든다.

정합성에 문제가 많은 격리 수준이다.
* READ COMMITTED 이상의 격리 수준을 사용할 것을 권장

### 5.4.2 READ COMMITTED
READ COMMITTED는 오라클에서 기본으로 사용되는 격리 수준
* 온라인 서비스에서 가장 많이 선택된다.

더티 리드(Dirty Read) 같은 현상은 발생하지 않는다.
* COMMIT이 완료된 데이터만 다른 트랜잭션에서 조회 가능


READ COMMITTED 격리 수준에서 사용자 A가 변경한 내용이 사용자 B에게 어떻게 조회되는지 보여주는 예제

![READ_COMMITTED01](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/69bb6c78-1f2f-4329-8a42-12f409888095)

사용자 B의 SELECT 결과는 언두 영역에 백업된 레코드에서 가져온다.
* 커밋되기 전까지 다른 트랜잭션에서 변경 내역을 조회할 수 없기 때문

"NON-REPEATABLE READ"라는 부정합의 문제가 있다.

"NON-REPEATABLE READ"가 왜 발생하고 어떤 문제를 만들어낼 수 있는지 보여주는 예제

![READ_COMMITTED02](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/4116e6e1-bb34-45ae-aa09-e00881924b03)

사용자 B가 하나의 트랜잭션 내에서 똑같은 SELECT 결과를 가져와야 한다는 "REPEATABLE READ" 정합성에 어긋난다.
* 이런 부정합은 일반적인 웹 프로그램에서는 크게 문제가 되지 않을 수 있지만
* 하나의 트랜잭션에서 동일 데이터를 여러 번 읽고 변경하는 작업이 금전적인 처리와 연결되면 문제가 될 수 있다.
* 중요한 것은 사용 중인 트랜잭션의 격리 수준에 의해 실행하는 SQL 문장이 어떤 결과를 가져올지 정확히 예측할 수 있어야 한다는 것

트랜잭션 내에서 실행되는 SELECT와 없이 실행되는 SELECT 문장의 차이
* READ COMMITTED 격리 수준에서
  * 차이가 별로 없다.
* REPEATABLE READ 격리 수준에서
  * 트랜잭션을 시작한 상태에서 온종일 동일한 쿼리를 반복해서 실행해도 동일한 결과만 보게 된다.

별로 중요해 보이지 않겠지만 이런 문제로 데이터의 정합성이 깨지고 그로 인한 버그는 찾아내기 쉽지 않다.


### 5.4.3 REPEATABLE READ
REPEATABLE READ는 InnoDB 스토리지 엔진에서 기본으로 사용되는 결리 수준
* 바이너리 로그를 가진 MySQL 서버에서는 최소 REPEATABLE READ 격리 수준 이상을 사용해야 한다.
* 부정합이 발생하지 않는다.

InnoDB 스토리지 엔진은 트랜잭션이 ROLLBACK될 가능성에 대비해 변경 전 레코드를 언두(Undo) 공간에 백업해두고 실제 레코드 값을 변경한다. MVCC
* 언두 영역의 데이터를 이용해 동일 트랜잭션 내에서는 동일한 결과를 보여줄 수 있게 보장한다.

REPEATABLE READ와 READ COMMITTED의 차이는 언두 영역에 백업된 레코드의 여러 버전 중 몇 번째 이전 버전까지 찾아들어가냐이다.

모든 InnoDB의 트랜잭션은 고유한 번호를 가지고 있다.
* 언두 영역에 백업된 모든 레코드에는 이것을 포함

언두 영역의 데이터는 InnoDB 스토리지 엔진이 불필요하다 판단하면 주기적으로 삭제
* REPEATABLE READ 격리 수준에서는 MVCC를 보장하기 위해 실행 중인 트랜잭션 중 가장 오래된 트랜잭션 번호보다 트랜잭션 번호가 앞선 영역의 데이터는 삭제할 수 없다.
* 특정 트랜잭션 번호 구간 내에 백업된 언두 데이터가 보존되어야 한다.


REPEATABLE READ 격리 수준이 작동하는 방식

![REPEATED_READ01](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/2a872bd2-c02a-48b9-a179-feb3efe099c5)


언두 영역에 백업된 데이터가 하나만 있는 것으로 표현됐지만 여러 개 존재할 수 있다.
* 트랜잭션을 시작하고 장시간 방치하면 무한정 커질 수 있다.
* 이 경우 MySQL 서버의 처리 성능이 떨어질 수 있다.

REPEATABLE READ 격리 수준에서 발생하는 부정합

![REPEATED_READ02](https://github.com/MyungHyun-Ahn/SystemProgramming/assets/78206106/227e4e67-d223-4026-be41-282273b68416)

같은 트랜잭션에서 두 번의 SELECT 쿼리는 똑같아야 한다.
* 그러나 사용자 B의 SELECT 쿼리의 결과는 서로 다르다.
* 이렇게 데이터가 보였다 안보였다 하는 현상을 PHANTOM READ(ROW) 라고 한다.
* 언두 레코드에는 잠금을 걸 수 없기 때문

### 5.4.4 SERIALIZABLE
가장 단순한 격리 수준이며 동시에 가장 엄격한 격리 수준
* 그만큼 동시 처리 성능도 다른 트랜잭션 격리 수준보다 떨어짐

InnoDB 테이블에서 기본적으로 순수한 SELECT 작업은 아무런 레코드 잠금도 설정하지 않고 실행
* "Non-locking consistent read(잠금이 필요 없는 일관된 읽기)"는 이를 의미

SERIALIZABLE 격리 수준에서는 이런 읽기 작업도 공유 잠금을 획득해야만 하고, 다른 트랜잭션은 변경하지 못한다.
* 즉, 한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서는 절대 접근할 수 없는 것

PHANTOM READ 문제가 발생하지 않는다.
* 그러나 InnoDB 스토리지 엔진에서 갭 락과 넥스트 키 락 덕분에 REPEATABLE READ 격리 수준에서도 이미 발생하지 않는다.
* 굳이 SERIALIZABLE 격리 수준을 사용할 필요성이 없다.