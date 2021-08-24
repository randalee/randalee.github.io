---
title: Redshift 성능개선을 위한 KEY 변경 시 주의점
tags:
- redshift
- database
toc: true
toc_sticky: true
category: redshift
---

### 개요
Redshift는 마스터노드와 컴퓨팅노드 여러개로 구성할 수 있다. 적절한 DIST_KEY 사용 시 데이터가 균일하게 여러 컴퓨팅 노드에 저장되기때문에 성능상에 잇점을 가지게 되는데, 분산키로 설정되지 않은 컬럼이 Join이 발생하게 되면 Redshift는 쿼리에 있어 컴퓨팅 노드를 함께 사용하기 위한 작업을 수행한다. 그 결과 성능이 하락하게 된다.

이 경우 적절하게 키 수정이 필요한 경우가 발생하는데 Size가 적은 테이블은 크게 문제가 없으나, 대용량의 테이블에 작업을 할 경우 발생한 이슈에 대하여 정리하기 위해 기록한다.

### ALTER
기본적으로 Table에 대한 ALTER 구문을 통해 DISK_KEY 및 SORT_KEY 수정이 가능하다.  [레드시프트 ALTER TABLE](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_ALTER_TABLE.html#r_ALTER_TABLE-synopsis)페이지에 자세한 ALTER TABLE 구문이 나와있으며, DIST_KEY/SORT_KEY의 경우 함께 변경이 가능하다.

```sql
ALTER TABLE tablename ALTER SORTKEY (column_list), ALTER DISTKEY column_Id;
ALTER TABLE tablename ALTER DISTKEY column_Id, ALTER SORTKEY (column_list);
ALTER TABLE tablename ALTER SORTKEY (column_list), ALTER DISTSTYLE ALL;
ALTER TABLE tablename ALTER DISTSTYLE ALL, ALTER SORTKEY (column_list);
```

분명히 이런 기능이 지원된다. 그러나 이러한 명령을 약 45억건이 존재하는 테이블에 수행하였더니, 약 5시간 이상의 매우 좋지않은 성능을 보여주었다. 또한 해당 키 변경이 발생하는 동안 테이블에 대한 `DML/SELECT`는 가능하나 [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)의 경우 사용할 수 없었다. [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)을 특정 조건에 의하여 자동으로 실행하게 설정한 경우 문제가 될 가능성이 있기에 주의가 필요하다.

또한 완료가 된 이후에도 문제가 발생하였다. 해당 테이블에 [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)을 수행하였더니, 약 3시간 이상 [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)이 실행된 이후에 완료되었다.  즉 해당 ALTER TALBE로는 [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)은 별도로 실행해야 한다는 것이다.  [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)의 낮은 성능은 둘째치고 하나의 명령만 사용할 수 있기에 이는 상당한 단점이 될 수 있다.


### 전체복사수행
[전체복사수행](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/performing-a-deep-copy.html)은 `ALTER KEY`의 낮은 성능을 회피 하고 보다 빠르게 작업하기 위한 방법이다.  새로운 테이블을 생성함으로 [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)을 이미 수행한 것과 같은 효과를 낼 수 있다. 공식 메뉴얼에서도 [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)보다 훨씬 빠른 성능을 보인다고 나와 있다.

사용방법은 간단하다. 원본테이블과 동일한 테이블을 생성하고, `INSERT INTO 신규테이블 SELECT * FROM 기존테이블`과 같은 형태로 데이터를 밀어넣는 것으로 작업은 완료된다. 단, DIST_KEY및 SORT_KEY는 미리 수정하도록 하자. 이 경우 테스트 결과 5시간걸리던 ALTER 작업이 INSERT로는 1시간이 조금 넘는 시간만에 완료되었다.  또한 [VACUUM](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/r_VACUUM_command.html)조차 진행할 필요가 없었다 :)

그러나 신규로 만들어진 테이블이기에 통계정보 갱신이 필요했었고, [Analyze](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/t_Analyzing_tables.html)구문을 사용했어야 한다. 해당 테이블에 [Analyze](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/t_Analyzing_tables.html)의 경우 약 20분 정도 소요되었다. 
[Analyze](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/t_Analyzing_tables.html)의 경우 COPY명령을 통한 경우 자동으로 분석한다고 나와있으나, `INSERT`를 통한 데이터 복제에는 적용되지 않았다.

작업이 완료되면, 복제본과 원본의 테이블을 스왑하여 성능을 파악하고, 최종적으로 문제가 없으면 기존의 테이블을 DROP하여 디스크 공간을 확보하여 작업을 완료한다.
