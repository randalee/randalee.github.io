---
title: Apache Superset(v1.3) 테스트 3편 - FEATURE_FLAGS
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
지금까지 Superset 설치 및 기본적인 UI에 대하여 알아보았다. 그러나 실제 Superset을 쓰기에는 아직 충분하지 않다.  

[Configuring Superset](https://superset.apache.org/docs/installation/configuring-superset) 을 보면 나와있듯이 별도의 Config를 설정하지 않는 한 [default config](https://github.com/apache/superset/blob/master/superset/config.py) 의 내용을 사용하도록 되어있다. 그중 `FEATURE_FLAGS`에 대한 내용이 존재하는데, 이는 말 그대로 특정 기능에 대하여 사용할 것인가 말것인가에 대한 활성화 여부를 결정한다. 그런데 Superset의 기본적인 Config에서 기본적으로 비활성화 되있지만 실제 환경에서는 당연히 동작해만 할 것 같은 기능들이 존재하였기에 수정이 필요하였다.  

모든 `FEATURE_FLAGS`에 대하여 기록하지는 못하지만 적어도 확인해야할만한 `FEATURE_FLAGS`에 대하여 정리하고자 한다. 


## 컨피그설정
`FEATURE_FLAGS`는 앞서 말한것과 같이 활성화 여부를 판단하기 위한 True/False를 값으로 Value로 가지고 있는 딕셔너리를 가지고있다. 
```python
DEFAULT_FEATURE_FLAGS: Dict[str, bool] = {
    # allow dashboard to use sub-domains to send chart request
    # you also need ENABLE_CORS and
    # SUPERSET_WEBSERVER_DOMAINS for list of domains
    "ALLOW_DASHBOARD_DOMAIN_SHARDING": True,
    # Experimental feature introducing a client (browser) cache
    "CLIENT_CACHE": False,
    "DISABLE_DATASET_SOURCE_EDIT": False,
    "DYNAMIC_PLUGINS": False,
    # For some security concerns, you may need to enforce CSRF protection on
    # all query request to explore_json endpoint. In Superset, we use
    # `flask-csrf <https://sjl.bitbucket.io/flask-csrf/>`_ add csrf protection
    # for all POST requests, but this protection doesn't apply to GET method.
    # When ENABLE_EXPLORE_JSON_CSRF_PROTECTION is set to true, your users cannot
    # make GET request to explore_json. explore_json accepts both GET and POST request.
    # See `PR 7935 <https://github.com/apache/superset/pull/7935>`_ for more details.
    "ENABLE_EXPLORE_JSON_CSRF_PROTECTION": False,
    "ENABLE_TEMPLATE_PROCESSING": False,
    "ENABLE_TEMPLATE_REMOVE_FILTERS": False,
    "KV_STORE": False,
    # When this feature is enabled, nested types in Presto will be
    # expanded into extra columns and/or arrays. This is experimental,
    # and doesn't work with all nested types.
    "PRESTO_EXPAND_DATA": False,
    # Exposes API endpoint to compute thumbnails
    "THUMBNAILS": False,
    "DASHBOARD_CACHE": False,
    "REMOVE_SLICE_LEVEL_LABEL_COLORS": False,
    "SHARE_QUERIES_VIA_KV_STORE": False,
    "TAGGING_SYSTEM": False,
    "SQLLAB_BACKEND_PERSISTENCE": False,
    "LISTVIEWS_DEFAULT_CARD_VIEW": False,
    # Enables the replacement React views for all the FAB views (list, edit, show) with
    # designs introduced in https://github.com/apache/superset/issues/8976
    # (SIP-34). This is a work in progress so not all features available in FAB have
    # been implemented.
    "ENABLE_REACT_CRUD_VIEWS": True,
    # When True, this flag allows display of HTML tags in Markdown components
    "DISPLAY_MARKDOWN_HTML": True,
    # When True, this escapes HTML (rather than rendering it) in Markdown components
    "ESCAPE_MARKDOWN_HTML": False,
    "DASHBOARD_NATIVE_FILTERS": False,
    "DASHBOARD_CROSS_FILTERS": False,
    "DASHBOARD_NATIVE_FILTERS_SET": False,
    "DASHBOARD_FILTERS_EXPERIMENTAL": False,
    "GLOBAL_ASYNC_QUERIES": False,
    "VERSIONED_EXPORT": False,
    # Note that: RowLevelSecurityFilter is only given by default to the Admin role
    # and the Admin Role does have the all_datasources security permission.
    # But, if users create a specific role with access to RowLevelSecurityFilter MVC
    # and a custom datasource access, the table dropdown will not be correctly filtered
    # by that custom datasource access. So we are assuming a default security config,
    # a custom security config could potentially give access to setting filters on
    # tables that users do not have access to.
    "ROW_LEVEL_SECURITY": True,
    # Enables Alerts and reports new implementation
    "ALERT_REPORTS": False,
    # Enable experimental feature to search for other dashboards
    "OMNIBAR": False,
    "DASHBOARD_RBAC": False,
    "ENABLE_EXPLORE_DRAG_AND_DROP": False,
    # Enabling ALERTS_ATTACH_REPORTS, the system sends email and slack message
    # with screenshot and link
    # Disables ALERTS_ATTACH_REPORTS, the system DOES NOT generate screenshot
    # for report with type 'alert' and sends email and slack message with only link;
    # for report with type 'report' still send with email and slack message with
    # screenshot and link
    "ALERTS_ATTACH_REPORTS": True,
    # FORCE_DATABASE_CONNECTIONS_SSL is depreciated.
    "FORCE_DATABASE_CONNECTIONS_SSL": False,
    # Enabling ENFORCE_DB_ENCRYPTION_UI forces all database connections to be
    # encrypted before being saved into superset metastore.
    "ENFORCE_DB_ENCRYPTION_UI": False,
    # Allow users to export full CSV of table viz type.
    # This could cause the server to run out of memory or compute.
    "ALLOW_FULL_CSV_EXPORT": False,
    "UX_BETA": False,
}
```
이렇게 기본값이 정해진 옵션을 수정하기 위해서는 `superset_config.py`에 `FEATURE_FLAGS = {}`와 같은 형태로 딕셔너리를 만들면, 동일한 KEY는 `FEATURE_FLAGS`의 VALUE를 따라 내용이 적용되게 된다. 총 35개의 KEY가 존재하나, 일부 내용에 대해서만 테스트를 진행하였다.   
그 이유는 이러한 각 옵션에 대한 상세한 내용은 공식홈페이지에도 내용이 정확하게 정리되있지 않기 때문에 옵션을 수정하고, Superset 동작확인 및 코드를 직접 살펴보면서 확인해야했었다 ㅜㅜ. 

### ENABLE_TEMPLATE_PROCESSING
[Jinja](https://jinja.palletsprojects.com/en/3.0.x/) 템플릿을 사용할 것인가에 대한 옵션으로 기본 값은 False이다. [SQL Templating](https://superset.apache.org/docs/installation/sql-templating) 에서 사용방법에 대한 대략적인 설명이 나와있다.  
일반적으로 SQL을 작성 시 값을 치환한다거나 할 경우에 유용하게 사용할 수 있다.

아래는 해당 옵션을 그대로 사용할 경우로 쿼리에 입력한 문법을 그대로 사용하였기에 SQL 문법에러가 발생하였다.  
![Image](/assets/posts/210908_superset_001.png)

해당 옵션을 True로 변경 후 다시 동일한 작업을 하였다.  
![Image](/assets/posts/210908_superset_002.png)

에러가 발생하긴 하였으나, 기존과는 다른 에러가 발생하였다. [Jinja](https://jinja.palletsprojects.com/en/3.0.x/) 템플릿을 사용할 경우 그 안에 들어간 내용이 Python에서 읽어 치환하도록 되어있다. `url_param`이라는 함수는 Superset에서 기본적으로 생성해둔 함수로, Superset URL 파라메터를 파싱하여 그 값을 가져오겠다는 의미이다.  
다만 현재 Bug가 있어 sqllab에서는 URL에 파라메터를 전달하더라도 결과가 무조건 None으로만 리턴되는 버그가 있다. 다만 `url_param`함수는 현재 sqllab에서는 동작하지 않는 이슈가 있으나, 이를 차트 및 대시보드에서는 정상적으로 동작한다. UI상에서도 Parameters라는 옵션이 추가되었다. 해당 내용은 Json을 입력할 수 있는 공간이고 치환을 어떻게 할 것인가 값을 입력할 수 있는 옵션이다.

현재는 [Jinja](https://jinja.palletsprojects.com/en/3.0.x/) 템플릿에 대한 sqllab쪽 버그가 있지만, `superset_config.py`에 별도의 함수를 만들어서 전달함으로, SQL에서 사용할 수 있어 SQL과 개발을 함께 사용할 경우 어느정도 유용하게 사용할 수 있을 것으로 보인다. `JINJA_CONTEXT_ADDONS` 라는 딕셔너리를 만들어, 임의로 생성한 함수를 전달할 경우 sql에서 사용할 수 있다.

다음과 같은 함수를 예로 보자
```python
import datetime
def tbetween(d=datetime.datetime.today().strftime('%Y-%m-%d'), daysadd=0):
    end=datetime.datetime.strptime(d, '%Y-%m-%d')
    start=end + datetime.timedelta(days=daysadd)

    return f"BETWEEN '{start.strftime('%Y-%m-%d')}' AND '{end.strftime('%Y-%m-%d')}'"

