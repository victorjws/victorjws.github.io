---
title: "MySQL default SQL Modes"
date: 2021-08-22 18:06:00 +0900
categories: MySQL
---
회사에서는 `NO_ENGINE_SUBSTITUTION`만을 사용하고 있었는데,
SQL MODE에 대해 찾아보다가 간혹 datetime 형식에
'0000-00-00 00:00:00'과 같이 잘못된 데이터가 들어가는 문제를 해결할 수 있는
방법이 될 수 있을 것 같아서 회사에 도입을 제안하게 되었다.

그 외에도 5.7 버전의 기본 모드들도 함께 활성화 도입을 추진했는데,
deprecated 되어 이후 MySQL version 에서는 항시 활성화되는 mode 도 있어
추후 MySQL version upgrade 시에 영항이 갈 수 있는 부분을 미리 최소화하기 위해서였다.

`ONLY_FULL_GROUP_BY`는 예외로 활성화하지 못했는데,
기존 코드에서 사용하는 query 중 GROUP BY를 사용하면서 `ONLY_FULL_GROUP_BY`를
위반하는 경우가 많아 문제가 발생했기 때문이었다.


## MySQL default SQL Modes
MySQL 5.7 version 기준 default sql modes 는 다음과 같다.
`ONLY_FULL_GROUP_BY`, `STRICT_TRANS_TABLES`, `NO_ZERO_IN_DATE`, `NO_ZERO_DATE`,
`ERROR_FOR_DIVISION_BY_ZERO`, `NO_AUTO_CREATE_USER`, `NO_ENGINE_SUBSTITUTION`

각 mode에 대해 설명하자면,

- `ONLY_FULL_GROUP_BY` :  GROUP BY 사용 시, 이름이 지정되지 않았거나, 집계함수(예. SUM(), COUNT() 등) 등으로 결정되는 값이 아닌 column이 select, having, order by 절에 존재할 경우 에러를 발생시키는 mode. 이는 group by 로 지정되지 않은 column을 select, having, order by 절에서 사용할 때 그룹화 되는 값 중 랜덤으로 선택되어 쿼리의 결과가 매번 달라질 수 있는 가능성을 차단하기 위한 mode이며, 이에 관한 예는 [여기](https://www.psce.com/en/blog/2012/05/15/mysql-mistakes-do-you-use-group-by-correctly/) 를 참조하면 이해하기 쉽다.
- `STRICT_TRANS_TABLES` : strict SQL mode를 활성화하는 mode. insert, update 시 유효하지 않은 값이 존재할 때 에러를 발생시킨다. 후술할 NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO 또한 STRICT_TRANS_TABLES  mode가 활성화 되어 있어야 에러를 발생시키고, 그렇지 않으면 경고(warning)만 생성한다. IGNORE를 추가할 경우 에러가 발생해도 무시할 수 있다.
- `NO_ZERO_IN_DATE` : "2010-00-01" 처럼 날짜 데이터 안에 0이 포함되면 에러를 발생시키는 mode. "0000-00-00"과 같은 값은 이 mode의 활성화로는 오류를 발생시키지 않는다. deprecated 상태.
- `NO_ZERO_DATE` : "0000-00-00" 과 같은 날짜데이터를 입력하면 에러를 발생시키는 mode. deprecated 상태.
- `ERROR_FOR_DIVISION_BY_ZERO` : 0으로 나누거나 MOD(N,0)를 실행하면 에러를 발생시킨다. 해당 mode 를 활성화하지 않으면 NULL 값으로 return 한다. deprecated 상태.
- `NO_AUTO_CREATE_USER` : 인증 정보(IDENTIFIED BY, IDENTIFIED WITH)가 제공되지 않을 경우 GRANT 문으로 계정을 자동으로 생성하지 못하도록 방지한다. 5.7에서는 deprecated 상태였으며, 8.0에서는 없어졌다.
- `NO_ENGINE_SUBSTITUTION` : table 생성, 변경 시 원하는 storage engine(예. InnoDB)을 사용할 수 없는 경우 에러를 발생시킵니다. 이 mode를 사용하지 않을 경우, 원하는 storage engine을 사용할 수 없을 때 자동으로 기본엔진을 사용합니다.

이 중 `NO_ZERO_IN_DATE`,`NO_ZERO_DATE`,`ERROR_FOR_DIVISION_BY_ZERO`,`NO_AUTO_CREATE_USER` 는 deprecated 된 것으로, 향후 release에서 `NO_ZERO_IN_DATE`,`NO_ZERO_DATE`,`ERROR_FOR_DIVISION_BY_ZERO` 는 strict mode 활성화 시 기본으로 포함될 예정이다. `NO_AUTO_CREATE_USER` 는 항상 활성화된 상태가 유지될 것이라고 표기되어 있으며 8.0에서는 삭제되었다.

## References
- [MySQL SQL Modes](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html)
- [mysql mistakes do you use group by correctly](https://www.psce.com/en/blog/2012/05/15/mysql-mistakes-do-you-use-group-by-correctly/)