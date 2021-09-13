---
title: Apache Superset(v1.3) 테스트 2편 - 메뉴설명
tags:
- python
- superset
- bi
- 슈퍼셋
toc: true
toc_sticky: true
category: python
---

#  개요
Superset 웹 UI에 대한 메뉴 설명에 대하여 정리한다. 기본적으로 Superset의 일부 기본적으로 사용할만한 기능들 중 상당수는 비활성화 되어 있기 때문에 활성화를 시켜야하는데, 우선 기본적인 메뉴 설명 및 각 메뉴별로 활성화 가능한 메뉴에 대하여 이번 포스팅에서 함께 정리한다.  

기본적인 테스트에 앞서, Superset의 예제를 로드하여 임의로 대시보드/차트에 대한 예제를 생성 할 수 있다. 해당 과정은 메타로 사용하는 DB안에 테이블을 생성하고, 관련된 데이터를 입력하는 작업을 수행한다. 그리고 완료되기까지 상당히 오랜 시간이 소요된다(테스트 상으로는 약 40분 실행)
```.bash
superset load-examples
```
![Image](/assets/posts/210906_superset_001.png)

# 메뉴바 구성
처음 Superset을 로그인 하게 되면 welcome페이지로 접근하게 된다. 이는 최근 사용한 내용, 대시보드, 차트 등에 대한 정보를 보여주는 Overview와 같은 페이지라고 볼 수 있다.  
Superset의 기본적인 메뉴는 UI 최상단에 존재하며, 이제 순서대로 UI에 대하여 알아보도록 한다.
![Image](/assets/posts/210906_superset_002.png)
## Dashboards
생성되어 있는 대시보드를 볼 수 있다.  
![Image](/assets/posts/210906_superset_003.png)

만들어진 대시보드에서 Title을 클릭함으로, 해당 대시보드에 접근이 가능하며, 권한이 있는 대시보드 우측의 연필모양의 아이콘을 클릭함으로 대시보드 수정이 가능하다. 또한 새로운 대시보드는 대시보드 목록에서 우측의 `+ DASHBOARD`를 클릭하여 만들 수 있다. 이러한 대시보드 Editer에서 각 항목에 대한 설명은 아래를 참조하도록한다.

![Image](/assets/posts/210906_superset_004.png)
1. 대시보드의 제목을 입력한다. 클릭 시 이름을 변경할 수 있다.
2. 클릭시 `Draft`와 `Published` 상태를 전환할 수 있다. `Draft`상태인 경우 대시보드의 소유자만 해당 대시보드를 볼 수 있다.
3. 즐겨찾기로 등록할 수 있다. 즐겨찾기로 등록된 경우 Dashboards 검색 필터에서 즐겨찾기 상태로 검색하는 필터가 존재한다.
4. `Undo` `Redo`로 대시보드 편집과정에서 변경내용을 전후로 원복 하고 싶을 때 이용한다.
5. 변경 사항을 저장하지 않고 편집기능을 나간다.
6. 변경 사항을 저장한다.
7. 추가적인 메뉴를 보여준다.
9. 대시보드를 꾸미기 위한 컴포넌트를 제공한다. 컴포넌트는 마우스 드래그를 이용하여 좌측 레이아웃에 배치하여 사용한다.
10. 대시보드에서 이용할 차트를 선택한다. 차트는 마우스 드래그를 이용하여 레이아웃에 배치하여 사용한다.
11. 대시보드 배치 레이아웃

## Chart
생성되어 있는 차트를 볼 수 있다.  
![Image](/assets/posts/210906_superset_005.png)