JINJA_CONTEXT_ADDONS = {
    'tbetween_func': tbetween
}
```
`tbetween`이라는 함수는 대략적으로 날짜 범위를 조회할때 자주 사용하는 문구를 생성해준다. 이를 `tbetween_func`라는 KEY로 입력하였다. sqllab에서 이를 실행해보도록 하자.  
![Image](/assets/posts/210908_superset_003.png)

Python에서 만든 함수에 대한 결과가 출력되었다. 이를 이용하여 다양한 방법으로 SQL을 템플릿화 하는 방식으로 사용할 수 있다.


### ENABLE_TEMPLATE_REMOVE_FILTERS

[Jinja](https://jinja.palletsprojects.com/en/3.0.x/) 템플릿을 이용하여 Dataset을 만들어 사용할 때 재귀함수를 쓸 경우 발생할 수 있는 이슈로 인하여 추가된 옵션이다.

자세한 사항은 [https://github.com/apache/superset/issues/13943](https://github.com/apache/superset/issues/13943)를 참조하도록 한다.

### ALERT_REPORTS
![Image](/assets/posts/210908_superset_004.png)
ALERT 및 REPORT를 활성화 시킬 것인가에 대한 옵션이다. 슈퍼셋은 이메일과 슬렉으로 경고 또는 리포트를 보낼 수 있는 기능이 존재하지만 기본적으로 비활성화 상태이다.  
이를 True로 변경할 경우 `Settings`에 관련된 메뉴가 추가되고 사용할 수 있다. 다만 UI에서 등록만 한다고 관련된 내용이 전달되지는 않는다.  
슈퍼셋은 `Celery`를 이용하여 등록된 내용을 실행시키기에 관련 프로세스를 추가적으로 실행시켜야만 동작한다. 다만, 여기에서 해당 내용에 대하여 적지는 않고 별도의 포스팅을 통하여 상세 내용을 기록하기로 한다.

### ESCAPE_MARKDOWN_HTML
마크다운에 HTML 삽입을 무시할 것인가에 대한 옵션으로 기본적으로는 `False`로 HTML을 사용할 수 있다. 마크다운은 Superset 대시보드 컴포넌트중의 하나로 대시보드를 꾸미는 수단이다.

대시보드 마크다운 컴포넌트를 드래그하여 배치 할 경우 기본적으로 보여주는 샘플이다.  
![Image](/assets/posts/210908_superset_005.png)

그러나 해당 옵션을 `True`로 변경 할 경우 테그가 그대로 출력 되버린다.  
![Image](/assets/posts/210908_superset_006.png)


### DASHBOARD_NATIVE_FILTERS
대시보드를 꾸미게 되면 실제 사용자들은 날짜, 카테고리 등 여러가지 항목을 필터링하여 대시보드를 검색할 수 있기를 원하게 된다. 초창기의 Superset은 이러한 필터 기능이 매우 조악하였는데(사실 지금도 완전 마음에 들지는..) 차트로 필터를 만든 후 대시보드에 필터차트와, 본문차트를 둘다 넣은 후 사용하는 방법을 썼었다. Superset의 load-examples로 가져온 예제는 모두 이런 방식으로 만들었는데, 이는 실제로 작업을 하면 대시보드를 꾸미는데 있어 상당히 불편함을 알 수 있다.   
네이티브 필터는 이러한 필터를 개선하기 위하여 만들어진 필터인데 기본적으로 비활성화이기에 꼭 활성화 하는것을 추천한다.

Superset 기본 예제 중 `World Bank's Data`를 보면 좌상단의 `Region Filter`가 클래식 필터이다.  
![Image](/assets/posts/210908_superset_007.png)

