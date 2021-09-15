---
title: Apache Superset(v1.3) 테스트 5편 - 데몬화/Daemonization
tags:
- python
- superset
- bi
- 슈퍼셋
toc: true
toc_sticky: true
category: python
---

## 개요
이전 글을 모두 읽었다면 알겠지만, Superset의 모든 기능을 사용하기 위해서는 3가지의 프로세스를 실행하여야한다.
* Superset websever
* Celery Worker
* Celery Beat

이러한 프로세스들을 데몬화 시킴으로, 실제 서비스하면서 언제나 실행될 수 있는 구조로 만드는것을 어떻게 해야하는가를 정리하기 위하여 이번 포스팅을 진행하였다.

## Webserser
Superset의 웹은 Flask를 기반으로 작성되어 있다. 따라서 Flask의 자체적인 웹서버 기능을 사용할 수 있으나, 프로덕션 환경에서는 아무래도 성능 등의 이슈가 있어 사용하기 애매한 부분이 있다.
Superset 공식 홈페이지에서도 관련하여 Flask의 웹서버 커맨드는 테스트 수준에서 쓰기를 권장하고 있으며, `gunicorn`을 이용하여 웹서버를 실행시키를 권장하고 있다.

슈퍼셋 설치와 함께 설치되는 `gunicorn`은 WSGI(Web Server Gateway Interface)인데, 파이썬에서 사용할 수 있는 WSGI중 자원사용량이 적으면서 응답속도가 빨라 많이 추천된다. 
Superset과 함께 Airbnb에서 시작한 Airflow의 웹서버 구동 명령은 이미 `gunicorn`을 이용하도록 커맨드가 제공되나, 슈퍼셋에서는 관련된 커맨드를 제공하지 않아 직접 명령어를 사용하여야한다.
`gunicron` 역시 bash에서 명령어로 사용할 수 있음으로 아래와 같이 간편하게 백그라운드에서 동작시킬 수 있다. 보다 다양한 파라메터등은 공식 문서를 참조하도록 하자.

```bash
gunicorn \
  -w 10 \
  -k gevent \
  --timeout 120 \
  -b 0.0.0.0:8088 \
  --limit-request-line 0 \ 
  --limit-request-field_size 0 
  superset.app:create_app() \
  -D
```
> 공식문서 예제에는 `-D` 옵션이 존재하지 않는다. 이는 데몬으로 `gunicorn`을 동작실킬건지에 대한 옵션이기에, 실제로는 반드시 사용해야하는 옵션이라고 볼 수 있다.  

## Celery
셀러리는 기본적으로 자체적인 데몬라이즈 관련된 명령을 지원한다. 
그러나 불행하게도 해당 기능으로는 Superset에 대한 관련 설정을 이용하지 않아 문제가 되었으며, 
이슈를 등록([https://github.com/apache/superset/issues/16562](https://github.com/apache/superset/issues/16562)) 하였으나 관련해서는 아직 오픈중인 상태이다.  

내용은 매우 심플하다. 관련된 옵션을 그대로 썼으며, 셀러리의 데몬화 관련 옵션을 적용했을 뿐이다. 그러나 포그라운드에서 실행할때와는 달리 앱을 지정했음에도 불구하고 `superset_config.py`에 입력한 셀러리 관련 옵션을 읽지 않았고, 그로인하여 스케쥴링 기능을 정상적으로 사용할 수 없었다. 그러나 그렇다고 언제나 세션을 열어 포그라운드로 실행하는 것 역시 문제가 되기에 아래와 같이 처리하였다.

* nohup을 이용하여 강제로 백그라운드에서 실행
  * 백그라운드에서 실행되기 때문에 추후 문제 발생 시 내용파악을 위한 로그파일을 명시
  * PID 파일을 생성하여 start/stop에서 사용할 ShellScript를 생성하기 쉽게 만들 것
  * loglevel을 지정해서 필요에 따라 디버깅 쉽게 할 것 
```bash
# 기본적인 nohup을 이용한 백그라운드 실행 예제
nohup celery worker --app=superset.tasks.celery_app:app --pool=prefork -O fair -c 4 --logfile=/home/ec2-user/superset/worker.log --pidfile=/home/ec2-user/superset/worker.pid --loglevel=WARN > /dev/null 2>&1 &
nohup celery beat --app=superset.tasks.celery_app:app --logfile=/home/ec2-user/superset/beat.log --pidfile=/home/ec2-user/superset/beat.pid --loglevel=WARN > /dev/null 2>&1 &
```


## 씨리즈
[Apache Superset(v1.3) 테스트 1편 - 설치](/python/superset-test-01/)  
[Apache Superset(v1.3) 테스트 2편 - 메뉴설명](/python/superset-test-02/)  
[Apache Superset(v1.3) 테스트 3편 - FEATURE_FLAGS](/python/superset-test-03/)  
[Apache Superset(v1.3) 테스트 4편 - Alert&Report](/python/superset-test-04/)    
[Apache Superset(v1.3) 테스트 5편 - 데몬화/Daemonization](/python/superset-test-05/) (지금이야기)  
[Apache Superset(v1.3) 테스트 6편 - 구글 OAuth 연동](/python/superset-test-06/)  
