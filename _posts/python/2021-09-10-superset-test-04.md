---
title: Apache Superset(v1.3) 테스트 4편 - Alert&Report
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
[Apache Superset(v1.3) 테스트 3편 - FEATURE_FLAGS](/python/superset-test-03/)을 통하여 Alert 및 Report 메뉴를 활성화 시키는 방법을 알아보았다.
그러나 이러한 메뉴는 조건을 세팅할수만 있을 뿐 실질적인 Trigger역활을 하지는 못한다.  
Superset은 셀러리를 통하여 스케쥴링을 관리하고 있기에 관련된 세팅을 추가해야하며, 그 외 실제 구현에 있어 겪은 내용을 정리하도록 한다.

Superset에서 지원하는 Alert 및 Report 방식은 `Slack`과 `이메일` 두가지 방식을 지원한다. 해당 예제에서는 SMTP설정 말고, `Slack`을 이용한 방식을 토대로 설명하기로 한다. SMTP 옵션 설정 하는 부분만 제외하고는 근본적으로 동일하기에 메일로 하기에도 큰 어려움은 사실 없다.  


## Redis 설치
셀러리는 메시지를 보내고 받기 위한 솔루션이 필요하다. 일반적으로 이것은 `message broker`라고 하는 별도의 서비스 형태로 제공 되는데 공식적으로는 `RabbitMQ` 또는 `Redis`를 권장하고 있다.  
해당 테스트에서는 `Redis`를 이용하기 위하여 서버에 설치하기로 한다. 설치하는 방법은 다양하지만 간단하게 yum을 통해 설치하기로 하자. 

```bash
sudo yum install redis 
```
설치가 완료 된 경우 `/usr/bin/` 경로 아래 redis 관련된 실행파일들이 존재하게 된며, 별다른 옵션 없이 아래오 같이 Redis 서버를 구동할 수 있다.
```bash
sudo redis-server
```
![Image](/assets/posts/210910_superset_001.png)

그러나 이경우 SSH 세션을 종료하거나 `Ctrl+C`와 같이 나갈 경우 구동된 Redis 서버역시 종료되기에 `/etc/redis.conf`를 수정하기로 한다.
```bash
sudo vi /etc/redis.conf
```
파일을 열어보면 128라인에 `daemonize`가 `no`로 되있는데 이를 `yes`로 변경한 후 저장한다. 
![Image](/assets/posts/210910_superset_002.png)

이후 redis-server 명령을 실행하면서 설정한 Config파일을 명시함으로 해당 설정을 읽고, 자동으로 백그라운드에서 실행하도록 할 수 있다.
```bash
sudo redis-server /etc/redis.conf
```
![Image](/assets/posts/210910_superset_003.png)


## Slack BOT Token 발급
Slack API에서 메시지를 발송하기 위한 방식은 크게 2가지가 존재한다.
* Webhook
* Bot App

보통 많은 알람들은 Webhook 방식으로 메시지를 보내는 것이 일반적이다. 이는 고유한 URL을 이용하여 메시지를 전송할 수 있는 방식으로 매우 간편하다. 하지만 Superset에서는 알람과 동시에 특정한 대시보드 또는 차트를 상태를 이미지로 저장하여 전송할 수 있는 기능이 존재한다. 이를 위하여 파일 업로드가 가능해야하는데 Webhook방식에선 파일 업로드를 할 수 있는 방법이 없다. 따라서 Bot App을 만들고, 토큰을 발급받아야만 Slack연동이 가능해진다.