그러면 해당 기능의 Flag를 True로 변경하여 보자. 대시보드 좌측에 없던 아이콘이 생겼으며, 이를 눌러 네이티브 필터를 볼 수 있다.    
![Image](/assets/posts/210908_superset_008.png)  
![Image](/assets/posts/210908_superset_009.png)
  
필터를 추가하는 방법은 연필모양의 아이콘을 추가할 수 있으며, 문자열, 숫자범위, 시간범위 등에 대한 몇가지 필터 타입이 정의되어 있어 이를 이용할 수 있다.
![Image](/assets/posts/210908_superset_010.png)  
![Image](/assets/posts/210908_superset_011.png)  


### DASHBOARD_NATIVE_FILTERS_SET
네이티브 필터셋에 대한 사용여부로, 네이티브 필터로 등록한 내용에 대한 템플릿을 저장하여 사용할 수 있게 하는 기능이다. 예를 들어 `2021-09-01~2021-09-10`을 날짜 범위로 상태값이 `성공, 진행중`인 필터를 이용했다면 이를 저장한 후 추후에 저장된 값을 재사용할 수 있게 해주는 편의 기능이다.  
해당 옵션을 활성화 할 경우 네이티브필터 UI에 탭으로 구분되어 보여진다.  

![Image](/assets/posts/210908_superset_012.png)
필터셋에 저장된 내용이 아닌 필터를 검색했을 경우 새로운 필터셋으로 저장하거나, 기존 필터셋을 업데이트할 수 있으며, 현재 적용중인 필터가 아닌 경우 `클릭->Apply`를 통해 여러가지 옵션을 한번에 손쉽게 바꿀 수 있게 된다.

