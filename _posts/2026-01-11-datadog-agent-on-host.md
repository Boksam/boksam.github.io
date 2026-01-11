---
title: "Datadog Agent 호스트 설치 및 설정 가이드"
date: 2026-01-11
categories: [Infrastructure, Observability]
tags:
  - Datadog
  - Observability
  - Monitoring
  - DevOps
image: "/assets/2026-01-11-datadog-agent-on-host/datadog-logo.png"
---

> 이 포스트는 Datadog Learning Center의 'The Agent on a Host' 코스를 실습하며 정리한 내용입니다.
> 호스트 모니터링의 핵심인 Agent의 구조와 설정 방법을 다룹니다.
{: .prompt-info}

## 1. 개요: Datadog Agent란?
---

Datadog Agent는 모니터링 대상 **호스트**에서 실행되는 오픈소스 소프트웨어입니다.
단순히 데이터를 보내는 도구를 넘어, 로컬 환경에서 발생하는 다양한 신호를 수집하고 가공하여 Datadog 플랫폼으로 안전하게 전달하는 역할을 합니다.

- **핵심 역할**: 호스트의 **이벤트(Events), 로그(Logs), 메트릭(Metrics), 트레이스(Traces)**를 수집하고 Datadog Cloud로 전송합니다.
- **작동 방식**: 수집된 데이터를 15~20초 주기로 Datadog으로 전송하며, 전송 실패 시 재시도 로직과 버퍼링 기능이 내장되어 있습니다.

## 2. Agent의 4가지 핵심 구성 요소
---

Agent는 하나의 프로그램처럼 보이지만, 내부적으로는 여러 컴포넌트가 각자의 역할을 수행하며 유기적으로 동작합니다.

- **Collector**: Python으로 작성된 플러그인('Check')을 실행하여 각종 서비스(Redis, Nginx 등)의 상태를 체크하고 메트릭을 수집하는 핵심 엔진입니다. 사용자가 직접 커스텀 체크를 만들 수도 있죠.
- **Forwarder**: 수집된 모든 데이터를 Datadog 서버로 안전하게 전송합니다. 데이터 압축, 재시도, 연결 관리 등 똑똑한 기능들이 들어있습니다.
- **APM Agent (Trace Agent)**: 애플리케이션의 분산 트레이싱(Traces) 데이터를 수집합니다. 보통 TCP `8126` 포트로 애플리케이션 라이브러리에서 보낸 트레이스 정보를 수신합니다.
- **Process Agent**: 호스트에서 실시간으로 실행 중인 프로세스 정보(CPU/메모리 점유율 등)를 수집하여 Datadog의 'Live Processes' 뷰에 데이터를 제공합니다.

![](/assets/2026-01-11-datadog-agent-on-host/datadog-agent-arch.png)
_Datadog Agent 내부 구조 및 데이터 흐름 다이어그램_

## 3. 설정 우선순위 및 원격 설정
---

Datadog Agent는 다양한 설정 파일과 환경 변수를 통해 어떤 데이터를 수집할지, 어떻게 전송할지 조정할 수 있습니다.
하지만 설정 간 충돌이 발생하는 경우 아래의 우선순위에 따라 적용됩니다.

| 순위 | 설정 방법 | 설명
| --- | --- | ---
| 1위 (최상) | Remote Configuration | Datadog UI에서 실시간으로 설정을 변경 (에이전트 재배포 없이 적용)
| 2위 | 환경 변수 (Env) | `DD_API_KEY`, `DD_LOGS_ENABLED` 등. 컨테이너 환경에서 주로 사용
| 3위 (최하) | `datadog.yaml` | 호스트에 위치한 기본 설정 파일. 가장 일반적인 설정 방식

### 원격 설정 (Remote Configuration)

에이전트가 주기적으로 Datadog 서버를 **폴링(Polling)**하여 변경 사항을 가져옵니다.
위에서 보았듯이 가장 높은 우선 순위를 가지며, 보안 정책이나 동적으로 샘플링 레이트를 조절하는 등 중앙에서 여러 에이전트를 관리할 때 매우 유용합니다.

## 4. Agent 설치 및 설정
---

