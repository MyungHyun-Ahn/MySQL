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