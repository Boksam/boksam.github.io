---
title: "dd-trace-py에 pg-hint-plan 이슈 해결하기"
date: 2026-08-30
categories: [Open Source, Python]
tags:
  - Datadog
  - PostgreSQL
  - dd-trace-py
  - pg-hint-plan
image: "/assets/datadog-logo.png"
---

> 이 글에서는 Datadog의 오픈소스인 [dd-trace-py](https://github.com/DataDog/dd-trace-py)에 pg-hint-plan 충돌 관련 이슈를 분석하는 과정에 대해 다룹니다.
>
> Issue: [DBM SQL comments hide leading pg_hint_plan hints
](https://github.com/DataDog/dd-trace-py/issues/19775)
{: .prompt-info}

## 1. Overview

Datadog에서 Technical Support Engineer로 일한지 어느덧 6개월이 되었습니다.
Datadog을 보면서 느낀 점은 제품들 간의 연동이 정말 뛰어나다는 점이였습니다.

예를 들어, Datadog에는 RUM(Real User Monitoring)이라는 제품이 있습니다.
RUM에서는 브라우저 혹은 모바일 환경에서 사용자가 어떤 화면을 클릭했고, 얼마나 머물렀는지 등을 분석해서 실제 사용자 경험을 모니터링할 수 있습니다.
APM(Application Performance Monitoring)에서는 하나의 요청을 처리하기 위해 여러 분산 서비스를 거칠 때, 요청이 어떤 경로로 처리되었고 각 명령어 단위를 일일이 분석할 수 있습니다.

여기서 사용자는 RUM과 APM을 연동할 수 있습니다.
이렇게 하면, 관리자는 사용자가 어떤 버튼을 클릭한 순간부터 생성된 요청이 어떻게 처리되었는지 모두 확인할 수 있게 됩니다.
다시 말하면, 실제 사용자 환경에서 Backend까지 어떤 일이 발생하는지 모두 관측할 수 있게 되는 것입니다.

그리고 여기서 한발짝 더 나아가서 APM은 DBM(Database Monitoring)과도 연동할 수 있습니다.
이렇게 하면, 아래와 같은 연동 구조가 만들어집니다. 이렇게 연동된 데이터를 통해 관리자는 UI에서 사용자의 행동부터 백엔드 서비스의 요청 처리 경로, 그리고 데이터베이스 쿼리 성능 지표를 하나의 연결된 구조로 모니터링 할 수 있습니다.

![](/assets/2026-08-30-ddtrace-contrib-pg-hint-plan/rum-apm-dbm-correlation.png)

이처럼 Datadog의 핵심 장점 중 하나는 제품간의 연동(**Correlation**) 이라고 할 수 있습니다.
그러던 중, dd-trace-py 레포지토리에서 다음과 같은 이슈를 발견했습니다.

[DBM SQL comments hide leading pg_hint_plan hints](https://github.com/DataDog/dd-trace-py/issues/19775)

이 이슈는 앞서 언급했던 Datadog의 APM-DBM Correlation과 pg-hint-plan이라는 PostgreSQL Extension이 충돌하는 문제입니다.
이번 글에서는 이 이슈를 분석하고 해결 방법을 제안하기 까지의 과정을 정리해보고자 합니다.

## 2. Datadog Correlation

우선 앞서 APM과 DBM이 어떻게 연동되는지 보겠습니다.
Datadog 공식문서 중 [Correlate Database Monitoring and Traces](https://docs.datadoghq.com/database_monitoring/connect_dbm_and_apm/?tab=postgres)를 보면, 다음과 같이 설명하고 있습니다.

> Connecting APM and DBM **injects APM trace identifiers into DBM data collection**, which allows correlation of these two data sources. This enables product features showing database information in the APM product, and APM data in the DBM product.

다시 정리하면, 백엔드 서비스가 Database를 호출할 때 현재 활성화된 APM Trace 정보를 실행할 SQL 쿼리 자체에 **주석**으로 주입합니다.
이후 DBM은 데이터베이스에서 수집한 쿼리 실행 정보에 포함된 이 주석을 함께 분석하여, 해당 쿼리를 발생시킨 APM Trace와 연결합니다.
즉, SQL 주석에 포함된 Trace 식별자가 두 데이터를 연결하는 공통 키 역할을 합니다. 이러한 방법으로 APM Trace와 DBM Telemetry를 연동할 수 있습니다.

PostgreSQL의 경우, 백엔드 서비스가 PostgreSQL에 전달하는 **쿼리 최상단**에 아래와 같이 주석으로 APM 관련 정보를 추가합니다.

```sql
/*dddb='<DATABASE>',dddbs='postgres',dde='<ENVIRONMENT>',ddh='<HOST>',ddps='<SERVICE>',ddpv='<VERSION>',traceparent='00-6a964e7600000000240ab5869b3ed43c-cbbd199fa9c836b4-01'*/
SELECT * FROM orders WHERE id = 123;
```

dd-trace-py에서 이를 담당하는 함수를 직접 확인해보겠습니다. [_database_monitoring.py](https://github.com/DataDog/dd-trace-py/blob/v4.14.0/ddtrace/propagation/_database_monitoring.py#L49-L62) 파일을 보면, `default_sql_injector`가 있습니다.
각 Database마다 Injector를 변경할 수 있으며, Postgres는 이 Default Injector를 그대로 사용하고 있습니다.

```python
def default_sql_injector(dbm_comment, sql_statement):
    # type: (str, Union[str, bytes]) -> Union[str, bytes]
    try:
        if isinstance(sql_statement, bytes):
            return dbm_comment.encode("utf-8", errors="strict") + sql_statement
        return dbm_comment + sql_statement
    except (TypeError, ValueError):
        log.warning(
            "Linking Database Monitoring profiles to spans is not supported for the following query type: %s. "
            "To disable this feature please set the following environment variable: "
            "DD_DBM_PROPAGATION_MODE=disabled",
            type(sql_statement),
        )
    return sql_statement
```

`dbm_comment + sql_statement` 를 반환하므로, 쿼리 최상단에 DBM 주석이 포함되는 것을 확인할 수 있었습니다.

## 3. PostgreSQL pg-hint-plan

PostgreSQL의 특징은 다양한 Extension을 설치하여 사용할 수 있다는 점입니다.
[pg-hint-plan](https://github.com/ossc-db/pg_hint_plan)도 PostgreSQL Extension 중 하나입니다.

pg-hint-plan은 SQL 쿼리에 힌트를 제공함으로써 사용자가 원하는 Query Plan으로 쿼리를 실행할 수 있도록 해주는 Extension입니다.
사실 이 SQL Hint라는 개념은 Orcale에서 [Optimizer Hints](https://docs.oracle.com/cd/B10500_01/server.920/a96533/hintsref.htm)라는 개념이 등장하면서 많이 사용되기 시작했습니다.
PostgreSQL은 기본적으로 Hint 기능이 지원되지 않으나, pg-hint-plan Extension을 설치하면 Oracle과 유사하게 힌트를 사용할 수 있습니다.
간단한 사용예제를 보겠습니다.

```sql
/*+
    HashJoin(a b)
    SeqScan(a)
*/
EXPLAIN SELECT *
    FROM pgbench_branches b
    JOIN pgbench_accounts a ON b.bid = a.bid
    ORDER BY a.aid;
```

pg-hint-plan의 Hint는 `/*+` 로 시작하는 주석에서 정의할 수 있습니다.
위의 예시에서는 `a` 테이블과 `b` 테이블을 JOIN 할 때 Hash Join을 사용하고, `a` 테이블을 조회할 때는 Sequential Scan을 사용하도록 지정합니다.
Postgres는 쿼리를 실행할 때마다 실행 계획(Query Plan)이 변경될 수 있으므로, pg-hint-plan은 안정적인 성능을 위해 검증된 Query Plan으로 쿼리를 실행할 수 있도록 도와주는 기능이라고 정의할 수 있습니다.

## 4. Issue 재현

이제 본격적으로 GitHub Issue의 문제에 대해 살펴보겠습니다.

[DBM SQL comments hide leading pg_hint_plan hints](https://github.com/DataDog/dd-trace-py/issues/19775)

Issue의 핵심은 아래와 같습니다.
- pg-hint-plan으로 쿼리 앞부분에 힌트를 정의하고 있습니다.
- Datadog DBM이 활성화되어 있어 DBM Comment가 쿼리 최상단에 위치합니다.
- **최상단에 위치한 DBM Comment로 인해 pg-hint-plan의 힌트가 무시됩니다.**

요약하면 DBM Comment와 pg-hint-plan의 Hint Comment가 모두 추가된 아래와 같은 최종 쿼리에서 Hint Comment가 무시되고 있는 문제입니다.

```sql
/*dddb='testdb',dddbs='postgres',dde='reproduction',ddh='postgres',ddps='dbm-pg-hint-plan',ddpv='1.0',traceparent='00-6a964e7600000000240ab5869b3ed43c-cbbd199fa9c836b4-01'*/
/*+ SeqScan(items) */
SELECT * FROM items WHERE id = 42
```

이 문제가 전역적으로 발생하고 있는지 확인하기 위해 간단한 애플리케이션과 PostgreSQL을 생성하여 테스트해보았습니다.
계획은 아래와 같습니다.

![](/assets/2026-08-30-ddtrace-contrib-pg-hint-plan/test-overview.png)

이렇게 하면, SQL문에서 DBM Comment(`/*dddb= ... */`)와 pg-hint-plan의 Hint(`/*+ SeqScan(items) */`) 가 추가된 상황에서 실제로 어떤 Query Plan을 사용했는지 `EXPLAIN` 명령어를 통해 확인할 수 있습니다.

Issue에 정의되어 있는 버전인 PostgreSQL 14와 pg-hint-plan 1.4.4를 사용하여 테스트해보니, 실제로 pg-hint-plan의 Hint가 무시된 것을 확인할 수 있었습니다.

**실제로 실행한 쿼리**

```sql
/*dddb='testdb',dddbs='postgres',dde='reproduction',ddh='postgres',ddps='dbm-pg-hint-plan',ddpv='1.0',traceparent='00-6aa4f071000000008d50e76884430175-7368cc1b40dd895d-01'*/
/*+ SeqScan(items) */
EXPLAIN (COSTS OFF) SELECT * FROM items WHERE id = 42
```

**EXPLAIN 결과**

```
Index Scan using items_pkey on items
    Index Cond: (id = 42)
```

위 테스트를 통해 이 문제가 실제로 재현 가능한 이슈임을 확인했습니다.
그렇다면 왜 pg-hint-plan의 Hint가 무시되는지를 확인해봐야 합니다.

## 5. DBM Comment와 pg-hint-plan Hint 충돌 원인

pg-hint-plan 1.4.4는 [`get_hints_from_comment`](https://github.com/ossc-db/pg_hint_plan/blob/REL14_1_4_4/pg_hint_plan.c#L1932-L2003) 함수에서 클라이언트가 전달한 원본 SQL 문자열로부터 힌트를 찾습니다. 먼저 SQL 전체에서 `/*+` 형식의 Hint Comment를 찾고, 그 앞에 있는 문자열이 허용된 문자만으로 구성되어 있는지 검증합니다.

```c
/* extract query head comment. */
hint_head = strstr(p, HINT_START);
if (hint_head == NULL)
    return NULL;

if (!pg_hint_plan_hints_anywhere)
{
    for (; p < hint_head; p++)
    {
        if (!(*p >= '0' && *p <= '9') &&
            !(*p >= 'A' && *p <= 'Z') &&
            !(*p >= 'a' && *p <= 'z') &&
            !isspace(*p) &&
            *p != '_' &&
            *p != ',' &&
            *p != '(' && *p != ')')
            return NULL;
    }
}
```

여기서 `HINT_START`는 `/*+`로 정의되어 있습니다.
따라서 함수는 strstr을 사용해 SQL 문자열 안에서 Hint Comment를 찾습니다.
다만 기본 설정에서는(pg_hint_plan.hints_anywhere = false) Hint Comment 앞에 영문자, 숫자, 공백, `_`, `,`, `(`, `)` 이외의 문자가 있으면 힌트를 찾지 못한 것으로 처리합니다.

일반적인 `EXPLAIN /*+ SeqScan(items) */ SELECT` ... 쿼리는 Hint 앞에 EXPLAIN과 공백만 존재하므로 이 검증을 통과합니다.
반면 이번 문제의 SQL은 Hint Comment 앞에 DBM Comment가 추가되어 있습니다.

pg-hint-plan은 두 번째 주석에서 `/*+`를 찾을 수는 있습니다.
하지만 그 앞부분에는 첫 번째 DBM Comment의 `/`, `*`, `'`, `=` 등의 문자가 포함됩니다.
이 문자는 허용 목록에 없으므로 위 검증문에서 return NULL이 실행되고, 결과적으로 Hint는 적용되지 않습니다.

즉, 문제의 원인은 pg-hint-plan이 단순히 첫 번째 주석만 읽기 때문이라기보다, 기본 설정에서 **Hint Comment 앞에 일반 SQL 토큰 이외의 문자가 있으면 Hint를 무시**하도록 구현되어 있기 때문입니다.

## 6. 해결 방안

문제의 원인이 Hint Comment 앞에 일반 SQL 토큰 이외의 문자(여기서는 DBM Comment)가 있기 때문임을 알았습니다.
이를 해결하기 위한 방법을 생각했을 때 가장 먼저 떠오르는 것은 Hint Comment를 DBM Comment 전에 위치시키는 것이였습니다.
그래서 DBM Comment를 넣기 전에 Hint Comment (`/*+`) 가 있는지를 검사하고, 있으면 그 뒤에 DBM Comment를 위치시키는 방법을 생각했습니다.

그런데 이 방법에는 문제가 있었습니다.

1. 쿼리를 실행하기 전에 SQL문을 순회하여 `/*+` 문자열을 찾아야 합니다. 쿼리를 실행할 때마다 검증하므로, 성능 오버헤드가 너무 큽니다.
2. DBM Comment가 최상단이 아닌 다른 곳에 위치할 경우, DBM에서 DBM Comment를 제대로 찾을 수 있을지 확실하지 않습니다. dd-trace-py 뿐만 아니라 DBM 코드도 수정이 필요할 수 있습니다.
3. 미래에 pg-hint-plan이 아닌 다른 Extension이 최상단에 Comment 추가를 요구하는 경우, 둘 중 하나는 포기해야 합니다. 확장성과 유지보수가 너무 좋지 않습니다.

그래서 DBM Comment를 최하단에 위치시키는 방법을 생각했습니다.
다만 이 경우 위 2번, 3번 문제는 동일하게 발생할 수 있습니다.
추가로, SQL 쿼리가 너무 긴 경우 Truncation이 발생할 수 있는데, DBM Comment가 최하단에 위치하게 되면 짤릴 수 있습니다.
이렇게 되면 APM - DBM Correlation이 불가능해지는 또 다른 문제를 낳을 수 있었습니다.

그래서 최선의 해결책이 무엇인지 계속 고민하며 팀원과 이 문제에 대해 논의하던 중 팀원이 "그건 pg-hint-plan 문제 아닌가요?" 라고 물어봤습니다.
그 때, 제가 너무 dd-trace-py를 수정하여 고치는 것에만 집착하고 있었다는 걸 깨달았습니다.

두 서비스가 충돌하면 둘 중 한 쪽을 해결해야 합니다.
이번 이슈의 경우 Datadog DBM과 pg-hint-plan이 충돌하고 있으므로, Datadog DBM을 수정하거나 pg-hint-plan을 수정해야 합니다.
다만, 저는 무조건 우리쪽에서 수정해서 고객에게 제공해야 한다라는 선입견을 가지고 문제에 접근하고 있었습니다.

팀원의 질문 덕분에 문제를 조금 더 넓게 바라보려고 노력했고, pg-hint-plan 쪽을 수정하면 이번 문제를 해결할 수 있음을 알아냈습니다.

### 6-1. pg_hint_plan.hints_anywhere 활성화

사실 생각해보면, SQL 최상단에 주석이 오는 것은 너무 자연스럽습니다.
예를 들어 개발자들끼리 이 SQL이 무슨 작업을 처리하는지를 설명하는 주석이 최상단에 위치할 수도 있겠죠.
그런데 Hint Comment 앞에 다른 주석이 오면 무시되는 현상이 되려 부자연스럽다는 생각이 들었습니다.

그래서 분명히 pg-hint-plan에서도 이 문제가 보고된 적이 있을 거라고 생각했고, 이를 해결하는 옵션이 제공되고 있는지를 확인했습니다.
그리고 `pg_hint_plan.hints_anywhere` 옵션에 대해서 알게 되었습니다.

`pg_hint_plan.hints_anywhere` 은 사실 이 글에서 이미 등장한 적이 있습니다.
["5. DBM Comment와 pg-hint-plan Hint 충돌 원인 분석"](#5-dbm-comment와-pg-hint-plan-hint-충돌-원인-분석) 에서 `pg_hint_plan_hints_anywhere` Boolean 변수가 이 옵션값입니다.
이 옵션이 활성화되면, Hint 앞에 어떤 문자가 있던 말던 오직 `/*+` 문자열이 있는지만 확인합니다.
테스트 결과, Hint Comment 앞에 DBM Comment가 있어도 정상적으로 Hint가 적용되는 것을 확인했습니다.

다만 이 경우, Hint Comment를 SQL문 전체에서 탐색하게 되므로 성능 오버헤드가 발생할 수 있습니다.

### 6-2. pg-hint-plan 1.7.0 업그레이드

두 번째 방법은 PostgreSQL 17과 호환되는 pg-hint-plan 1.7.0으로 업그레이드하는 것입니다.

기존 pg-hint-plan 1.4.4는 strstr()로 SQL 문자열에서 `/*+`를 찾은 뒤, Hint Comment 앞부분을 직접 순회하면서 허용된 문자만 존재하는지 검사했습니다.
이 방식에서는 DBM Comment에 포함된 `/`, `*`, `'`, `=` 등의 문자를 만나면 Hint 탐색을 중단하므로, DBM Comment 뒤에 위치한 Hint Comment를 인식하지 못했습니다.

1.7.0부터는 이 자체 문자열 탐색 로직 대신, PostgreSQL lexer 규칙을 기반으로 만든 Flex lexer를 사용하도록 변경되었습니다. ([커밋 bfb4544](https://github.com/ossc-db/pg_hint_plan/commit/bfb45447c9d4181790519c8ef4041c6b9e98fbff))
이 변경은 SQL 문자열, 일반 주석, 문자열 리터럴을 SQL 문법에 맞게 구분하여 Hint를 정확하게 찾기 위해 적용되었습니다.

따라서 아래 SQL을 lexer가 처리할 때, 첫 번째 DBM Comment는 일반 주석으로 무시되고 두 번째 `/*+ ... */` Comment가 Hint로 추출됩니다.
결과적으로 pg-hint-plan 1.7.0에서는 DBM Comment가 쿼리 최상단에 위치하더라도 Hint Comment를 정상적으로 찾고 적용할 수 있습니다.

4번 섹션에서 설명한 테스트를 통해서, pg-hint-plan 1.7.0을 사용하면 DBM Comment가 최상단에 위치해도 정상적으로 Hint Comment를 찾을 수 있음을 확인했습니다.

단, pg-hint-plan의 메이저 & 마이너 버전은 PostgreSQL 메이저 버전과 대응합니다.
즉, pg-hint-plan 1.7.0은 PostgreSQL 17용이므로, 기존 PostgreSQL 14 및 pg-hint-plan 1.4.4 환경에서는 Extension만 1.7.0으로 교체할 수 없습니다.
이 해결책은 PostgreSQL 17 업그레이드가 가능한 환경에서 선택할 수 있습니다.

## 7. 해결책 논의

위에서 찾은 2가지의 방법을 GitHub Issue에 전달했습니다.

![](/assets/2026-08-30-ddtrace-contrib-pg-hint-plan/result-share.png)

그리고 해당 문제를 겪고 있는 유저로부터 pg-hint-plan을 1.7.0으로 업그레이드 하는 옵션이 그들의 환경에 적용 가능할 것이라는 답변을 받았습니다.
처음에는 hints_anywhere 옵션과 1.7.0 업그레이드가 둘 다 필요한 줄 이해하셨던 것 같었던 것 같은데(영어 작문 이슈..), 둘 중 하나를 선택 가능하다고 하니 1.7.0 업그레이드가 문제를 해결해줄 수 있다고 전달 받았습니다.

![](/assets/2026-08-30-ddtrace-contrib-pg-hint-plan/user-response.png)

## 8. Conclusion

이렇게 dd-trace-py 레포지토리에서 또 하나의 Issue를 해결을 도울 수 있었습니다.
고객분께 서포트 티켓을 통해 지원해드리는 것 이외에도, 이런 방식으로 고객을 도울 수 있어서 뿌듯했습니다.

그리고 해결책을 찾는 과정에서 무조건 우리 쪽에서 수정해야 한다는 선입견을 버리고 넓게 보려고 노력하니 더 좋은 해결책을 찾을 수 있었다고 생각합니다. (조언해준 팀원분께 무한한 감사를..)
앞으로도 넓은 시각으로 다양한 옵션을 고려해보고 최적의 방법을 찾을 수 있도록 연습해야겠습니다.

긴 글 읽어주셔서 감사드립니다.