### 설치하기 (Linux 기준)

Datadog 플랫폼에서 제공하는 원라인 명령어를 통해 간편하게 설치할 수 있습니다.

```bash
# your_api_key_here 부분에 실제 API 키를 넣고 실행합니다.
DD_API_KEY=your_api_key_here \
DD_SITE="datadoghq.com" \
bash -c "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"
```

### 상태 점검

설치가 끝나면 가장 먼저 에이전트가 정상적으로 실행되고 있는지 확인해야 합니다.

```bash
# 에이전트 전체 상태 조회
datadog-agent status
```

위 명령어를 입력하면 굉장히 긴 출력이 나오는데요,
여기서 `Forwarder` 섹션의 `Transaction Successes` 숫자가 0 이상이고 계속 올라간다면, 에이전트가 Datadog 서버와 성공적으로 통신하고 있다는 뜻입니다.

```text
...
========
Forwarder
========
  Transactions
    ✔ Transactions: 524
    ✔ Requeued: 0
    ✔ Successes: 524  <-- 이 숫자가 올라간다면 성공적으로 데이터가 전송되고 있는 것
    ✔ Errors: 0
...
```

## 5. 주요 기능 활성화 및 트러블슈팅
---

이제 가장 많이 사용하는 로그, APM 기능을 활성화하여 로그 및 트레이스를 수집하는 방법에 대해 알아보겠습니다.

### 로그(Logs) 수집 활성화

로그 수집은 기본적으로 비활성화(`false`)되어 있습니다. `/etc/datadog-agent/datadog.yaml` 파일을 열어 아래와 같이 수정해야 합니다.

```yaml
# /etc/datadog-agent/datadog.yaml

# 1. 로그 수집 기능 활성화
logs_enabled: true

# 2. (Optional) 컨테이너 환경이라면 모든 컨테이너의 로그를 수집하도록 설정
logs_config:
  container_collect_all: true
```

> `container_collect_all: true` 설정은 Docker 환경에서 굉장히 유용하지만, 불필요한 로그까지 모두 수집할 수 있으니 주의해야 합니다.
{: .prompt-tip}

### 권한 문제 해결 (Docker)

에이전트는 `dd-agent`라는 별도의 유저 권한으로 실행됩니다.
그런데 Docker 컨테이너 로그를 수집하려면 Docker 소켓 파일(`/var/run/docker.sock`)에 접근해야 합니다.
`dd-agent` 유저는 기본적으로 이 파일에 접근할 권한이 없어서 문제가 발생합니다.

따라서 아래 명령어로 `dd-agent` 유저를 `docker` 그룹에 추가해주어야 합니다.

```bash
# dd-agent를 docker 그룹에 추가
sudo usermod -a -G docker dd-agent

# 변경사항 적용을 위해 에이전트 재시작
sudo systemctl restart datadog-agent
```

### APM Traces 수신 설정 (Docker 환경)

호스트에 Agent를 설치하고, 애플리케이션은 Docker 컨테이너에서 실행되는 경우가 많습니다.
이때 컨테이너 내부의 앱에서 보낸 트레이스(Trace)를 호스트의 Agent가 수신하려면 '외부 트래픽'을 허용해줘야 합니다.

마찬가지로 `datadog.yaml`에 다음 설정을 추가합니다.

```yaml
# /etc/datadog-agent/datadog.yaml

apm_config:
  # 외부 컨테이너로부터의 트레이스 수신을 허용
  apm_non_local_traffic: true
```

이 설정을 켜지 않으면 `localhost`에서 오는 트레이스만 받기 때문에 컨테이너 환경에서는 트레이스가 보이지 않는 문제가 발생할 수 있으므로 주의해야 합니다.

## 6. 통합(Integrations) 및 Checks 설정
---

Datadog는 **통합(Integrations)** 기능을 사용하여 Nginx, Redis, Postgres 등 700가지가 넘는 기술 스택의 메트릭과 로그를 손쉽게 수집할 수 있도록 만들어져 있습니다.

모든 통합 설정은 `/etc/datadog-agent/conf.d/` 디렉토리에서 관리됩니다. 예를 들어 Redis 통합을 설정하려면 `redisdb.d/` 디렉토리 안에 `conf.yaml` 파일을 생성하는 식입니다.

