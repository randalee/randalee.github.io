---
title: Apache Superset(v1.3) 테스트 1편 - 설치
tags:
- python
- superset
- bi
- 슈퍼셋
toc: true
toc_sticky: true
category: python
---

##  개요
`Self Service BI`란 통계적 분석이나 데이터마이닝과 같은 전문지식이 없는 현업 직원 누구라도 기업데이터에 접속하여 분석 작업을 할 수 있는 것을 의미한다. 따라서 이를 충족하기 위해서 비전공자여도 쉽게 접근 할 수 있는 수많은 제품들이 있다.  이러한 다양한 제품들은 [Top BI Tools Of 2021](https://blog.panoply.io/top-25-business-intelligence-tools-and-how-to-decide)과 같이 정리된 글을 통해 다양하게 찾아 볼 수있다. 국내에서는 상용으로는 [Tableau](https://www.tableau.com/ko-kr), [Power BI](https://powerbi.microsoft.com/ko-kr/)를 오픈소스로는 [Re:dash](https://redash.io/) [Metabase](https://www.metabase.com/) 를 사용하는 기업들이 있다.

[Superset](https://superset.apache.org/)은 이렇게 다양한 제품 중 Apache에서 관리하고 있는 오픈소스로, 시작은 워크플로우 툴로 유명한 [Airflow](https://airflow.apache.org/)를 만든 Airbnb에서 시작한 [프로젝트](https://airbnb.io/projects/superset/)이다. Superset은 Python으로 만들어졌으며, 처음 봤던 초기에는 정말 단순한 기능밖에 없었다. 그래서인지 한국에서는 슈퍼셋에 대한 대략적인 글뿐이 없었고, 실제 설치하고 사용하는 것에 대한 레퍼런스가 많이 부족했다.

그래서 이번 기회에 현재 시점기준으로 가장 최신버전인 v1.3을 설치하며 겪은 오류와 같은 내용을 정리하기 위하여 이 글을 쓴다.
테스트를 진행한 환경은 다음과 같다.

* AWS Amazon linux 2 / m5.xlarge
* 도커가 아닌 Python venv환경을 이용하여 테스트
	* 라이브러리 의존성 등을 체크하고, 동작 구조를 좀더 유연하게 테스트하기 위함
	* Superset을 설치하는 방법은 아래와 같은 방법들이 있다.
		* [pypi apache-superset](https://pypi.org/project/apache-superset/)
		* [apache repo 이용](https://dist.apache.org/repos/dist/release/superset/)
		* [github 설치](https://github.com/apache/superset/tree/latest)
		* docker compose 또는 docker image설치

##  설치를 위한 사전 작업
### 1. Python 3.9 설치
Amazon linux2에는 이미 Python 3.7버전이 설치가 되어있다. 그러나 아쉬움(?)이 존재하여 최신버전인 Python을 직접 설치하였다. 최신버전은 [파이썬 공식홈페이지](https://www.python.org/downloads/)에서 확인 할 수 있으며, 이 글을 쓰는 기준으로 3.9.7버전이 존재하나 테스트 당시에는 3.9.6을 설치하였다. 

`yum`, `apt-get`과 같은 패키지 관리자로 설치하는 경우가 아닌 경우, 직접 컴파일을 진행해야하기에 별로의 라이브러리가 필요하다.

```bash
sudo yum install gcc openssl-devel bzip2-devel libffi-devel sqlite-devel xz-devel

cd /home/ec2-user
mkdir src
wget https://www.python.org/ftp/python/3.9.6/Python-3.9.6.tgz
tar zxvf Python-3.9.6.tgz

cd /home/ec2-user/src/Python-3.9.6/
sudo ./configure --enable-optimizations
sudo make && make altinstall
```

정상적으로 설치가 완료된 경우 `/usr/local/bin/` 아래 설치된 것을 확인할 수 있다.
![Image](/assets/posts/210903_superset_001.png)

### 2. venv 환경 구성
Superset을 설치하면서 생기는 라이브러리의 의존성에 문제가 없게 하기 위하여 [venv 환경](https://docs.python.org/ko/3/tutorial/venv.html)을 구성한다. 


```bash
cd /home/ec2-user
mkdir superset
cd superset
/usr/local/bin/python3.9 -m venv env
source /home/ec2-user/superset/env/bin/activate
```
해당 ENV를 활성화시키기 위하여 `source $venv/bin/activate` 로 활성화 시킨다.  활성화를 시킬 경우 `python` `pip`와 같은 커맨드가 모두 가상환경의 커맨드로 치환되기에 편하게 작업을 할 수 있다.
![Image](/assets/posts/210903_superset_002.png)

### 3. mysql8 설치
Superset에서 메타용도로 사용할 DB를 설치하여야한다. DB엔진으로 [sqlalchemy](docs.sqlalchemy.org)를 사용하고 있기에, 여러가지 DB를 선택할 수 있다. superset은 기본적으로 sqlite를 사용하도록 기본 컨피그가 설정되어 있으나, 복잡한 작업을 하기에는 좋지않기에 RDBMS를 이용하는게 보편적이다. 일반적으로 무료인 MySQL 또는 Postgre를 많이 사용하는 편이나, 익숙한 mysql을 설치하였다.

```bash
### 계정추가
sudo su
cd /home
groupadd mysql
adduser mysql -g mysql

### sudo 권한 추가. 아래파일을 열어 mysql 추가 // mysql ALL=(ALL) NOPASSWD:ALL
vi /etc/sudoers.d/90-cloud-init-users

## mysql 설치 ref ===>> https://techviewleo.com/how-to-install-mysql-8-on-amazon-linux-2
yum install https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
yum install mysql-community-devel
amazon-linux-extras install epel -y
yum install mysql-community-server

su - mysql
cd /home/mysql
mkdir data
mkdir log
mkdir temp

sudo cp /etc/my.cnf /etc/my.cnf.210903 # 백업
sudo cat > /etc/my.cnf # 설정은 아래 별로 Config 참조하여 내용 입력
sudo systemctl enable --now mysqld

# mysql 구동로그를 통해 초기 패스워드 확인
cat /home/mysql/log/mysqld.log |grep "temporary password"

# 위에서 확인한 패스워드를 이용하여 mysql 접속
mysql -uroot -p
```

```sql
-- 여기서부터 mysql 접속 후 초기 계정 세팅. 테스트이기에 host제한 없이 간편하게 설정
alter user 'root'@'localhost' IDENTIFIED BY '패스워드';
create user 'superset'@'%' identified by '패스워드';
create database superset;
grant all privileges on superset.* to 'superset'@'%';
flush privileges;
```

```bash
# my.cnf 샘플

[client]
default-character-set=utf8mb4
port=3306
socket=/home/mysql/log/mysql.sock
default-character-set=utf8mb4

[mysqld]
default_authentication_plugin=mysql_native_password

## remote setting
bind_address=*

## Password validation
validate_password.length=2
validate_password.mixed_case_count=0
validate_password.number_count=0
validate_password.policy=LOW
validate_password.special_char_count=0

## dir Setting
datadir=/home/mysql/data
socket=/home/mysql/log/mysql.sock
log-error=/home/mysql/log/mysqld.log
pid-file=/home/mysql/log/mysqld.pid

## chracter
character-set-client-handshake=FALSE
init_connect=SET collation_connection = utf8mb4_general_ci
init_connect=SET NAMES utf8mb4
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci
explicit_defaults_for_timestamp

## skip-options
skip-external-locking
skip-name-resolve

## connection
max_connections=1000
max_connect_errors=1000
wait_timeout=3600
```


## Superset 설치
### .bash_porfile 환경변수 추가
세션 접속 시 기본적으로 Superset에 대한 환경변수를 추가하도록 한다.
```bash
PATH=$PATH:$HOME/.local/bin:$HOME/bin:/home/ec2-user/superset
export SUPERSET_HOME=/home/ec2-user/superset
export SUPERSET_CONFIG_PATH=$SUPERSET_HOME/superset_config.py
export FLASK_APP=superset
```

`superset_config.py`는 [superset의 기본 config](https://github.com/apache/superset/blob/1.3.0/superset/config.py)를 오버라이딩 할 수 있으며, 각 항목에 대한 내용은 Airflow와는 달리 불친절하게 [Superset 공식문서](https://superset.apache.org/docs/installation/configuring-superset)에도 대략적으로만 있고 모든 내용이 안나와있기에(ㅜㅜ) 코드를 보면서 내용을 파악해야한다. 아마 이러한 불친절한 특성때문에 superset에 대한 것들이 좀 적은게 아닐까 싶다.

### Superset 설치
venv환경에서 설치를 진행하면된다
```
source /home/ec2-user/superset/env/bin/activate
pip install apache-superset
```

그러나 설치 도중 에러가 발생하였다. 로그 확인결과 의존성패키지 `python-geohash` 설치중 에러가 발생하였다.
```bash
    Running setup.py install for python-geohash ... error
    ERROR: Command errored out with exit status 1:
     command: /home/ec2-user/superset/env/bin/python3.9 -u -c 'import io, os, sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-l6nfkmi0/python-geohash_59f44f283a9843e49da2bd1a46dd18bd/setup.py'"'"'; __file__='"'"'/tmp/pip-install-l6nfkmi0/python-geohash_59f44f283a9843e49da2bd1a46dd18bd/setup.py'"'"';f = getattr(tokenize, '"'"'open'"'"', open)(__file__) if os.path.exists(__file__) else io.StringIO('"'"'from setuptools import setup; setup()'"'"');code = f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' install --record /tmp/pip-record-qwa4vq5f/install-record.txt --single-version-externally-managed --compile --install-headers /home/ec2-user/superset/env/include/site/python3.9/python-geohash
         cwd: /tmp/pip-install-l6nfkmi0/python-geohash_59f44f283a9843e49da2bd1a46dd18bd/
    Complete output (16 lines):
    running install
    running build
    running build_py
    creating build
    creating build/lib.linux-x86_64-3.9
    copying geohash.py -> build/lib.linux-x86_64-3.9
    copying quadtree.py -> build/lib.linux-x86_64-3.9
    copying jpgrid.py -> build/lib.linux-x86_64-3.9
    copying jpiarea.py -> build/lib.linux-x86_64-3.9
    running build_ext
    building '_geohash' extension
    creating build/temp.linux-x86_64-3.9
    creating build/temp.linux-x86_64-3.9/src
    gcc -pthread -Wno-unused-result -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -fPIC -DPYTHON_MODULE=1 -I/home/ec2-user/superset/env/include -I/usr/local/include/python3.9 -c src/geohash.cpp -o build/temp.linux-x86_64-3.9/src/geohash.o
    gcc: error trying to exec 'cc1plus': execvp: No such file or directory
    error: command '/usr/bin/gcc' failed with exit code 1
    ----------------------------------------
ERROR: Command errored out with exit status 1: /home/ec2-user/superset/env/bin/python3.9 -u -c 'import io, os, sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-l6nfkmi0/python-geohash_59f44f283a9843e49da2bd1a46dd18bd/setup.py'"'"'; __file__='"'"'/tmp/pip-install-l6nfkmi0/python-geohash_59f44f283a9843e49da2bd1a46dd18bd/setup.py'"'"';f = getattr(tokenize, '"'"'open'"'"', open)(__file__) if os.path.exists(__file__) else io.StringIO('"'"'from setuptools import setup; setup()'"'"');code = f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' install --record /tmp/pip-record-qwa4vq5f/install-record.txt --single-version-externally-managed --compile --install-headers /home/ec2-user/superset/env/include/site/python3.9/python-geohash Check the logs for full command output.
```

검색을 해보니 관련된 [이슈 및 해결책](https://github.com/apache/superset/issues/9558#issuecomment-658904680)을 superset github에서 발견할 수 있었다. 해당 라이브러리를 설치한 후 문제없이 superset에 대한 설치를 진행할 수 있었다.
```bash
sudo yum install gcc gcc-c++ libffi-devel python3-devel python3-pip python3-wheel openssl-devel cyrus-sasl-devel openldap-devel
pip install apache-superset
```

### Superset 초기화
Superset을 설치한 후 구동시키기 위해서는 메타 DB세팅을 해야한다. 그러나 별도의 설정을 하지 않으면 메타 DB는 SQLITE를 이용하기에 사전에 설치한 mysql을 이용할 수 있도록 컨피그를 설정한다.

```
cd /home/ec2-user/superset
cat > superset_config.py # 아래 내용 입력

SQLALCHEMY_DATABASE_URI = 'mysql+mysqlconnector://user:user@127.0.0.1:3306/superset'
```
먼저 언급한대로 Superset는 SQLALCHMY를 사용하기 때문에 사용하려는 메타DB에 따라 적절한 커넥션을 세팅하면 된다.  

superset이 설치되고 나면 CLI에서 superset 커맨드를 사용할 수 있게된다. DB 초기화에 앞서 `Pillow` 패키지가 의존성패키지에는 포함되지 않으나 실제로는 필요하기에 설치하도록 한다. 또한 위에서 mysql을 접속하기 위해 mysqlconnector를 사용하기로 했기때문에 `mysql-connector-python` 역시 설치한다.

```
pip3 install Pillow mysql-connector-python
superset db upgrade
superset init
superset fab create-admin # 슈퍼셋 웹에 접근 가능한 어드민 계정을 만든다.
```
![Image](/assets/posts/210903_superset_003.png)


### Superset 웹서버 실행 및 접근
superset에서는 gunicon과 같은 비동기 라이브러리로 웹서버를 구성하는것을 권장하고있다. 그러나 우선 기능을 테스트하기 위함이기 때문에 디버그 모드로 실행하도록 한다.
```bash
superset run -h 0.0.0.0 -p 8088 --with-threads --reload --debugger
```
![Image](/assets/posts/210903_superset_004.png)

기본적으로 Airflow와 같이 Flask베이스이기 때문에 실행 명령어 역시 비슷비슷하다. ec2에서 sg는 해당 8088포트에 접근 가능하게 이미 설정되어있다고 가정하면 아래와 같이 로그인페이지가 나온다.  
![Image](/assets/posts/210903_superset_005.png)

그러면 사전에 `superset fab create-admin`명령어로 생성한 계정으로 로그인하고, 로그인을 하면 본격적인 Superset UI를 볼 수 있다.  
![Image](/assets/posts/210903_superset_006.png)


## 다른 이야기
[Apache Superset(v1.3) 테스트 1편 - 메뉴설명](/python/superset-test-02/)
