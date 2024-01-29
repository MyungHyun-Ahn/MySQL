# Chap 02 설치와 설정
## 2.1 MySQL 서버 설치
다양한 방법으로 설치 가능
* 그러나 리눅스의 RPM이나 운영체제 별 인스톨러 권장

### 2.1.1 버전과 에디션(엔터프라이즈와 커뮤니티) 선택
버전을 선택할 경우 다른 제약 사항(기존 솔루션이 특정 버전만 지원하는 경우)이 없다면 가능한 최신 버전 설치
* 기존 버전에서 새로운 메이저 버전으로 업그레이드 하는 경우라면 최소 패치 버전이 15~20번 이상 릴리스된 버전을 선택하는 것이 안정적인 서비스에 도움
* 즉, MySQL 8.0 버전이라면 8.0.15 ~ 8.0.20 추천

갓 출시된 메이저 버전을 선택하는 것은 위험하다.
* 치명적이거나 보완하는 데 많은 시간이 걸릴 만한 버그가 있을 수도 있다.

엔터프라이즈와 커뮤니티 에디션
* MySQL 상용화 전략의 핵심은 두 에디션이 동일
* 엔터프라이즈 버전에 특정 부가 기능이 포함
* 이런 상용화 방식을 Open Core Model이라고 함

엔터프라이즈 에디션에서만 지원하는 것
* Thread Pool
* Enterprise Audit
* Enterprise TDE (Master Key 관리)
* Enterprise Authentication
* Enterprise Firewall
* Enterprise Monitor
* Enterprise Backup
* MySQL 기술지원

Percona에서 출시하는 Percona Server에서 지원하는 플러그인 등을 활용하면 커뮤티니 버전의 부족함을 메꿀 수 있었다.
* 하지만 기술지원은 별개의 문제
* 엔터프라이즈 에디션에서 지원하는 것들이 꼭 필요한지 검토해보는 것이 좋다.

### 2.1.2 MySQL 설치
전 윈도우만 할겁니다.
### 윈도우 MSI 인스톨러 설치
https://dev.mysql.com/downloads/installer/
* 위 사이트에서 설치
* 최신 버전

Installer 설정 순서
1. Custom
2. MySQL Server(클라이언트 포함), Shell, Router 포함시키고 Next
3. 설치하고 Next
4. Config - Development Computer, 포트 Default
   * Development Computer를 선택하면 서버가 허용하는 커넥션의 개수를 적게 설정함
5. 사용자 로그인 시점에 사용할 비밀번호 인증 방식
   * Strong Password Encryption : SHA-2 Auth 플러그인 사용
   * Legacy Auth Method : Native Auth 플러그인 사용 - 지금은 이것을 선택
6. 비밀번호 설정하고 Next
7. 설정 적용
8. 라우터 설정은 Cancel


MySQL 서버의 프로그램과 설정 파일 위치는 MySQL80 서비스의 등록 정보를 통해 확인 가능하다.
* "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqld.exe" --defaults-file="C:\ProgramData\MySQL\MySQL Server 8.0\my.ini" MySQL80
* 설정 파일 위치 - C:\ProgramData\MySQL\MySQL Server 8.0\my.ini

MySQL 서버 디렉터리의 구조
* bin : MySQL 서버와 클라이언트 프로그램, 그리고 유틸리티를 위한 디렉터리
* include : C/C++ 헤더 파일들이 저장된 디렉터리
* lib : 라이브러리 파일들이 저장된 디렉터리
* share : 다양한 지원 파일들이 저장, 에러 메시지나 샘플 설정 파일이 있는 디렉터리


## 2.2 MySQL 서버의 시작과 종료
Windows에서는 이미 서버가 켜진 상태
* MySQL 8.0 Command Line Client - Unicode 실행하고 비밀번호 입력

SHOW DATABASES; 명령어 입력하면 현재 데이터베이스 목록 보여줌


## 2.3 MySQL 서버 업그레이드
두 가지 방법
1. MySQL 서버의 데이터 파일을 그대로 두고 업그레이드하는 방법
2. mysqldump 도구 등을 이용해 MySQL 서버의 데이터를 SQL 문장이나 텍스트 파일로 덤프한 후, 업그레이드 버전의 서버에 데이터를 적재하는 방법

1번을 In-Place Upgrade라고 하고 2번을 Logical Upgrade라고 한다.


### 2.3.1 인플레이스 업그레이드 제약사항
동일 메이저 버전에서 마이너 버전 간 업그레이드는 대부분 데이터 파일의 변경 없이 진행되며, 많은 경우 여러 버전을 건너뛰어서 업그레이드하는 것도 허용
* ex 8.0.16 -> 8.0.21

메이저 버전 간 업그레이드는 대부분 크고 작은 데이터 파일의 변경이 필요하기 때문에 반드시 직전 버전만 허용
* ex 5.5 -> 5.6 OK, 5.5 -> 5.7 X

만약 5.1 버전을 사용하는데 8.0으로 업그레이드해야 한다면
* 5.1 -> 5.5 -> 5.6 -> 5.7 -> 8.0
* 이 때는 논리적 업그레이드가 더 나은 방법일 수 있다.

인플레이스 업그레이드에서 메이저 버전 업그레이드가 특정 마이너 버전에서만 가능한 경우도 있다.

### 2.3.2 MySQL 8.0 업그레이드 시 고려 사항
5.7과의 차이점
* 사용자 인증 방식 변경
* MySQL 8.0과의 호환성 체크
* 외래키 이름의 길이
* 인덱스 힌트
* GROUP BY에 사용된 정렬 옵션
* 파티션을 위한 공용 테이블스페이스

### 2.3.3 MySQL 8.0 업그레이드
MySQL 5.7에서 MySQL 8.0으로의 업그레이드는 두 가지 단계로 나뉘어 처리된다.
1. 데이터 딕셔너리 업그레이드
2. 서버 업그레이드


## 2.4 서버 설정
윈도우에서는 my.ini

### 2.4.1 설정 파일의 구성


### 2.4.2 MySQL 시스템 변수의 특징
시스템 변수(System Variables) : 서버가 기동되며 필요한 내용을 저장해둔 것


SHOW GLOBAL VARIABLES 명령으로 확인 가능
* 633개가 나온다.

### 2.4.3 글로벌 변수와 세션 변수
MySQL의 시스템 변수는 적용 범위에 따라 글로벌 변수와 세션 변수로 나뉜다.
* 글로벌 변수는 MySQL 서버 인스턴스에서 전체적으로 영향을 미친다.
* 세션 변수는 MySQL 클라이언트가 MySQL 서버에 접속할 때 기본으로 부여하는 옵션의 기본값을 제어하는데 사용
* 세변 변수 중 MySQL 서버의 설정 파일에 명시해 초기화할 수 있는 변수는 대부분 Both라고 명시돼 있다.

### 2.4.4 정적 변수와 동적 변수
MySQL 서버의 시스템 변수는 서버가 기동 중인 상태에서 변경 가능한지에 따라 동적 변수와 정적 변수로 나뉜다.