### Redis 통합 예시

1.  **설정 파일 생성**: `redisdb.d/conf.yaml.example` 파일을 `conf.yaml`로 복사하여 설정을 시작합니다.

    ```bash
    cd /etc/datadog-agent/conf.d/
    sudo cp redisdb.d/conf.yaml.example redisdb.d/conf.yaml
    ```

2.  **설정 파일 수정**: `redisdb.d/conf.yaml` 파일을 열어 아래와 같이 수정합니다. 호스트 주소와 비밀번호를 자신의 환경에 맞게 변경해주세요.

    ```yaml
    # /etc/datadog-agent/conf.d/redisdb.d/conf.yaml
    
    init_config:
    
    instances:
      - host: localhost # Redis 호스트 주소
        port: 6379      # Redis 포트
        # password: your_redis_password # 비밀번호가 있다면 주석 해제
    
    # 로그 수집 설정
    logs:
      - type: file
        path: /var/log/redis/redis-server.log # Redis 로그 파일 경로
        service: redis # Datadog에서 보여질 서비스 이름
        source: redis  # Datadog에서 사용할 로그 소스
    ```

3.  **권한 부여 및 재시작**: Agent가 Redis 로그 파일을 읽을 수 있도록 `dd-agent` 유저에게 읽기 권한을 주고, 에이전트를 재시작하여 설정을 적용합니다.

    ```bash
    # adm 그룹에 dd-agent를 추가하여 로그 파일 접근 권한 획득
    sudo usermod -a -G adm dd-agent
    
    # 에이전트 재시작
    sudo systemctl restart datadog-agent
    ```

### Postgres 통합 예시

Postgres는 데이터베이스 접속 정보뿐만 아니라, 여러 줄에 걸쳐 나오는 스택 트레이스 같은 로그를 하나의 로그로 묶어주는 설정이 중요합니다.

1.  **설정 파일 생성 및 수정**: `postgres.d/conf.yaml` 파일을 만들고 아래 내용을 참고하여 채웁니다.

    ```yaml
    # /etc/datadog-agent/conf.d/postgres.d/conf.yaml

    init_config:
    
    instances:
      - host: localhost
        port: 5432
        username: datadog
        password: your_db_password # datadog 유저의 비밀번호
        dbname: postgres # 모니터링할 데이터베이스 이름
        # 여러 데이터베이스를 모니터링하려면 relations: true 설정 추가
    
    # 로그 수집 설정
    logs:
      - type: file
        path: /var/log/postgresql/postgresql-*.log
        service: postgres
        source: postgresql
        # 멀티라인 로그 처리 규칙 (스택 트레이스 예시)
        log_processing_rules:
          - type: multi_line
            name: new_log_start_with_date
            pattern: \d{4}-\d{2}-\d{2} # YYYY-MM-DD 형식으로 시작하는 라인을 새 로그로 인식
    ```

2.  **상태 확인 및 재시작**: 설정을 마친 후에는 에이전트를 재시작하고, `integration` 명령어로 Postgres 통합이 잘 작동하는지 확인할 수 있습니다.

    ```bash
    # 에이전트 재시작
    sudo systemctl restart datadog-agent

    # 1~2분 후 Postgres 통합 상태 확인
    datadog-agent integration status postgres
    ```

    명령어 실행 결과 `OK`가 표시되면 정상적으로 설정된 것입니다.

## 7. 마치며
---

이번 포스트에서는 Datadog Agent의 내부 구조와 주요 설정 방법에 대해 알아보았습니다.
이번에 실습을 진행하면서, 한 줄 명령어로 Agent를 설치하고, 다양한 Integrations를 통해 손쉽게 모니터링 환경을 구축할 수 있다는 점이 매우 인상적이었습니다.

현재 [코드플레이스](https://code.pusan.ac.kr) 프로젝트에서 OpenTelemery + Grafana LGTM 스택을 사용하여 Observability 환경을 구축하고 있는데요,
구축이 완료되면 Datadog과 비교 분석하는 포스트도 작성해보려고 합니다.