### DASHBOARD_CROSS_FILTERS
크로스 필터 사용여부로 기본값은 `False`이다. 크로스 필터랑 대시보드에 차트끼리 연동할 수 있는 필터를 의미한다.  
예를 들어, [A, B, C, D]의 값을 보여주는 차트 값이 있을 경우, 차트에서 A를 클릭할 경우 연관된 다른 차트에서도 A가 자동으로 필터링 되어 대시보드의 모습을 변화시켜주는 기능을 의미한다.

해당 기능을 활성화한 후 차트에서 `Most Dominant Platforms`을 보면 Query에 `EMIT DASHBOARD CROSS FILTERS`라는 체크박스가 생긴 것을 알 수 있다. 크로스 필터는 기본적인 옵션 활성화 뿐만 아니라 차트에서 해당 옵션을 체크하여야만 정상적으로 동작한다. 해당 기능을 체크하고 저장한 후 사용하고 있는 대시보드인 `Video Game Sales`로 이동하자.  
![Image](/assets/posts/210908_superset_013.png)

처음에는 전체에 대한 그래프가 보이며, `Publishers of Top 25 Games` 파이차트에서 특정한 범례를 클릭하게되면 해당 내용에 대해서 다른 대시보드가 필터링 됨을 확인할 수 있다.
![Image](/assets/posts/210908_superset_014.png)
![Image](/assets/posts/210908_superset_015.png)



### 결론(?)
최종적으로 아래와 같은 `FEATURE_FLAG`를 컨피그로 설정하였다.
```python
FEATURE_FLAGS = {
    "ENABLE_TEMPLATE_PROCESSING": True,
    "ENABLE_TEMPLATE_REMOVE_FILTERS": True,
    "ALERT_REPORTS": True,
    "ESCAPE_MARKDOWN_HTML": False,
    "DASHBOARD_NATIVE_FILTERS": True,
    "DASHBOARD_CROSS_FILTERS": True,
    "DASHBOARD_NATIVE_FILTERS_SET": True,
}
```

## 씨리즈
[Apache Superset(v1.3) 테스트 1편 - 설치](/python/superset-test-01/)  
[Apache Superset(v1.3) 테스트 2편 - 메뉴설명](/python/superset-test-02/)  
[Apache Superset(v1.3) 테스트 3편 - FEATURE_FLAGS](/python/superset-test-03/) (지금이야기)  
[Apache Superset(v1.3) 테스트 4편 - Alert&Report](/python/superset-test-04/)  