만들어진 차트의 경우 Title을 클림하여 차트로 들어가 편집이 가능하며 `+ CHART`를 클릭하여 새로운 차트를 생성할 수 있다. 하나의 차트는 하나의 `Data visualization`을 담당한다. 
차트를 만들기 위해서는 `DATASET`이 필요하며, 이는 물리적인 Table 또는 SQL을 이용하여 미리 생성하여야 한다. 차트는 [Apache ECharts](https://echarts.apache.org/en/index.html)를 기반으로 만들 수 있는 여러가지 차트가 있다. 이러한 사용가능한 내부적인 내용에 대해서는 여기에서 별도로 언급하지는 않으며, 예제 중 하나인 `Unicode Colud`를 기반으로 UI에 대한 대략적인 설명을 진행한다.

![Image](/assets/posts/210906_superset_006.png)
1. 해당 차트가 이용하고있는 `Dataset`을 나타낸다.
2. 해당 차트가 이용하고 있는 `Dataset`에서 사용할 수 있는 메트릭과 컬럼을 표시한다.
3. `RUN`은 차트를 만들기 위한 SQL을 실행하는 기능이며 `SAVE`는 차트를 저장하기 위한 옵션이다.
4. 어떤 비쥬얼 모양을 이용할 것인지 선택한다.
5. 데이터를 만들기 위한 시간 범위를 설정한다.
6. 어떤 컬럼을 사용할 것인가, 어떤 데이터를 집계할 것인가 등에 대한 내용을 설정한다.
7. 설정된 내용을 토대의 최종 결과물을 보여준다.
8. 7의 기반이 되는 원본 데이터를 표시한다.

이렇듯 Superset의 경우 `SLEECT ~ FROM ~ GROUP BY`와 같은 SQL을 전혀 모르더라도 UI만을 통하여 `Data visualization`을 가능하게 한다. 그러나, 이게 내가 원하는 데이터를 뽑은게 맞는지를 더욱 확실히 할 수 있게, 설정된 내용을 기반으로 데이터를 조회할 때 사용하는 SQL을 확인할 수 있는 기능을 제공한다.
![Image](/assets/posts/210906_superset_007.png)
![Image](/assets/posts/210906_superset_008.png)

## SQL Lab
![Image](/assets/posts/210906_superset_009.png)
SQL Lab은 SQL을 직접 다룰 수 있는 사용자에게 매우 유용한 메뉴로, Superset에 등록한 데이터베이스에 접근하여 쿼리를 실행하고 결과를 보여주는 IDE와 같은 기능을 할 수 있게 된다.
### SQL Editor
IDE와 같은 에디터 창을 지원하며 각 UI에 대한 설명은 아래를 참조한다.

![Image](/assets/posts/210906_superset_010.png)
1. Superset에 등록되어있는 데이터베이스와 스키마를 선택한다.
2. 선택된 데이터베이스의 테이블 리스트를 확인할 수 있으며, 선택할 수 있다.
3. 2에서 선택한 정보를 토대로 컬럼과 타입정보를 출력한다.
4. SQL을 입력할 수 있는 편집창
5. `RUN`을 클릭하거나 `Ctrl+Enter`를 입력 시 SQL 편집창에 입력된 SQL이 실행된다. `LIMIT`의 경우 데이터 건수가 많을 경우 제한을 두기 위한 옵션이고, 우측의 시간은 SQL의 실행시간 정보를 나타낸다. 
6. 쿼리를 저장할수 있다. 쿼리를 저장하는 경우 고유한 링크를 생성하여 링크를 복사할 수 있는 `COPY LINK`버튼이 활성화 된다. 마지막 `…`의 경우 추가적인 `Autocomplete`에 대한 스위치 버튼이 나타나는데, 기본값은 활성화이나 비활성화할경우 SQL Editer에서 자동완성 기능이 비활성화 된다.
7. 쿼리 결과 등을 보여주는 영역이다.
	8. `RESULTS` 말 그대로 SQL에 대한 결과를 보여준다. `RESULTS`에는 `EXPLORE` 버튼이 있는데 이를 누르게 되면 SQL을 `Dataset`으로 만들고, 이를 이용할 수 있다. 즉, Join과 같은 작업을 미리 수행한 후 Dataset으로 만들고 이를 차트로 만들때 사용하게된다.
	9. `DOWNLOAD TO CSV`의 경우 출력된 결과를 CSV로 다운로드 받을 수 있으며, `COPY TO CLIPBOARD`의 경우 클립보드로 결과를 복제한다.
  
### Save Queries
`SQL Editor`에서  저장한 SQL에 대한 리스트를 볼 수 있다.

### Query History
SQL Lab을 통해 실행된 쿼리에 대한 히스토리를 보여준다.

## Data
![Image](/assets/posts/210906_superset_011.png)
Data는 원천이 되는 데이터에 대한 세팅을 위한 메뉴이다.   

### Databases
![Image](/assets/posts/210906_superset_012.png)
Superset에서 사용하는 데이터베이스 관리한다. 등록하는 데이터베이스에 대하여 대략적인 Rule을 정의할 수있다.  
Database를 생성할 경우 서버에 설치된 Python 라이브러리에 따라 선택할 수 있는 Database 종류가 확장된다. 이는 [Install Database Drivers](https://superset.apache.org/docs/databases/installing-database-drivers)를 참조하여 원하는 Database를 설치할 수 있다.  

예를 들어 `BigQuery`를 설치하는 것을 예로 들면 아래와 같다. 관련된 라이브러리가 설치되지 않은 경우 Database를 등록하는 화면에서는 관련 내용을 확인할 수 없다.  
![Image](/assets/posts/210906_superset_013.png)

그러면 서버에 해당 문서에서 말한 라이브러리를 설치하도록 하자.
```bash
pip install pybigquery
```
![Image](/assets/posts/210906_superset_014.png)

그 다음 다시 Database 추가 메뉴를 확인 할 경우 정상적으로 선택할 수 있는 DataSource에 추가된 것을 알 수 있다. 단, 웹서버를 재시작하여야 새로 설치된 라이브러리가 반영된다. 중간에 `Athena` 관련 라이브러리도 설치해서 함께 보인다.  
![Image](/assets/posts/210906_superset_015.png)

### Datasets
![Image](/assets/posts/210906_superset_016.png)
`Dataset`을 관리하기 위한 메뉴이다. Superset은 Database를 등록한것만으로 차트를 생성할 수 없다. 따라서 Dataset에 관련내용을 등록해야한다.  
`Dataset`은 2가지 종류가 있으며, SQL 쿼리를 기반으로 하는 가상 Dataset에 대한 편집은 해당 메뉴에서 할 수 있지만 생성은 불가능하다. 가상 Dataset에 대한 생성은 [#sql-editor](#sql-editor)를 참고하도록 한다. 반면 물리적인 Table을 기반으로 하는 경우 `+DATASE`메뉴를 통해 추가할 수 있다. 

등록된 `Dataset`을 클릭할 해당 내용을 가지고 차트를 만들 수 있는 페이지로 리다이렉트 되며, `Action -> Edit`를 통해 `계산된 컬럼`과 같은 Dataset의 커스터마이징이 가능해진다. `계산된 컬럼`이란 예를들어 어떤 특정 컬럼에 1~10까지의 숫자가 있을 경우 1~4는 초기, 5~8은 중기, 9~10은 말기로 대체할 수 있을 때 SQL로는 CASE문을 이용할 수 있다. Superset에서는 이러한 SQL문을 이용 정의함으로 가상의 컬럼을 이용 할  수 있게 하는 기능이다. 
![Image](/assets/posts/210906_superset_017.png)

### Upload a CSV
말 그대로 CSV를 Database에 업로드할 수 있는 기능이다. 등록한 `Database`에 관련된 권한이 존재하여야만 사용할 수 있다. 다만 그 성격의 특성상 `Superset`을 통해서 하기도다는 별도의 엔지니어를 통하여 작업하는것이 더 올바른 사용방법일 것으로 보이기에 관련된 내용을 더 자세히 기술하지는 않는다.
![Image](/assets/posts/210906_superset_018.png)
## + 메뉴
![Image](/assets/posts/210906_superset_019.png)
해당 메뉴는 새로운 SQL, 차트, 대시보드를 만들기위한 핫링크를 지원하기 위한 단순한 UI이다.

## Settings
![Image](/assets/posts/210906_superset_020.png)
Superset에 대한 세팅을 위한 메뉴이다.

### Security
Superset의 웹은 Flask베이스로, Flask에서 사용가능한 회원에 대한 관리를 위한 전체적인 메뉴이다.

### Manage
Superset의 일반적인 관리를 관리를 위한 메뉴들이 존재한다.
1. Annotation Layer
	2. 차트에서 사용할 수 있는 주석을 관리하기 위한 메뉴
	3. 특정 기간범위를 라벨링 한 후 시계열 차트에서 적용할 경우 추가한 날짜범위가 차트에 표시된다.
		4. 특정한 이슈가 있는 경우에 미리 등록함으로 차트를 보는 사람이 그 범위를 인지할 수 있게 할때 사용한다.
	5.  Superset을 커스터마이징 하여 서비스하는 [Preset의 예제](https://docs.preset.io/docs/superset-annotations)를 보면 이해가 쉽다.(UI는 조금 다르게 생김)
6. CSS templates
	7. 대시보드에 적용할 CSS 템플릿을 관리할 수 있는 메뉴
	8. CSS에 대한 지식이 있다면 대시보드의 색상 등을 커스터마이징 할 수 있다.
8. Import Dashboard(s)
	8. Json형태로 아웃풋한 대시보드를 Import한다.
	9. Superset 마이그레이션에서 사용하기위한 메뉴

### User
Superset 웹에 로그인한 유저에 대한 정보확인, 로그아웃 등을 위한 메뉴이다.
### About
Superset 버전 정보를 표시한다.


# 씨리즈
[Apache Superset(v1.3) 테스트 1편 - 설치](/python/superset-test-01/)  
[Apache Superset(v1.3) 테스트 2편 - 메뉴설명](/python/superset-test-02/) (지금이야기)  
[Apache Superset(v1.3) 테스트 3편 - FEATURE_FLAGS](/python/superset-test-03/)  
[Apache Superset(v1.3) 테스트 4편 - Alert&Report](/python/superset-test-04/)  
[Apache Superset(v1.3) 테스트 5편 - 데몬화/Daemonization 및 기타](/python/superset-test-05/)  