Slack bot token을 발급받는 방법은 많은 글들이 있기에 간략한 정보만을 기록하기로 하겠다.
1. [https://api.slack.com/apps/](https://api.slack.com/apps/) 접근
2. `Create New App`을 이용하여 앱 생성
    1. 생성과정에서 앱 이름과 워크스페이스 지정(여러개 사용하고있는 경우
3. `Features -> OAuth & Permissions`메뉴 클릭
4. `Scopes`에 적절하게 필요한 권한 추가. 최소한 아래 권한이 있어야한다.
    1. `incoming-webhook`
    2. `files:write`
    3. `chat:write`
5. 권한을 설정하고 난 후 앱 인스톨을 하여 Bot User OAuth Token 확인
   ![Image](/assets/posts/210910_superset_004.png)

추가적으로 Slack BOT의 경우 BOT APP이 메시지를 받기위한 채널에 존재하여야하기 때문에 반드시 해당 채널에 BOT을 초대하여야한다. 


## Chrome 웹 드라이브 설치
Superset에서 Alert 또는 Report를 전송 시 [Selenium](https://selenium-python.readthedocs.io/)을 이용한다.
셀레니움은 파이어폭스, 인터넷 익스플로어, 크롬등과 같은 브라우저를 컨트롤 할 수 있게 해준다. 이를 통해 Alert 및 Report에서 설정한 대시보드를 이미지 파일로 저장하여 전송하는데 이용한다.

셀레니움은 기본적으로 `webdriver`모듈을 이용하도록 되있는데, Superset의 기본값은 Firefox모듈을 사용하고 있으나, Amazon linux2에서 파이어폭스 웹드라이브 설치를 하려고 여러 문서를 찾아보고 테스트 하였으나 실패하여 크롬 웹 드라이버를 설치하게 되었다.

```bash
sudo yum install chromedriver
```
위와 같이 yum을 통해 설치가 가능하며, [https://chromedriver.chromium.org/downloads](https://chromedriver.chromium.org/downloads) 사이트에서도 직접 드라이버를 다운 받을 수 있다.
설치된 경우 아래와 같이 정상적으로 실행할 수 있어야 한다.
![Image](/assets/posts/210910_superset_005.png)

다만, 기본적으로 리눅스에는 영문만을 사용할 수 있기에, Superset에서 한글을 입력한 제목과 같은경우 글자가 깨지는 현상이 발생하게 된다.
따라서 Linux서버에 한글이 안깨지도록 관련된 라이브러리 설치가 필요하다.

```python
sudo yum install fonts-korean

fc-cache -r
sudo cd /usr/share/fonts/
sudo wget http://cdn.naver.com/naver/NanumFont/fontfiles/NanumFont_TTF_ALL.zip
sudo unzip NanumFont_TTF_ALL.zip -d NanumFont
sudo rm -f NanumFont_TTF_ALL.zip
fc-cache -r
```

## Superset 컨피그 수정
이제 앞서 준비한 내용을 사용할 수 있도록 `superset_config.py`를 수정하기로 한다.

아래는 셀러리에 대한 옵션이다.   
1. `BROKER_URL`과 `CELERY_RESULT_BACKEND`를 로컬에 설치한 redis로 설정하였다. 만약 로컬이 아니라 외부거나 포트가 다른 경우 적절하게 바꾸면 된다.  
2. `CELERYBEAT_SCHEDULE`은 셀러리 비트의 수행 주기를 세팅하는 부분이다.
3. 그외 부분은 공식문서에서 가이드한것과 비슷한 형태로 입력하였다.
4. 
```python
from celery.schedules import crontab
class CeleryConfig(object):
    BROKER_URL = 'redis://localhost:6379/0'
    CELERY_IMPORTS = (
        'superset.sql_lab',
        'superset.tasks',
        'superset.tasks.thumbnails'
    )
    CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
    CELERYD_LOG_LEVEL = 'DEBUG'
    CELERYD_PREFETCH_MULTIPLIER = 10
    CELERY_ACKS_LATE = True
    CELERY_ANNOTATIONS = {
        'sql_lab.get_sql_results': {
            'rate_limit': '100/s',
        },
        'email_reports.send': {
            'rate_limit': '1/s',
            'time_limit': 120,
            'soft_time_limit': 150,
            'ignore_result': True,
        },
    }
    CELERYBEAT_SCHEDULE = {
        'email_reports.schedule_hourly': {
            'task': 'email_reports.schedule_hourly',
            'schedule': crontab(minute=1, hour='*'),
        },
        'reports.scheduler': {
            'task': 'reports.scheduler',
            'schedule': crontab(minute='*', hour='*'),
        },
        'reports.prune_log': {
            'task': 'reports.prune_log',
            'schedule': crontab(minute=0, hour=0),
        },
        'alerts.schedule_check': {
            'task': 'alerts.schedule_check',
            'schedule': crontab(minute='*', hour='*'),
        },
    }

CELERY_CONFIG = CeleryConfig
```

Slack을 위한 토큰 값을 지정한다. 발급받은 SlackBot 토큰을 입력한다.  
이메일 발송을 위한 SMTP의 경우 설정해야할 변수역시 기록하였으니 참고하도록 한다. 
```python
SLACK_API_TOKEN = "xoxb-~~~~ 토큰"

# 이메일을 사용할 경우 아래 내용이용
SMTP_HOST = "smtp.sendgrid.net" # 적절하게 수정
SMTP_STARTTLS = True
SMTP_SSL = False
SMTP_USER = "your_user"
SMTP_PORT = 2525 # your port eg. 587
SMTP_PASSWORD = "your_password"
SMTP_MAIL_FROM = "noreply@youremail.com"
```

웹드라이브에 대한 설정을 진행한다.
```python
WEBDRIVER_TYPE = "chrome"

WEBDRIVER_WINDOW = {"dashboard": (1920, 1080), "slice": (1920, 1080)}
WEBDRIVER_OPTION_ARGS = [
    "--high-dpi-support=2.0",
    "--headless",
    "--disable-gpu",
    "--disable-dev-shm-usage",
    "--window-size=1920x1080",
]
WEBDRIVER_BASEURL_USER_FRIENDLY = "http://remote-domain:8088/"
WEBDRIVER_BASEURL = "http://0.0.0.0:8088/"
```
1. `WEBDRIVER_WINDOW`는 대시보드 또는 차트(slice)를 브라우져로 볼 경우 해상도를 의미한다. 기본 설정은 4K해상도에 맞춰져 있다보니 이미지가 너무 과하게 큰 문제가 있었다. 따라서 FHD인 1920*1080으로 설정하였다.
2. `WEBDRIVER_OPTION_ARGS`는 웹드라이브에서 사용가능한 적절한 옵션을 사용했다.
3. `WEBDRIVER_BASEURL_USER_FRIENDLY`는 외부에 메시지를 발송할때 링크에 넣을 주소이다. 만약 별도로 설정하지 않을경우 링크를 로컬주소를 넣기때문에 적절하지 않다. 따라서 외부에서 접속시 사용할 내용을 입력하면 된다.
4. `WEBDRIVER_BASEURL`는 Superset 로컬서버에서 셀레니움을 통해 웹서버 접근할 때 사용할 주소이다. `0.0.0.0`은 로컬을 의미한다.


## Celery 구동
셀러리에서 구동해야할 것은 크게 2개이다.
* beat
  * 스케쥴링을 담당.
* worker
  * 실제 작업을 담당.

Superset 1.3버전의 [setup.py](https://github.com/apache/superset/blob/1.3/setup.py#L70) 70라인을 보면 셀러리 버전 의존성을 확인할 수 있다.  
`"celery>=4.3.0, <5.0.0, !=4.4.1",`

즉, 4.4.1버전을 제외한 4.3 이상버전이면서 5버전 아래를 의미하며, 해당 버전에 해당하는 latest버전인 `4.4.7` 버전이 superset설치와 함께 설치된다.
이렇게 설치가 된 경우 bash상에서 `celery` 커맨드를 이용할 수 있게 된다. 이를 이용하여 Superset App을 명시하여 기능을 동작시킨다.
```bash
celery worker --app=superset.tasks.celery_app:app --pool=prefork -O fair -c 4
celery beat --app=superset.tasks.celery_app:app
```

다만 기본적으로 데몬으로 동작하지 않기 때문에 위와 같이 실행 시킬경우 2개의 세션이 필요하다. 
정상적으로 실행된 경우 다음과 같이 실행되며, 해당 화면에서 `Ctrl+C`로 프로세스를 빠져나올 수 있다.

![Image](/assets/posts/210910_superset_006.png)
![Image](/assets/posts/210910_superset_007.png)

## 테스트 진행
셀러리 프로세스까지 정상적으로 실행되었다면 실제로 메시지가 전송되는지 테스트하도록 한다.    
Superset 웹페이지에서 `Setting -> Alerts & reports` 메뉴를 클릭하여 해당 메뉴로 이동한다. 기본적으로는 `Alerts` 페이지를 표시하며, `Reports`를 클릭하여 전환할 수
이 아무것도 없기에 리스트에는 아무것도 표시되지 않으며 우측의 `+ALERT`버튼(Report의 경우 `+Report`)을 클릭하여 새로운 알람 또는 리포트를 생성할 수 있다.

> Alert과 Report의 차이는 메시지를 보내기 위한 Trigger조건이 추가적으로 있는가에 대한 여부에 따라 존재한다. 두가지 모두 특정 주기로 실행하는 형태이기는 하나, Alert는 설정한 "임계치"에 해당하는 경우 동작하도록 추가적인 조건이 존재한다. 반면 Report는 실행하는 주기마다 항상 동작하게 된다.

![Image](/assets/posts/210910_superset_008.png)

아래는 Alert에 대한 내용을 등록 또는 수정할 때 나오는 화면으로 각 항목에 대한 설명은 아래를 참조한다.
![Image](/assets/posts/210910_superset_009.png)

|카테고리|항목|내용|
|:---|:---|:---|
|최상단|ALERT NAME|등록한 ALERT의 이름으로, 알림을 보낼 때 제목으로 이용한다.|
|최상단|OWNERS|ALTER에 대한 소유자 설정|
|최상단|DESCRIPTION|ALERT에 대한 설명으로, 알림을 보낼 때 함께 관련 문구가 전송된다.|
|Alert condition|DATABASE|등록된 Database source를 선택한다.|
|Alert condition|SQL QUERY|선택한 database에서 사용할 수 있는 SQL문을 설정한다. 이 때 결과는 스칼라 서브쿼리처럼 단일 결과여야 한다.|
|Alert condition|TRIGGER ALERT IF...|`SQL QUERY`에서 나온 결과를 어떻게 비교할 것인가에 대한 연산자를 선택한다. 일반적인 부등호와 Not null조건을 가진다. |
|Alert condition|VALUE|비교대상이 되는 값이다. 만약 해당 값이 1이고 `TRIGGER ALERT IF...`가 `==`로 설정한 경우 `SQL QUERY`의 결과가 1인 경우 Alert이 동작하게 된다.|
|Alert condition schedule|Every~/CRON Schedule|해당  Alert이 실행되는 주기를 설정한다. Crontab형태 또는 패턴 둘중 하나를 사용하게 된다. 최소 Interval은 매분(Every minute)이다.|
|Alert condition schedule|TIMEZONE|실행주기에 대한 타임존이다. 예를들어 09시에 실행된다고 설정하였을때 UTC를 기준으로 하는지 KST를 기준으로 하는지와 같은 설정을 담당한다.|
|Alert condition schedule|TIMEZONE|실행주기에 대한 타임존이다. 예를들어 09시에 실행된다고 설정하였을때 UTC를 기준으로 하는지 KST를 기준으로 하는지와 같은 설정을 담당한다.|
|Schedule settings|LOG RETENTION|로그를 보관할 주기를 설정한다.|
|Schedule settings|WORKING TIMEOUT|ALTER 발생 시 작업이 완료될때까지 걸리는 타임아웃 시간을 설정한다. 만약 해당 시간을 초과하면 Alert 발송은 실패한다.|
|Schedule settings|GRACE PERIOD|Alert은 동일한 유형을 자주 받을 필요성이 없다는것을 간주한 옵션이다. 만약 Alert조건에 속하더라도 마지막 Alert과 현재시점이 해당 시간 이내라면 Alert을 발송하지 않는다.|
|Message content|Dashboard|Alert과 함께 보낼 대시보드를 선택한다. 셀레니윰 모듈을 통해 대시보드를 이미지로 캡쳐한 후 전송한다.|
|Message content|Chart|Alert과 함께 보낼 차트를 선택한다. 셀레니윰 모듈을 통해 대시보드를 이미지로 캡쳐한 후 전송하거나, CSV로 원본 데이터를 보내는 2가지 기능을 가진다.|
|Notification method|Add delivery method|해당 Alert을 전송할 주체를 선택한다. 이메일과 슬렉을 지원한다.|

아래는 Report 대한 내용을 등록 또는 수정할 때 나오는 화면이다. 특정 시간마다 발송하는 Report의 특성상 Alert과의 차이점은 `Alert condition`을 설정하는 부분이 없다는것을 제외하고는 기본적으로 동일하다. 
![Image](/assets/posts/210910_superset_010.png)

우선 테스트를 위하여 `Alert`을 하나 추가하기로 한다. 아래와 같이 설정하였다. SQL은 언제나 1을 리턴하도록 하였으며, 그 값이 1일 경우 Alert을 발송하도록 되있다. 즉, 언제나 Alert이 발송되는 샘플이다. 
![Image](/assets/posts/210910_superset_011.png)

스케쥴링이 돌아오는 시간이 되면 셀러리 워커 로그에서 관련한 로그를 확인할 수 있다.
![Image](/assets/posts/210910_superset_012.png)

그리고 실제 Slack에도 관련된 내용이 전달됨을 확인할 수 있다.
![Image](/assets/posts/210910_superset_013.png)


## Alert 유예기간 관련 코드 이슈
Alert의 `GRACE PERIOD`기능은 불필요할 정도로 과한 알람을 방지하기 위한 기능이다.  
그러나 의도하지 않은 동작을 야기시키는 부분이 있어(이부분은 기존이 더 좋다고 느낄 수도 있겠지만) 관련된 내용에 대하여 기록 및 수정방법을 기록한다.

우선 발견한 이유는 해당 시간을 1초로 하여 1분마다 발생하는 `Alert`이 항상 발송될 것이라는 것을 가정하고 테스트를 진행하였는데, 실제로는 2분마다 Slack발송이 되었다.
그래서 먼저 Superset issue에서 관련된 내용이 있는가 찾아보았고 `2021년 3월 3일` [오픈된 이슈](https://github.com/apache/superset/issues/13430) 가 존재하지만 여전히 해결되지 않은 상태를 확인했다.

따라서 실제로 Alert을 발송시킬때 사용하는 코드분석을 시작하였으며, 그 결과 `Alert 발생 -> GRACE PERIOD`기간이 넘어가는 첫번재 케이스는 Alert을 발송하지 않고 관련 Exception을 발생시킴을 확인하였다. 
따라서 관련된 코드를 아래와 같이 수정하여 첫번째로 넘어가는 케이스에도 무조건 Alert을 발생시키도록 코드를 수정하였고 정상적으로 매 분마다 발송되는 것을 확인하였다.

수정한 next function [https://github.com/apache/superset/blob/master/superset/reports/commands/execute.py#L535](https://github.com/apache/superset/blob/master/superset/reports/commands/execute.py#L535)
```python
    def next(self) -> None:
        self.set_state_and_log(ReportState.WORKING)
        if self._report_schedule.type == ReportScheduleType.ALERT:
            if self.is_in_grace_period():
                self.set_state_and_log(
                    ReportState.GRACE,
                    error_message=str(ReportScheduleAlertGracePeriodError()),
                )
                return
            # If it's an alert check if the alert is triggered
            if not AlertCommand(self._report_schedule).run():
                self.set_state_and_log(ReportState.NOOP)
                return
        try:
            self.send()
            self.set_state_and_log(ReportState.SUCCESS)
        except CommandException as ex:
            self.set_state_and_log(ReportState.ERROR, error_message=str(ex))
```

## 씨리즈
[Apache Superset(v1.3) 테스트 1편 - 설치](/python/superset-test-01/)  
[Apache Superset(v1.3) 테스트 2편 - 메뉴설명](/python/superset-test-02/)  
[Apache Superset(v1.3) 테스트 3편 - FEATURE_FLAGS](/python/superset-test-03/)  
[Apache Superset(v1.3) 테스트 4편 - Alert&Report](/python/superset-test-04/) (지금이야기)  
