---
title: Apache Superset(v1.3) 테스트 6편 - 구글 OAuth 연동
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
Superset은 python의 웹프레임워크 라이브러리 중 [Flask App Builder](https://flask-appbuilder.readthedocs.io/en/latest/intro.html) 를 기반으로 만들어졌다. 
[Flask App Builder](https://flask-appbuilder.readthedocs.io/en/latest/intro.html) 는 Django-admin에 영감을 받아 만들어졌으며 다양한 기능을 제공하며, 해당 라이브러리에 이해가 높다면 직접적인 Superset의 커스터마이징을 하는데 도움을 줄 수 있다.

반면, 이러한 프레임워크에서 자주 사용될만한 기능들은 이미 라이브러리로 만들어진 경우도 있는데 [authlib](https://authlib.org/) 이 대표적이다. Flask의 경우 구글 연동에 대한 기본적인 설정만 사용하여 authlib를 사용할 수 있다.
> Superset에서 사용하기 위하여 포스팅하고 있으나, 사실 Flask를 사용하는 곳 모두 적용가능하다


## 사전 제한
Google의 Oauth 기능은 사용자를 보호하기 위해 Google에서는 OAuth를 사용하는 앱만 승인된 도메인을 이용할 수 있도록 허용하도록 강제하고 있다.  
이러한 승인은 반드시 도메인을 승인하여야 하기 때문에 도메인이 없는 경우 해당 기능을 사용할 수 없다. 도메인 등록방법은 다양하기 때문에 여기서 별도로 가이드하지 않으나, 테스트에서는 route53을 통하여 도메인을 사용할 수 있게 하였다.

## Authlib 설치
설치 자체는 특별한 부분 없이 pip를 통해 설치하면 된다. 다만, 설치할때는 `authlib`와 같이 대소문자 구분없이 해도 문제가 되지는 않으나 사용시에는 `Authlib`로 첫 글자가 대문자이다.
```bash
pip install authlib
```
![Image](/assets/posts/202109/210915_superset_001.png)


## 구글 프로젝트 설정
구글 인증을 하기 위해서는 구글에 먼저 프로젝트를 등록하여야한다. 따라서 우선 아래와 같이 프로젝트를 등록한다.

1. [Google Cloud Platform](https://console.cloud.google.com/projectselector2) 페이지 이동
2. 우측의 `프로젝트 만들기`를 클릭
![Image-full](/assets/posts/202109/210915_superset_002.png)

3. 적절하게 프로젝트 이름 부여. 만약 사용하는 구글 계정이 개인 계정이 아닌 회사의 `G-Suite`라면 상위 조직 또는 폴더를 선택가능하다.  
![Image](/assets/posts/202109/210915_superset_003.png)

4. 프로젝트가 생성되면 죄측 상단의 `탐색메뉴`를통해 `API 및 서비스 -> 사용자 인증 정보`로 진입한다.  
![Image](/assets/posts/202109/210915_superset_004.png)

5. `사용자 인증정보 만들기 -> OAuth 클라이언트 ID 만들기`를 클릭한다. 

6. 웹 애플리케이션을 선택하고 적절하게 내용을 조정한다. 이때 URI에는 슈퍼셋 주소를 입력해야한다. 단, 이때 IP주소와 같은 경우 등록되지 않기에 도메인이 반드시 필요하다.  승인된 자바스크립트 원본의 URI에는 슈퍼셋 접속 URL을 입력하고, 승인된 리디렉션 URI에는 원본 뒤에 `/oauth-authorized/google`를 추가적으로 붙인다.
![Image](/assets/posts/202109/210915_superset_006.png)  

7. 생성이 완료되면 관련된 클라이언트 아이디와 비밀번호가 제공된다.
![Image](/assets/posts/202109/210915_superset_005.png)  

8. 이제 준비가 구글 OAuth연동을 위한 준비는 끝났다!


## Superset config 수정 및 연동 테스트
이제 앞서 준비한 내용을 적용하기 위하여 `superset_config.py`를 수정하고자 한다.
아래 템플릿을 이용하여 적절하게 수정하도록 하자.

```python
from flask_appbuilder.security.manager import AUTH_OAUTH
AUTH_TYPE = AUTH_OAUTH
AUTH_ROLE_PUBLIC = 'Public'
AUTH_USER_REGISTRATION = True
OAUTH_PROVIDERS = [{
        "name": "google",
        "token_key": "access_token",
        "icon": "fa-google", "remote_app": {
                "api_base_url": "https://www.googleapis.com/oauth2/v2/",
                "client_kwargs": {
                        "scope": "email profile"
                },
                "access_token_url": "https://accounts.google.com/o/oauth2/token",
                "authorize_url": "https://accounts.google.com/o/oauth2/auth",
                "request_token_url": None,
                "client_id": "여기에 발급된 클라이언트 아이디 입력",
                "client_secret": "여기에 발급된 클라이언트 비밀번호 입력"
        }
}]
WEBDRIVER_BASEURL_USER_FRIENDLY = "여기에 슈퍼셋 접속 URL 입력"
```

이후 웹에 접속하게 되면 웹서버 로그에서 연결한 구글쪽으로 리다이렉트 하는 로그를 확인할 수 있으며, 웹페이지에서는 구글 로그인 화면이 나오게된다.
![Image](/assets/posts/202109/210915_superset_007.png)
![Image](/assets/posts/202109/210915_superset_008.png)


## 관리자 권한 부여
OAuth 연동을 완료한 후 로그인 시 `AUTH_USER_REGISTRATION = True` 옵션을 넣었기에 만약 등록되지 않은 경우 자연스럽게 가입되며 로그인이된다 
그러나 `AUTH_ROLE_PUBLIC = 'Public'` 이기 때문에 권한이 Public로 로그인되게 되며 메뉴를 전부 볼 수 없다. 따라서 메타DB로 사용하는 테이블을 수정하여 권한을 부여하도록 한다.

FlaskAppBuilder를 이용한 회원정보에 관한 테이블은 다음과 같다.
* ab_user - 회원정보를 담고있는 테이블
* ab_user_role - 회원과 롤을 매핑하는 테이블
* ab_role - 롤에 대한 테이블. 기본적으로 `superset init` 커맨드까지 완료하면 6개의 role를 보유하게된다.

이를 종합하여 아래와 같이, 유저별 Role를 추출 가능하다.  
```sql
SELECT 
	au.id,
	au.first_name,
	username,
	ar.name
FROM ab_user au 
	LEFT OUTER JOIN ab_user_role aur ON au.id = aur.user_id
	LEFT OUTER JOIN ab_role ar ON aur.role_id = ar.id
```
![Image](/assets/posts/202109/210915_superset_009.png)

즉, 저 권한을 Admin으로 수정하면 되는 것이기에, ab_user_role에서 적절하게 수정하면 된다.
```sql
-- 대략적으로 이런느낌??
UPDATE ab_user_role 
SET role_id = (SELECT id FROM ab_role WHERE name = 'Admin')
WHERE user_id = 1;
```
수정 후 정상적으로 Role이 변경된 것을 확인할 수 있다.  
![Image](/assets/posts/202109/210915_superset_010.png)

이제 다시 Superset 로그인을 진행해본다. 모든 메뉴가 활성화 되며, Admin 롤을 보유하게 된다. 
따라서 추가로 가입하는 유저의 경우 슈퍼셋의 회원관리 기능을 통해 적절하게 할 수 있다.
![Image](/assets/posts/202109/210915_superset_011.png)


## 씨리즈
[Apache Superset(v1.3) 테스트 1편 - 설치](/python/superset-test-01/)  
[Apache Superset(v1.3) 테스트 2편 - 메뉴설명](/python/superset-test-02/)  
[Apache Superset(v1.3) 테스트 3편 - FEATURE_FLAGS](/python/superset-test-03/)  
[Apache Superset(v1.3) 테스트 4편 - Alert&Report](/python/superset-test-04/)    
[Apache Superset(v1.3) 테스트 5편 - 데몬화/Daemonization](/python/superset-test-05/)  
[Apache Superset(v1.3) 테스트 6편 - 구글 OAuth 연동](/python/superset-test-06/) (지금이야기)
