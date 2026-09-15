# MQTT 기반 중앙서버 → Agent 명령 전달 설계

목표: **Docker로 띄운 MQTT 브로커**를 경유해 **Python 중앙서버**가 **특정 Agent(Java daemon)** 하나를 지목해 command를 보내고, 해당 Agent만 그 command를 수신한다.

---

## 1. 전체 구성

```
┌───────────────────────┐                    ┌────────────────────────┐
│  Python 중앙서버        │                    │  Java Daemon (Agent)    │
│  (Controller)          │   ┌────────────┐   │  agentId = agent-001    │
│                        │   │ Mosquitto  │   │                         │
│  publish 전용 ─────────┼──▶│ (Docker)   │──▶│  subscribe              │
│  cmd/agent-001/req     │   │ :1883      │   │  cmd/agent-001/req      │
│                        │   │            │   │                         │
│  구독 없음              │   │            │◀──│  publish (LWT / birth)  │
│                        │   └────────────┘   │  evt/agent-001/status   │
│                        │                    │                         │
│  REST API              │                    │                         │
│  POST /api/v1/results  │◀── HTTP (범위 밖) ──│                         │
└───────────────────────┘                    └────────────────────────┘
```

| 구성요소 | 기술 | 역할 |
|---|---|---|
| Broker | `eclipse-mosquitto:2.0` (Docker) | 메시지 라우팅, 인증/ACL, 오프라인 큐잉 |
| Controller | Python 3.11 + `paho-mqtt>=2.0` | **명령 발행 전용.** 구독하지 않는다 |
| Agent | Java 17 + `org.eclipse.paho.mqttv5.client:1.2.5` | 자기 토픽 구독, 명령 실행 |

프로토콜은 **MQTT 5.0**을 기준으로 한다. 3.1.1에는 메시지 만료(Message Expiry Interval)가 없어 §3.1의 1시간 자동 소멸을 브로커가 강제할 수 없다.

> **범위 경계**: 명령 실행 **결과는 MQTT로 회신하지 않는다.** Agent가 Python 서버의 REST API로 직접 전송하며, 그 경로는 이 프로젝트 범위 밖이다(§14). 이 문서가 보장하는 범위는 **"명령이 브로커를 거쳐 Agent에 도달하는 것"** 까지다.

---

## 2. 식별자 · 토픽 설계

### 2.1 Agent 식별
- `agentId`: 전역 유일한 문자열 (예: `agent-001`, 또는 호스트명+UUID). **MQTT clientId로 그대로 사용**한다.
  - clientId가 유일해야 브로커가 기존 세션을 강제 종료(takeover)하지 않는다.
  - clientId == agentId == MQTT username 으로 통일하면 ACL 패턴이 단순해진다.

### 2.2 토픽 규칙

| 토픽 | 방향 | QoS | Retain | 용도 |
|---|---|---|---|---|
| `cmd/{agentId}/req` | 서버 → Agent | 1 | false | **명령 전달 (핵심)** |
| `evt/{agentId}/status` | Agent → (구독자 없음) | 1 | **true** | `online`/`offline` (LWT). 운영 관측용 |
| `cmd/broadcast/req` | 서버 → 전체 | 1 | false | 전체 브로드캐스트(옵션) |

`cmd/{agentId}/res` 는 **없다.** 결과는 REST로 나간다(§14).

설계 원칙:
- **토픽에 agentId를 박아 라우팅한다.** 페이로드에 목적지를 넣고 전 Agent가 필터링하는 방식은 쓰지 않는다 (불필요한 트래픽 + 정보 노출).
- **서버는 어떤 토픽도 구독하지 않는다.** 발행만 하므로 Agent가 늘어도 서버 코드·부하는 그대로다.
- Agent는 `cmd/{자기 agentId}/req` 만 구독. 다른 Agent 토픽은 ACL로 **읽기 자체가 차단**된다.
- `cmd/{agentId}/req` 는 **retain=false**. retain을 켜면 Agent 재접속 시 과거 명령이 재실행되는 사고가 난다.
- 상태 토픽은 retain=true로 유지하되 **구독자는 없다.** 비용이 사실상 0(300건 × ~120 B)이면서, 장애 시 운영자가 CLI 한 줄로 생존 여부를 확인할 수 있다:
  ```bash
  mosquitto_sub -h localhost -u central -P '<pw>' -t 'evt/+/status' -v -C 300 -W 3
  ```

---

## 3. 메시지 스키마 (JSON, UTF-8)

### 3.1 Command (`cmd/{agentId}/req`)

```json
{
  "cmdId": "0f3c1e1a-8f4a-4b2f-9a7e-2f1c9d0b7a11",
  "type": "RESTART_SERVICE",
  "args": { "serviceName": "nginx", "force": false },
  "issuedAt": "2026-09-11T08:31:02.412Z"
}
```

발행 시 MQTT 5 **`Message Expiry Interval = 3600`** 속성을 함께 싣는다(§8.1). 페이로드가 아니라 프로토콜 속성이다.

| 필드 | 필수 | 설명 |
|---|---|---|
| `cmdId` | ✔ | UUIDv4. **멱등성 키** — Agent는 최근 처리한 cmdId를 캐시해 중복 실행을 막는다 (QoS 1은 at-least-once라 중복 수신 가능). |
| `type` | ✔ | 명령 종류 (enum). Agent 측 handler 매핑 키. |
| `args` | | 명령별 파라미터 |
| `issuedAt` | ✔ | 발행 시각(UTC ISO-8601). **로깅·감사용 메타데이터일 뿐, 만료 판정에는 쓰지 않는다.** |

#### 만료 — 브로커 단독 판정

| 주체 | 기준 | 효과 |
|---|---|---|
| **브로커** | `Message Expiry Interval` **3600s** | 오프라인 큐에서 1시간 경과 시 **배달 없이 폐기**. 큐 메모리도 함께 회수된다. |

**판정 주체가 브로커 하나뿐이고, 브로커 자체 시계만 쓴다.** 중앙서버·Agent의 시계가 서로 어긋나 있어도 만료는 정확하다. 운영 환경에 NTP 동기화가 없으므로 이 성질이 중요하다.

Agent 측에는 **만료 검사 코드가 없다.** 배달된 명령은 곧 "만료되지 않은 명령"이다. 애플리케이션이 시각 비교를 하지 않으니 시계 오차로 인한 오판·디버깅 난이도가 애초에 발생하지 않는다.

> 3.1.1에는 이 기능이 없어 페이로드에 `issuedAt`/`timeoutSec`을 싣고 Agent가 직접 비교해야 했다. 메시지는 1시간 내내 큐에 남아 있다가 **결국 배달된 뒤에야** 버려졌고, 판정이 각 호스트 시계에 의존했다. MQTT 5 전환으로 두 문제가 함께 사라진다.

> ⚠️ **전제**: Agent의 **Session Expiry Interval이 메시지 만료(3600s)보다 길어야** 한다(§4에서 86400s). 세션이 먼저 사라지면 큐 자체가 없어져 만료 의미가 퇴색한다.
>
> 큐에서 대기하다 만료 전에 배달되는 경우, 브로커는 사양에 따라 **대기한 시간만큼 차감한 잔여 만료값**을 실어 보낸다. 30분 대기 후 배달되면 Agent는 1800s가 남은 메시지를 받는다.

### 3.2 Result — **범위 밖**

실행 결과는 MQTT로 회신하지 않는다. Agent가 Python 서버의 **REST API로 직접 POST** 한다. 스키마·재시도·멱등성은 그쪽 설계에 속하며, 이 문서는 **`cmdId`를 상관관계 키로 넘긴다**는 계약만 정의한다(§14).

이 결정의 귀결:
- 서버는 MQTT 구독을 하지 않는다 → §8.1이 발행 전용으로 축소된다.
- 서버가 MQTT 경로에서 얻는 보장은 **PUBACK(브로커 수신)** 까지다. Agent 도달·실행 여부는 알 수 없다.
- 미도달 감지 책임은 결과 경로(REST)로 넘어간다. 결과가 안 오면 그쪽에서 타임아웃으로 잡는다.

### 3.3 Status (`evt/{agentId}/status`, retain)

```json
{ "agentId": "agent-001", "state": "online", "version": "1.0.3", "ts": "..." }
```
- LWT(Last Will)로 `{"state":"offline"}` 을 retain=true, QoS 1로 등록 → 비정상 종료 시 브로커가 자동 발행.
- 정상 기동 시 `online` 을 직접 발행(birth message).
- **구독자는 없다.** 컨트롤러는 이 토픽을 구독하지 않으며, 주기적 하트비트도 두지 않는다(keepAlive 타임아웃으로 브로커가 대신 감지하므로 불필요). 순수하게 **운영자가 CLI로 조회하는 용도**이고, 구독자가 없으므로 상시 트래픽은 0이다.

---

## 4. 연결 옵션 (핸드셰이크 정책)

| 항목 | Agent (Java) | Controller (Python) | 이유 |
|---|---|---|---|
| 프로토콜 | **MQTT 5.0** | **MQTT 5.0** | 메시지 만료(§3.1)에 필수 |
| `clientId` | `{agentId}` | `controller-{hostname}-{pid}` | 유일성 보장 |
| `cleanStart` | **false** | true | Agent는 **오프라인 중 도착한 QoS1 명령을 브로커가 큐잉**해 재접속 시 전달해야 함 |
| `sessionExpiryInterval` | **86400s (24h)** | 0 (연결 종료 시 소멸) | ★ 명령 만료 3600s보다 길어야 큐가 유지된다 |
| `keepAlive` | 30s | 60s | NAT/LB 타임아웃보다 짧게 |
| LWT | `evt/{agentId}/status` = offline, QoS1, retain | 없음 | 운영 관측용 |
| 자동 재접속 | ON + 지수 백오프(1s→최대 60s) + jitter | ON | 브로커 재시작 시 썬더링 허드 방지 |
| 구독 | `connectComplete` 콜백에서 **매번 재구독** | **없음 (발행 전용)** | cleanStart=false라도 재구독이 안전 |

> ⚠️ **오프라인 큐잉 전제조건**: `cleanStart=false` Agent가 **최소 한 번 접속해서 구독을 완료한 이력**이 있어야 브로커가 세션을 유지하고 메시지를 쌓아준다. 세션이 없는 agentId로 발행하면 메시지는 큐잉조차 되지 않고 **조용히 버려지는데 PUBACK은 정상으로 돌아온다.**
>
> 서버가 구독을 하지 않으므로 이를 코드로 가드할 수단이 없다. 다음 두 가지로 대응한다.
> 1. **배포 순서를 운영 규칙으로 고정** — Agent를 먼저 띄워 세션을 만들고 나서 명령을 보낸다.
> 2. **미도달 감지는 REST 결과 경로에 위임** — 결과가 오지 않으면 그쪽에서 타임아웃으로 처리한다(§14).
>
> 운영자가 세션 존재 여부를 확인해야 할 때는 §2.2의 `evt/+/status` retain 조회를 쓴다.

---

## 5. 보안 (인증 · 인가)

### 5.1 인증 — 자격증명 생성과 관리

Mosquitto `password_file` 기반 계정 분리.

| 계정 | 수 | 권한 | 주입 대상 |
|---|---|---|---|
| `central` | 1 | `topic write cmd/#` — **300대 전체 명령 권한** | Python 중앙서버 |
| `agent-NNN` | 300 | 자기 토픽만 (`%u` 치환) | 각 Agent 호스트 |
| `ops` | 1 | 상태·`$SYS` 읽기 | 운영자 단말 |
| `health` | 1 | `$SYS/broker/uptime` 만 | docker healthcheck |

**Agent별 계정은 타협 불가**다. §5.2 ACL이 `%u`(접속 username) 치환에 의존하므로, 계정을 공유하면 `pattern read cmd/%u/req` 가 모두 같은 토픽으로 풀려 **아무 Agent나 남의 명령을 구독**할 수 있다. 토픽 격리가 통째로 무너진다.

#### 생성 주체 — 브로커가 아니다

Mosquitto에는 계정 발급·등록 API가 없다. Agent가 "계정을 달라"고 요청하는 흐름은 존재하지 않는다. **운영자(또는 배포 파이프라인)가 오프라인에서 만들어 뿌린다.**

```
[운영자]
  ├─ openssl rand ─▶ 평문 비밀번호
  │                    ├─▶ mosquitto_passwd ─▶ password_file (해시)  → 브로커
  │                    └─▶ Agent 설정파일 (평문)                      → Agent 호스트
  └─ SIGHUP ────────────────────────────────────────────────────────▶ 브로커 재읽기
```

해시와 평문이 **서로 다른 경로로** 나간다. 브로커는 평문을 가진 적이 없고, 발급한 적도 없다. `mosquitto_passwd` 는 브로커 데몬과 별개의 CLI이며 브로커가 떠 있지 않아도 동작한다.

#### 일괄 생성

```bash
: > broker/config/passwd                      # 새로 시작할 때만
for i in $(seq -f "%03g" 1 300); do
  PW=$(openssl rand -base64 24)               # ~144 bit. 사람이 외울 일이 없으니 길게
  docker run --rm -v "$PWD/broker/config:/mosquitto/config" eclipse-mosquitto:2.0 \
    mosquitto_passwd -b /mosquitto/config/passwd "agent-$i" "$PW"
  echo "agent-$i,$PW" >> /dev/shm/agent-creds.csv     # tmpfs. 배포 직후 파기
done
chown 1883:1883 broker/config/passwd && chmod 600 broker/config/passwd
```

- 생성 결과인 `password_file` 은 `username:$7$...` 형태의 해시 목록이다. 평문은 없다.
- 배포용 CSV는 **디스크에 남기지 않는다**(tmpfs 사용). 배포 후 즉시 삭제.
- `.gitignore` 에 `broker/config/passwd` 가 있어야 한다.

#### Agent 측 보관

| 방식 | 판단 |
|---|---|
| **설정파일 (권한 600, 서비스 계정 소유)** | **기본.** 단순하고 검증이 쉽다 |
| systemd `LoadCredential=` | 강화 시. 자격증명이 `/run/credentials/` 에만 노출되고 프로세스 트리 밖에서 안 보인다 |
| 환경변수 | **권장하지 않음.** `/proc/<pid>/environ`, `ps e`, 코어덤프, 자식 프로세스로 전파된다 |
| jar 내부 하드코딩 / 이미지 레이어 | **금지.** 300대에 같은 값이 박히고 회수가 불가능하다 |

Java 쪽에서는 읽은 뒤 로그·예외 메시지에 절대 싣지 않는다. §16.7의 연결 실패 로그가 `MqttConnectionOptions` 를 통째로 찍으면 자격증명이 로그로 샌다 — `reasonString` 만 찍는 이유가 여기에도 있다.

#### 중앙서버 측

계정은 1개지만 **가장 강력하다.** `topic write cmd/#` 는 300대 전체에 임의 명령을 보낼 수 있는 권한이다. Agent 자격증명 하나가 새면 그 Agent 하나가 위험하지만, `central` 이 새면 전체가 위험하다.

- `.env` 파일(권한 600) 또는 배포 파이프라인 시크릿으로 주입. `.gitignore` 에 `.env` 포함.
- 컨테이너로 운영한다면 `docker inspect` 로 환경변수가 그대로 보인다는 점에 유의한다.

#### 헬스체크 자격증명 — 숨기지 말고 무력화한다

§7.2 의 healthcheck는 compose 파싱 시점에 값이 컨테이너 설정에 박혀 **`docker inspect` 로 평문 노출**된다. 이를 숨기려 애쓰는 대신, **노출돼도 아무것도 못 하는 계정**을 쓴다.

```
user health
topic read  $SYS/broker/uptime      # 이 토픽 하나. 발행 권한 없음
```

> ⚠️ 여기에 `central` 을 쓰면 안 된다. `central` 은 읽기 권한이 없으므로 ACL이 구독을 거부하고 **헬스체크가 영구 실패해 컨테이너가 unhealthy 로 재시작을 반복**한다.

#### 갱신(rotation)

Mosquitto는 **SIGHUP으로 `password_file` 을 재읽기**하며 기존 연결을 끊지 않는다. 인증은 CONNECT 시점에만 일어나기 때문이다.

```bash
docker run --rm -v "$PWD/broker/config:/mosquitto/config" eclipse-mosquitto:2.0 \
  mosquitto_passwd -b /mosquitto/config/passwd agent-001 '<new-pw>'
docker kill -s HUP mqtt-broker          # 재시작 아님. 300대 연결 유지됨
```

순서가 중요하다. 브로커를 먼저 갱신하면 해당 Agent는 **접속 중에는 살아 있지만 재접속하는 순간 실패**한다. 따라서:

1. 브로커 갱신 + SIGHUP
2. 해당 Agent 설정파일 교체
3. Agent 재기동

2~3 사이에 Agent가 끊기면 복구되지 않으므로, **300대를 한 번에 돌리지 않고 배치로 나눈다.**

만료 정책은 두지 않는다 — 기계 간 인증이고 사람이 외우지 않으므로 주기적 변경의 이득이 없다. **유출 의심·담당자 변경·감사 요구 시에만** 갱신한다.

#### mTLS를 쓰지 않는 이유

§5.3에 강화 옵션으로 `use_identity_as_username true` + 클라이언트 인증서가 있고, ACL 패턴은 그대로 동작한다. 그럼에도 이 환경에서는 채택하지 않는다.

| | `password_file` | mTLS |
|---|---|---|
| 배포 대상 | Agent별 비밀번호 | Agent별 keystore(개인키 포함) — **부담은 동일** |
| **만료** | **없음** | **있음 → 300대 동시 접속 불가 위험** |
| 시계 의존 | 없음 | X.509 유효기간을 **로컬 시계로 검증** |
| 폐쇄망 갱신 | 파일 갱신 + SIGHUP | 300장 재발급·재배포 |

배포 부담이 비슷한데 **만료라는 실패 모드만 추가**된다. NTP를 쓰지 않으므로(§3.1) 시계가 밀린 호스트가 유효한 인증서를 거부할 수 있고, 폐쇄망은 만료 알림도 오지 않는다(§15.4). **전송 구간 TLS(서버 인증서)는 유지하되, 클라이언트 인증은 `password_file` 로 간다.**

#### clientId 충돌 주의

Mosquitto는 `clientId == username` 을 강제하지 않는다. agent-002가 설정 실수로 clientId를 `agent-001` 로 쓰면 ACL은 username 기준이라 토픽 접근은 막히지만, **clientId가 겹쳐 서로를 강제 종료(session takeover)** 시킨다. 두 Agent가 무한히 서로를 끊는 플래핑이 되고 §16.7의 로그 폭증으로 이어진다.

Agent 설정에서 **`agentId` 하나로부터 clientId·username을 모두 파생**시키면 구조적으로 막힌다.

### 5.2 ACL (`config/acl`)

```
# 컨트롤러: 명령 발행 전용. 읽기 권한 없음
user central
topic write cmd/#

# 운영자 계정 — CLI 로 상태 조회만
user ops
topic read  evt/#
topic read  $SYS/#

# 헬스체크 전용 — 토픽 하나만. 자격증명이 노출돼도 할 수 있는 게 없다 (§5.1)
user health
topic read  $SYS/broker/uptime

# Agent 공통 패턴 — %u 는 접속 username 으로 치환됨
pattern read  cmd/%u/req
pattern read  cmd/broadcast/req
pattern write evt/%u/status
```

이 ACL로 **agent-001은 agent-002의 명령 토픽을 구독조차 못 한다.** 토픽 기반 라우팅의 보안 이점이 여기서 나온다.

`central` 계정에 읽기 권한이 아예 없다는 점에 주의. 발행 전용 설계가 ACL 레벨에서도 강제되므로, 실수로 구독 코드가 들어가도 브로커가 거부한다.

### 5.3 운영 환경 추가 조치
- 8883 포트 + TLS(서버 인증서). 사내망이라도 `allow_anonymous false`는 필수.
- 강화 시 mTLS(클라이언트 인증서) + `use_identity_as_username true` → 인증서 CN이 username이 되어 위 ACL 패턴 그대로 동작.
- 자격증명은 Agent 측 설정파일(권한 600) 또는 환경변수로 주입. 코드/이미지에 넣지 않는다.
- 운영은 **인터넷 불가 폐쇄망**이다. 방화벽 신청과 사내 CA 인증서 발급이 선행되어야 하므로 §15를 함께 본다.

---

## 6. 디렉터리 구조

```
mqtt/
├── DESIGN.md
├── docker-compose.yml
├── broker/
│   ├── config/
│   │   ├── mosquitto.conf
│   │   ├── passwd          # mosquitto_passwd 로 생성 (git ignore)
│   │   └── acl
│   ├── data/               # 영속 세션/메시지 (git ignore)
│   └── log/
├── controller/             # Python
│   ├── pyproject.toml
│   ├── src/controller/
│   │   ├── config.py
│   │   ├── mqtt_client.py  # 접속/재접속, publish 래퍼 (구독 없음)
│   │   ├── command.py      # 스키마(pydantic), cmdId 생성, 만료 상수
│   │   └── cli.py          # send-command 엔트리포인트
│   └── tests/
└── agent/                  # Java daemon
    ├── build.gradle.kts
    └── src/main/java/com/example/agent/
        ├── AgentMain.java        # 부트스트랩, 시그널 훅
        ├── MqttConnector.java    # 접속 옵션(v5), LWT, 재구독
        ├── CommandRouter.java    # type → handler 디스패치, cmdId 중복 차단
        ├── handler/              # RestartServiceHandler, PingHandler ...
        └── ResultReporter.java   # 인터페이스만. 구현은 범위 밖 (§14.4)
```

`registry.py`(온라인 상태·pending 추적)와 `ResultPublisher`는 **없다.** 서버가 구독하지 않고 결과도 MQTT로 오지 않으므로 추적할 상태 자체가 없다.

---

## 7. 브로커 설정

### 7.1 `broker/config/mosquitto.conf`

```conf
listener 1883
protocol mqtt

allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl

# 세션/메시지 영속화 — 컨테이너 재시작에도 오프라인 큐 유지
persistence true
persistence_location /mosquitto/data/
autosave_interval 30

# 오프라인 Agent 세션 보존 기간. 명령 만료(1h)보다 충분히 길게 잡는다.
# Agent 가 CONNECT 에 실은 sessionExpiryInterval(24h, §4)의 상한으로도 작동한다.
persistent_client_expiration 7d

# 세션당 큐 상한 — 명령은 1h 에 자동 소멸하므로 1000건을 쌓을 이유가 없다 (§12.5)
max_queued_messages 100
max_inflight_messages 20
memory_limit 512MB

log_dest stdout
log_type error
log_type warning
log_type notice
log_type information
connection_messages true
```

### 7.2 `docker-compose.yml`

```yaml
services:
  broker:
    image: eclipse-mosquitto:2.0     # 폐쇄망: 사내 레지스트리 경로로 교체
    container_name: mqtt-broker
    restart: unless-stopped
    ports:
      - "1883:1883"
    volumes:
      - ./broker/config:/mosquitto/config:ro
      - ./broker/data:/mosquitto/data
      - ./broker/log:/mosquitto/log
    healthcheck:
      # central 은 읽기 권한이 없다(§5.2). health 전용 계정을 쓴다
      test: ["CMD", "mosquitto_sub", "-h", "localhost", "-p", "1883",
             "-u", "health", "-P", "${MQTT_HEALTH_PW}",
             "-t", "$$SYS/broker/uptime", "-C", "1", "-W", "3"]
      interval: 30s
      timeout: 5s
      retries: 3
```

- `data/`는 컨테이너 내 uid 1883이 써야 하므로 초기 1회 `chown -R 1883:1883 broker/data broker/log`.
- `passwd` 생성 (PoC용 최소 예시. **Agent 300개 일괄 생성과 배포·갱신 절차는 §5.1**):
  ```bash
  docker run --rm -v "$PWD/broker/config:/mosquitto/config" eclipse-mosquitto:2.0 \
    mosquitto_passwd -c -b /mosquitto/config/passwd central '<pw>'    # -c 는 최초 1회만
  docker run --rm -v "$PWD/broker/config:/mosquitto/config" eclipse-mosquitto:2.0 \
    mosquitto_passwd -b /mosquitto/config/passwd health '<pw>'        # healthcheck 전용
  docker run --rm -v "$PWD/broker/config:/mosquitto/config" eclipse-mosquitto:2.0 \
    mosquitto_passwd -b /mosquitto/config/passwd agent-001 '<pw>'
  ```
  `.env` 에 `MQTT_CENTRAL_PW`, `MQTT_HEALTH_PW` 를 둔다 (권한 600, `.gitignore` 포함).
- 로컬 개발 단계에서만 `allow_anonymous true` + ACL 미적용으로 단순화 가능. 단, PoC 통과 직후 인증을 켜는 것을 전제로 한다.
- **MQTT 5 관련 설정은 없다.** mosquitto 2.0은 3.1.1과 5.0을 같은 리스너에서 동시에 받으며, `Message Expiry Interval` 처리는 기본 동작이다. 클라이언트 쪽 프로토콜 버전만 올리면 된다(§8).

---

## 8. 구현 골격

### 8.1 Python 컨트롤러 — 명령 발행 (발행 전용)

```python
# src/controller/mqtt_client.py (요지)
import json, uuid
from datetime import datetime, timezone
import paho.mqtt.client as mqtt
from paho.mqtt.packettypes import PacketTypes
from paho.mqtt.properties import Properties

CMD_EXPIRY_SEC = 3600          # §3.1 — 브로커가 큐에서 자동 폐기

class Controller:
    def __init__(self, host, port, user, pw):
        self.c = mqtt.Client(
            mqtt.CallbackAPIVersion.VERSION2,
            client_id=f"controller-{uuid.uuid4().hex[:8]}",
            protocol=mqtt.MQTTv5,                   # ★ 메시지 만료에 필수
        )
        self.c.username_pw_set(user, pw)
        self.c.connect(host, port, keepalive=60, clean_start=True)
        self.c.loop_start()                         # PING 유지 전용. 콜백 없음

    def send(self, agent_id: str, type_: str, args: dict) -> str:
        cmd_id = str(uuid.uuid4())
        cmd = {
            "cmdId": cmd_id, "type": type_, "args": args,
            "issuedAt": datetime.now(timezone.utc).isoformat(),   # 감사용
        }
        props = Properties(PacketTypes.PUBLISH)
        props.MessageExpiryInterval = CMD_EXPIRY_SEC        # ★ 1시간 뒤 자동 소멸
        info = self.c.publish(f"cmd/{agent_id}/req", json.dumps(cmd),
                              qos=1, retain=False, properties=props)
        info.wait_for_publish(timeout=5)             # PUBACK = 브로커 수신 확인
        return cmd_id                                # 결과는 REST 측에서 cmdId 로 추적
```

`on_message`·`subscribe()`·`pending`/`results` 딕셔너리가 **전부 없다.** 그 결과:

- 콜백 스레드 블로킹 문제가 원천적으로 없다. `loop_start()`는 PINGREQ만 처리한다.
- 컨트롤러를 멀티프로세스(gunicorn 등)로 띄워도 상태 공유가 필요 없다.
- `send()`는 **fire-and-forget**이며 반환값은 `cmdId` 하나다.

> ⚠️ `wait_for_publish()` 가 보장하는 것은 **브로커가 메시지를 받았다**는 사실뿐이다. Agent 도달·실행 여부와는 무관하다(§3.2).

사용:
```bash
python -m controller.cli send --agent agent-001 --type RESTART_SERVICE \
       --args '{"serviceName":"nginx"}'
# → cmdId 출력. 결과 조회는 REST 측 API 로.
```

### 8.2 Java Agent — 명령 수신

의존성이 `org.eclipse.paho.client.mqttv3` → **`org.eclipse.paho.mqttv5.client`** 로 바뀐다. 패키지·클래스명이 달라지므로 단순 버전 업이 아니다(`MqttConnectOptions` → `MqttConnectionOptions`, `MqttCallbackExtended` → `MqttCallback`).

```java
// MqttConnector.java (요지) — org.eclipse.paho.mqttv5.client
String agentId = cfg.agentId();                       // = clientId = username
MqttClient client = new MqttClient(cfg.brokerUrl(), agentId,
        new MqttDefaultFilePersistence(cfg.persistDir()));  // QoS1 in-flight 보존

MqttConnectionOptions opts = new MqttConnectionOptions();
opts.setCleanStart(false);                            // 오프라인 큐잉 수신
opts.setSessionExpiryInterval(86400L);                // ★ 명령 만료 3600s 보다 길게
opts.setUserName(agentId);
opts.setPassword(cfg.password().getBytes(UTF_8));
opts.setKeepAliveInterval(30);
opts.setAutomaticReconnect(true);                     // 백오프 재접속
opts.setReceiveMaximum(20);                           // v3 의 setMaxInflight 대체
opts.setWill("evt/" + agentId + "/status",
        new MqttMessage(offlinePayload(agentId), 1, true, null));   // LWT

client.setCallback(new MqttCallback() {
    @Override public void connectComplete(boolean reconnect, String uri) {
        connectedAt = System.currentTimeMillis();             // §16.7 — 안정성 판정용
        try {
            client.subscribe("cmd/" + agentId + "/req", 1);   // 매 접속마다 재구독
            client.subscribe("cmd/broadcast/req", 1);
            client.publish("evt/" + agentId + "/status",
                    onlinePayload(agentId), 1, true);         // birth
            if (reconnect && !token.getSessionPresent()) {    // §16.2 — (B) 감지
                log.warn("session was lost on broker side — commands may have been missed");
            }
        } catch (MqttException e) { log.error("resubscribe failed", e); }
        // ★ 여기서 "재접속 성공" 로그를 찍지 않는다. 60초 안정 후 판정 (§16.7)
    }
    @Override public void messageArrived(String topic, MqttMessage m) {
        router.dispatch(topic, new String(m.getPayload(), UTF_8));  // 즉시 워커로 넘김
    }
    // ↓ 시간 기준 억제. 구현 전문은 §16.7 — 여기에 그대로 쓰면 플래핑 시 디스크가 찬다
    @Override public void disconnected(MqttDisconnectResponse r) { connLog.onDisconnected(r); }
    @Override public void mqttErrorOccurred(MqttException e)     { connLog.onError(e); }
    @Override public void deliveryComplete(IMqttToken t) { }
    @Override public void authPacketArrived(int reasonCode, MqttProperties props) { }
});
client.connect(opts);
```

> `sessionExpiryInterval` 을 설정하지 않으면 기본값이 0(연결 끊기면 세션 즉시 소멸)이라 **오프라인 큐잉이 통째로 동작하지 않는다.** v3의 `cleanSession=false` 와 달리 v5는 이 값을 명시해야 한다. 가장 흔한 마이그레이션 실수다.

```java
// CommandRouter.java (요지)
private final ExecutorService pool = Executors.newFixedThreadPool(4);
private final Set<String> seen =                       // cmdId 멱등성 캐시
        Collections.newSetFromMap(new LinkedHashMap<String, Boolean>() {
            protected boolean removeEldestEntry(Map.Entry<String, Boolean> e) {
                return size() > 1000;
            }
        });

public void dispatch(String topic, String json) {
    Command cmd = mapper.readValue(json, Command.class);
    synchronized (seen) {
        if (!seen.add(cmd.cmdId())) { log.info("dup {} ignored", cmd.cmdId()); return; }
    }
    // 만료 검사 없음 — 배달된 명령은 곧 만료되지 않은 명령이다 (§3.1)
    CommandHandler h = handlers.get(cmd.type());
    if (h == null) { results.report(cmd, State.REJECTED, "unknown type"); return; }

    pool.submit(() -> {                                // MQTT 콜백 스레드 블로킹 금지
        try   { results.report(cmd, State.SUCCEEDED, h.handle(cmd.args())); }
        catch (Exception e) { results.report(cmd, State.FAILED, e.getMessage()); }
    });
}
```

**핵심 규칙 2가지**

1. `messageArrived`는 Paho의 단일 네트워크 콜백 스레드에서 실행된다. 여기서 오래 걸리는 작업을 하면 keepAlive PINGRESP가 밀려 연결이 끊긴다. 반드시 **executor로 위임**한다.
2. `results` 는 **MQTT 퍼블리셔가 아니라 REST 클라이언트**다(§14). 이 프로젝트 범위에서는 인터페이스만 정의하고, 구현은 결과 경로 담당이 맡는다.
3. **연결 수명주기 콜백에서 직접 로깅하지 않는다.** 끊김/에러는 `connLog`(시간 기준 억제기, §16.7)에 위임한다. 콜백에서 `log.warn(..., e)` 를 바로 호출하면 플래핑 시 3일에 465 MB가 쌓인다.

```java
// 범위 경계 — 이 인터페이스까지가 본 설계의 책임
public interface ResultReporter {
    void report(Command cmd, State state, String payload);   // → POST /api/v1/results
}
```

### 8.3 데몬화
- systemd unit (`Type=simple`, `Restart=always`, `RestartSec=5`)로 Java 프로세스를 관리. JVM 내부에서 재접속을 처리하므로 프로세스 재시작은 크래시 대비용.
- `Runtime.getRuntime().addShutdownHook` 에서 `status=offline` 발행 → `client.disconnect(5000)` 순서로 graceful shutdown (정상 종료는 LWT가 안 뜨므로 명시 발행 필요).

---

## 9. 검증 절차 (PoC → 통합 순서)

1. **브로커 단독**
   ```bash
   docker compose up -d broker && docker compose logs -f broker
   ```
2. **CLI로 토픽 단방향 확인** (Agent/서버 코드 없이 먼저)
   ```bash
   # 터미널 A: Agent 역할 (MQTT 5)
   mosquitto_sub -h localhost -V 5 -u agent-001 -P '<pw>' -t 'cmd/agent-001/req' -q 1 -v
   # 터미널 B: 컨트롤러 역할 — -x 가 Message Expiry Interval(초)
   mosquitto_pub -h localhost -V 5 -u central -P '<pw>' -t 'cmd/agent-001/req' -q 1 -x 3600 \
     -m '{"cmdId":"t1","type":"PING","issuedAt":"2026-09-11T00:00:00Z"}'
   ```
3. **ACL 격리 확인** — `agent-002` 계정으로 `cmd/agent-001/req` 구독 시 수신 0건이어야 한다.
   `central` 계정으로 `evt/#` 구독 시에도 **거부**되어야 한다(발행 전용 강제).
4. **Java Agent 연결** → `evt/agent-001/status` = online retain 확인.
5. **Python 컨트롤러 연결** → `send()` 가 `cmdId` 를 반환하고 Agent 로그에 수신이 찍히는지 확인.
   (결과 왕복은 REST 측 검증 항목이다 — 범위 밖)
6. **오프라인 큐잉 확인** — Agent 종료 → 명령 발행 → Agent 재기동 → 명령 수신 확인.
7. **브로커 재시작 내구성** — `docker compose restart broker` 후에도 큐잉 메시지가 남는지 확인 (`persistence true` 검증).
8. **중복 확인** — 동일 cmdId 2회 발행 시 1회만 실행.
9. **★ 1시간 만료 검증** — 핵심 신규 요구사항. 실제로 1시간 기다릴 필요는 없다.
   ```bash
   # (a) 짧은 만료로 논리 검증 — Agent 중단 상태에서 발행
   mosquitto_pub -h localhost -V 5 -u central -P '<pw>' -t 'cmd/agent-001/req' \
     -q 1 -x 20 -m '{"cmdId":"exp-1","type":"PING", ...}'
   sleep 25 && <Agent 재기동>        # → 수신 0건이어야 한다

   # (b) 만료 전 재접속 — 잔여 만료값이 차감되어 배달되는지
   mosquitto_pub ... -x 60 -m '{"cmdId":"exp-2", ...}'
   sleep 10 && <Agent 재기동>        # → 수신 1건. 정상 실행되어야 한다

   # (c) 세션 만료가 메시지 만료보다 짧으면 안 된다는 것 확인
   #     Agent 의 sessionExpiryInterval 을 10s 로 낮춰 재현 → 큐 자체가 사라짐
   ```
   운영값(3600s)은 §12.8의 `$SYS/broker/heap/current` 가 시간 경과에 따라 회수되는지로 간접 확인한다.

---

## 10. 확장 · 대안 검토

| 주제 | 판단 |
|---|---|
| **MQTT 5.0** | **적용 완료(§4, §8).** `Message Expiry Interval`로 1시간 자동 소멸을, `Session Expiry Interval`로 세션 수명을 브로커가 강제한다. `Response Topic`/`Correlation Data`는 결과가 REST로 나가므로 쓰지 않는다. 라이브러리: Python `paho-mqtt 2.x`(MQTTv5), Java `org.eclipse.paho.mqttv5.client`. |
| **브로커 교체** | 수천 Agent + 클러스터링/관측성이 필요해지면 EMQX 또는 HiveMQ. Mosquitto는 단일 노드 기준 수천 커넥션까지는 충분하고 설정이 가장 단순해 시작점으로 적합. |
| **명령 이력/감사** | 브로커는 메시지 저장소가 아니다. 컨트롤러가 **발행 시점에 DB 기록**하고 MQTT는 전송 채널로만 쓴다. `cmdId`가 DB PK이며, 결과 쪽 레코드는 REST 수신부가 같은 PK로 갱신한다(§14). |
| **대규모 팬아웃** | `cmd/broadcast/req` 는 결과 폭주를 유발한다. 단 폭주가 쏟아지는 곳은 MQTT 브로커가 아니라 **REST 엔드포인트**다. Agent 측에 `0~N초` 랜덤 지연 후 결과 전송을 넣는다(범위 밖이지만 계약으로 명시할 것). |
| **Agent 등록/프로비저닝** | 초기에는 계정 수동 생성. 규모가 커지면 mosquitto의 `auth_plugin`(JWT/HTTP 백엔드) 또는 EMQX의 HTTP 인증으로 중앙 DB 연동. |
| **파일 전송류 명령** | MQTT 페이로드로 대용량을 보내지 않는다. 명령에는 presigned URL만 담고 실제 전송은 HTTP로 분리. |

---

## 11. 구현 순서 체크리스트

**선행 (운영 폐쇄망 — 리드타임이 길다. 구현과 병행 착수)**

- [ ] **방화벽 신청** 6개 항목 (§15.2)
- [ ] **사내 CA 인증서 발급 신청** — SAN에 FQDN + IP (§15.4)

**구현**

- [ ] `broker/config/{mosquitto.conf,acl}` 작성, `passwd` 생성, `docker compose up`
- [ ] §9-2, §9-3 CLI 단방향 전달 및 ACL 격리 검증 (`central` 읽기 거부 포함)
- [ ] Java Agent: **mqttv5 클라이언트로 마이그레이션** + `sessionExpiryInterval 86400` + LWT + `cmd/{agentId}/req` 구독
- [ ] Python 컨트롤러: **`protocol=mqtt.MQTTv5`** + `send()` 발행 전용 + `MessageExpiryInterval 3600`
- [ ] cmdId 멱등성 캐시 (만료 검사는 브로커가 하므로 Agent 측 구현 없음)
- [ ] 오프라인 큐잉 / 브로커 재시작 내구성 검증
- [ ] **§9-9 만료 검증** — 미배달 폐기(a), 잔여 만료 배달(b), 세션 만료 역전(c)
- [ ] `ResultReporter` 인터페이스 확정 후 결과 경로 담당에 §14 계약 전달
- [ ] systemd unit + graceful shutdown(offline 발행)
- [ ] TLS(8883) 적용 및 자격증명 외부 주입
- [ ] §12 리스크 검증 (오프라인 큐 메모리 상한, 재접속 폭주)

**폐쇄망 배포**

- [ ] §15.5 배포 순서대로 기동, `nc`/`openssl s_client` 로 경로 확인
- [ ] **1시간 유휴 후 명령 도달 테스트** — 방화벽 idle timeout 검증 (§15.3-a)

---

## 12. 부하 산정 — Agent 300개 / 컨트롤러 1개

> 결론 먼저: **MQTT 경로에는 부하라고 부를 것이 없다.** 컨트롤러가 발행만 하고 구독하지 않기 때문에, 이전 설계의 병목(응답 폭주 → 단일 콜백 스레드)이 구조적으로 사라졌다. 남은 관리 대상은 **오프라인 큐 메모리** 하나다.

### 12.1 가정

| 항목 | 값 |
|---|---|
| Agent 수 | 300 (모두 상시 접속) |
| keepAlive | Agent 30s / 컨트롤러 60s |
| Command 페이로드 | ~300 B (JSON) → 와이어 ~330 B, TCP 포함 ~390 B |
| 명령 만료 | 3600s (Message Expiry Interval) |
| LAN RTT | 0.5 ~ 1 ms |

### 12.2 정상 유휴 상태 (명령 없음)

| 항목 | 계산 | 결과 |
|---|---|---|
| 동시 TCP 커넥션 | 300 + 1 | 브로커 한도의 ~3% |
| PINGREQ/RESP | 300 / 30s | **10 회/s** (20 packets/s, ~1 KB/s) |
| 상태 메시지 | 구독자 없음 · 하트비트 없음 | **0 msg/s** |

유휴 상태에서 Mosquitto CPU는 단일 코어의 **1~3%**, 대역폭은 **수 KB/s**. 사실상 부하가 아니다.

### 12.3 명령 발행 시

**시나리오 A — 특정 Agent 1대에 명령 1건** (기본 유스케이스)

편도 ~390 B. 브로커 기여 지연 1 ms 미만. **측정 불필요 수준.**

**시나리오 B — 300대 전체 브로드캐스트** (최대 버스트)

| 단계 | 계산 | 결과 |
|---|---|---|
| 컨트롤러 publish | 300 × QoS1, inflight 20 | 300/20 × 1 ms ≈ **15 ms** |
| 브로커 패킷 처리 | (PUBLISH+PUBACK) × 2 hop × 300 | 1,200 packets, ~100 KB |
| 컨트롤러 수신 | **없음** | **0** |

이전 설계에서 문제였던 **600건 응답 폭주가 MQTT 경로에서 사라졌다.** 결과는 REST로 나가므로 부하는 그쪽으로 이동한다 — 산정과 대책은 §14를 참조해 결과 경로 담당이 맡는다.

### 12.4 컨트롤러 — 병목 없음

`paho-mqtt`의 고질적 문제였던 *"`on_message` 반환 전까지 PUBACK이 나가지 않는다"* 는 **구독을 하지 않으므로 해당 사항이 없다.** `loop_start()` 스레드는 PINGREQ와 송신 PUBACK 수신만 처리하며, 어떤 애플리케이션 코드도 그 스레드에서 돌지 않는다.

발행 측 상한만 확인하면 된다.

```
publish 처리량 ≈ max_inflight / RTT = 20 / 1 ms ≈ 20,000 msg/s
```

실제 워크로드(버스트 300건)의 **60배 여유**다. 컨트롤러를 멀티프로세스로 띄워도 공유 상태가 없어 그대로 스케일한다.

### 12.5 브로커 메모리 — 오프라인 큐가 유일한 실질 리스크

`cleanStart=false` Agent가 오프라인이면 브로커가 QoS1 명령을 세션별로 쌓는다. **컨트롤러가 상태를 구독하지 않으므로 어떤 Agent가 죽었는지 모른 채 계속 발행하게 된다.** 이 설계에서 상한 고정이 선택이 아니라 필수인 이유다.

최악의 경우(네트워크 분단으로 300대 전원 오프라인):

| `max_queued_messages` | 계산 | 메모리 |
|---|---|---|
| 1000 (기본값) | 300 × 1000 × ~400 B | **~120 MB** |
| **100 (적용값)** | 300 × 100 × ~400 B | **~12 MB** |

여기에 MQTT 5 메시지 만료가 시간축 상한으로 함께 작동한다. **1시간이 지난 명령은 브로커가 큐에서 스스로 제거**하므로, 장기 장애에서도 큐가 단조 증가하지 않는다. 3.1.1에서는 불가능했던 성질이다.

```conf
max_queued_messages 100
memory_limit 512MB        # 안전장치: 초과 시 신규 큐잉 거부
```

세션 메타데이터 자체는 클라이언트당 수 KB 수준이라 300대면 **5 MB 내외**로 무시 가능하다.

### 12.6 재접속 폭주 (thundering herd)

브로커 재시작 시 300대가 동시에 재접속한다.

| 항목 | 부하 |
|---|---|
| CONNECT + SUBSCRIBE × 2 + 상태 publish | 300 × ~5 packets ≈ 1,500 packets |
| 평문(1883) | 수백 ms 내 완료. **문제 없음** |
| TLS(8883), RSA-2048 핸드셰이크 | 300 × ~3 ms CPU ≈ **~1 초간 코어 점유** |

평문이면 무대책으로도 통과한다. TLS 사용 시 ECDSA 인증서 + 세션 재개(session resumption)를 쓰고, §4의 **재접속 지수 백오프에 jitter를 반드시 포함**한다(Paho의 `setAutomaticReconnect`는 백오프는 하지만 jitter가 없어 위상이 겹칠 수 있다 — 초기 지연을 `random(0, 5s)`로 직접 흩뿌린다).

재접속 직후 밀린 큐가 한꺼번에 배달되지만, 세션당 100건 상한이므로 최대 30,000건 × 390 B ≈ 12 MB가 분산 배달된다. 브로커 기준 수 초.

### 12.7 결론 및 리소스 산정

| 판정 | 내용 |
|---|---|
| **브로커** | Agent 300대는 Mosquitto 단일 노드 용량(~10k 커넥션)의 **3% 수준**. 교체·클러스터링 불필요. **1 vCPU / 1 GB로 충분**, 여유 두려면 2 vCPU / 2 GB. |
| **네트워크** | 유휴 수 KB/s, 최대 버스트 ~100 KB. 1 Gbps LAN 기준 고려 대상 아님. |
| **컨트롤러** | 발행 전용이므로 **병목 없음.** 여유 60배. |
| **실질 리스크 1순위** | 오프라인 큐 메모리. `max_queued_messages 100` + `memory_limit` + 메시지 만료 3600s 의 3중 상한으로 고정. |
| **범위 밖 리스크** | 브로드캐스트 시 REST 엔드포인트로 몰리는 결과 트래픽 (§14). |
| **재검토 임계점** | Agent **3,000대** 초과 시 → EMQX 전환 검토. 컨트롤러는 발행 전용이라 수평 확장에 제약이 없다. |

### 12.8 부하 검증 방법

실제 Agent 300대 없이도 검증할 수 있다.

```bash
# 300개 가상 Agent 커넥션 + 구독 유지 (mosquitto-clients / mqtt-benchmark)
mqtt-bench --broker tcp://localhost:1883 --clients 300 --qos 1 \
           --topic 'cmd/agent-%d/req' --action sub --duration 300s

# 브로커 내부 지표 — $SYS 토픽으로 실시간 관측 (ops 계정)
mosquitto_sub -h localhost -u ops -P '<pw>' -v -t '$SYS/broker/#' | \
  grep -E 'clients/connected|messages/(sent|received)|heap/current|publish/messages/dropped'
```

관측 필수 지표 3종:
- `$SYS/broker/clients/connected` — 300 유지되는가 (플래핑 없는가)
- `$SYS/broker/heap/current` — 오프라인 큐 증가 시 상한을 넘지 않는가. **만료(3600s) 경과 후 감소하는지도 함께 확인**
- `$SYS/broker/publish/messages/dropped` — **0이어야 한다.** 0이 아니면 큐 초과 = 명령 유실

> 컨트롤러 측 큐 깊이·`dropped` 카운터는 **더 이상 존재하지 않는다**(수신 경로 없음). 대신 발행 실패(`wait_for_publish` 타임아웃) 건수를 카운터로 노출한다.

---

## 13. Kafka 대비 효율

결론: **이 유스케이스(특정 Agent 1대에 명령 전달)에서는 MQTT가 압도적으로 효율적이다.** Kafka는 처리량이 부족해서가 아니라 **문제의 형태가 다르기 때문에** 맞지 않는다.

### 13.1 근본적 차이

| | MQTT | Kafka |
|---|---|---|
| 정체 | 다수 단말용 경량 **메시지 라우터** | 고성능 **분산 커밋 로그** |
| 주소 지정 | 토픽 = 주소. 계층/와일드카드 | 토픽 = 로그 파일 집합. 파티션 = 병렬 단위 |
| 클라이언트 전제 | 수만 개, 간헐 접속, NAT 뒤, 저사양 | 소수, 상시 접속, 데이터센터 내부 |
| 메시지 수명 | 전달되면 소멸. **MQTT 5는 미배달분을 지정 시간에 자동 폐기**(§3.1) | **보존 기간 내 재생 가능** |
| 접속 상태 | LWT로 프로토콜이 알려줌 | 개념 없음 |

“300대 중 **한 대**를 지목”은 MQTT의 기본 연산이고, Kafka에서는 직접 구현해야 하는 안티패턴이다.

### 13.2 Kafka로 구현하면 생기는 문제

**(a) 대상 지정 라우팅이 없다.** 두 가지 우회 방법 모두 대가가 있다.

| 방식 | 결과 |
|---|---|
| Agent별 토픽 (`cmd.agent-001`) | 300개 토픽 × RF 3 = **파티션 레플리카 900개**. 대부분 비어 있는 채로 파일 핸들·인덱스·메타데이터만 소모 |
| 단일 토픽 300 파티션 + `key=agentId` | Agent가 `subscribe()` 대신 `assign()`으로 자기 파티션을 수동 고정해야 함. 그리고 **파티션 단위 ACL이 없어 아무 Agent나 전체 파티션을 읽을 수 있다** → §5.2의 격리가 불가능 |

**(b) 부기(bookkeeping) 비용이 실제 트래픽을 넘는다.** Agent 300대가 각자 컨슈머 그룹을 가지면 오프셋 커밋만으로:

```
300 그룹 / auto.commit.interval.ms 5s ≈ 60 커밋/s  →  __consumer_offsets 쓰기
실제 명령 트래픽                      ≈ 10 msg/s
```

**관리 트래픽이 페이로드의 6배.** 여기에 그룹 코디네이터 상태 300개, 하트비트 세션 300개가 더해진다.

**(c) 오프셋 되감기 = 과거 명령 재실행.** Agent가 오래된 오프셋으로 재시작하면 밀린 명령을 전부 다시 실행한다. Kafka에는 메시지 만료가 없으므로, MQTT 5가 브로커 차원에서 공짜로 주는 1시간 자동 소멸(§3.1)을 **애플리케이션에서 직접 구현**해야 한다. 그러면 판정이 각 호스트 시계에 의존하게 되어 NTP가 없는 폐쇄망에서는 신뢰하기 어렵다. 로그의 재생 능력이 제어 평면에서는 위험 요소로 작동한다.

**(d) 접속 상태를 알 수 없다.** LWT가 없으므로 §3.3의 online/offline 판정을 하트비트 + 타임아웃 감지로 직접 만들어야 한다.

**(e) 네트워크 제약.** Kafka 클라이언트는 `advertised.listeners`의 **모든 브로커에 직접 TCP 연결**을 맺어야 한다. NAT 뒤 300대 엣지 호스트에서는 방화벽 정책이 브로커 수만큼 늘어난다. MQTT는 엔드포인트 하나(필요시 443/WSS)로 끝난다.

### 13.3 리소스 대비 (동일 워크로드: 유휴 ~0 msg/s, 버스트 300 msg)

| 항목 | Mosquitto | Kafka (KRaft) |
|---|---|---|
| 최소 프로덕션 구성 | 컨테이너 1개 | 브로커 3노드 (HA·RF3 기준) |
| 브로커 메모리 | **1 GB** (§12.7) | 노드당 **4~8 GB** JVM + 페이지 캐시 |
| 디스크 | 세션 영속화 수 MB | 로그 세그먼트. 7일 보존 × RF3 ≈ 5 GB + 관리 필요 |
| 클라이언트 라이브러리 | `paho.mqttv5` **~250 KB** | `kafka-clients` **~5 MB** + 의존성 |
| Agent 프로세스 부담 | 소켓 1개 + 콜백 스레드 1개 | 컨슈머 폴링 루프 + 브로커별 커넥션 + 코디네이터 하트비트 |
| 유휴 비용 | PINGREQ **2 바이트** / 30s | fetch 요청 상시 폴링 |
| 용량 활용률 | 3% (§12.7) | **~0.003%** — 용량의 99.997%를 안 쓰고 운영비만 지불 |
| 명령 왕복 지연 | LAN QoS1 **1~2 ms** | `acks=all` + 복제 디스크 왕복 **5~20 ms** |

### 13.4 Kafka가 이기는 영역

Kafka가 열등한 게 아니라 **다른 문제를 푼다.** 다음이 필요해지면 그때가 Kafka 차례다.

| 요구사항 | 이유 |
|---|---|
| **명령·감사 이력 영속화** | MQTT 브로커는 저장소가 아니다. 단 300대 규모면 §10대로 **컨트롤러가 DB에 기록**하는 게 더 싸고 쿼리도 된다 |
| **텔레메트리/로그 파이프라인** | Agent 로그·메트릭이 지속 1,000 msg/s를 넘고 보존·재생·집계가 필요하면 Kafka가 정답 |
| **동일 스트림의 독립 소비자 다수** | 분석·알림·아카이빙이 각자 오프셋으로 소비. MQTT는 공유 구독(MQTT 5) 없이는 불가 |
| **스트림 처리** | 윈도우 집계, 조인, exactly-once 트랜잭션 |

### 13.5 권장 아키텍처 — 필요해지면 병행

제어 평면과 데이터 평면을 분리하고, 각 평면에 맞는 도구를 쓴다.

```
         [제어 평면: 소량·저지연·대상 지정]
Python 중앙서버 ──MQTT(QoS1, expiry 1h)──▶ Mosquitto ──▶ Java Agent 300대
       ▲  │                                                      │
       │  └─▶ PostgreSQL (cmdId PK, 명령 이력·감사)              │
       └──────── REST POST /api/v1/results (범위 밖, §14) ───────┘

         [데이터 평면: 대량·보존·재생]  ※ 로그/메트릭이 실제로 문제가 될 때만
Java Agent ──MQTT(QoS0)──▶ Broker ──bridge──▶ Kafka ──▶ 분석/알림/아카이브
```

MQTT→Kafka 브리지는 직접 만들지 않는다. **EMQX 또는 HiveMQ의 Kafka 브리지 확장**, 또는 Kafka Connect MQTT Source 커넥터를 쓴다. 이 시점에 브로커를 Mosquitto에서 EMQX로 교체하는 것이 §10의 판단과 일치한다.

### 13.6 판단 요약

- **지금(Agent 300, 명령 위주)**: MQTT 단독 + DB 이력. Kafka 도입은 **운영 복잡도 3노드분을 지불하고 얻는 것이 없다.**
- **Kafka를 검토할 시점**: 텔레메트리가 지속 1,000 msg/s를 넘고 보존·재생·다중 소비자가 요구될 때. 그때도 **명령 전달 경로는 MQTT에 남긴다.**
- **절대 하지 말 것**: 제어 평면을 Kafka로 교체. Agent별 토픽 폭발, 파티션 ACL 부재, 오프셋 되감기로 인한 과거 명령 재실행을 감당하게 된다.

---

## 14. 인터페이스 경계 — 결과 경로 (범위 밖)

이 프로젝트의 책임은 **"명령이 브로커를 거쳐 Agent에 도달하는 것"** 까지다. 실행 결과는 Agent가 Python 서버의 REST API로 직접 전송하며, 그 설계·구현은 별도다. 여기서는 **양쪽이 지켜야 할 계약**만 고정한다.

### 14.1 넘기는 것 (본 설계 → 결과 경로)

| 항목 | 계약 |
|---|---|
| **`cmdId`** | UUIDv4. 명령과 결과를 잇는 **유일한 상관관계 키**. 결과 레코드는 이 값을 PK로 쓴다. |
| `agentId` | 명령을 수신한 Agent. 토픽에서 추출되며 페이로드에는 없다. |
| 전달 의미론 | **at-least-once.** Agent는 같은 `cmdId`를 두 번 받을 수 있고, 멱등성 캐시로 거른다(§8.2). |
| 만료 | 발행 후 **1시간** 내 미배달 명령은 브로커가 폐기한다. 결과가 영원히 오지 않는 정상 케이스가 존재한다. |
| 미실행 상태 | `REJECTED`(미지원 type)도 결과로 보고된다. 만료된 명령은 **Agent에 도달조차 하지 않으므로** 결과 자체가 없다. |

### 14.2 받는 것 (결과 경로 → 본 설계)

없다. **MQTT 경로는 결과를 참조하지 않는다.** 이 단방향성이 §12의 "병목 없음" 결론을 만든다.

### 14.3 결과 경로 담당이 알아야 할 것

MQTT 쪽에서 넘어가는 성질 때문에 REST 설계에 제약이 생긴다.

1. **`cmdId` 멱등성이 서버 쪽에도 필요하다.** Agent의 HTTP 재시도로 같은 결과가 중복 POST될 수 있다. `(cmdId, state)` 기준 upsert 권장.
2. **브로드캐스트 시 결과가 몰린다.** 300대 동시 실행 → 300건이 거의 같은 시각에 도착. 핸들러는 **검증 + 큐 적재만 하고 202를 반환**하고, 워커 풀이 DB 기록을 맡는 구조가 안전하다(동기 Flask + 소수 워커는 여기서 막힌다).
3. **Agent 측에 전송 전 `random(0, N초)` 지터를 넣는다.** §10의 팬아웃 항목과 동일한 대책이다.
4. **재시도 증폭에 주의.** MQTT QoS1은 브로커가 재전송을 관리했지만 HTTP는 클라이언트가 관리한다. 서버가 느려지면 일제 재시도 → 부하 증가 → 더 느려지는 자기강화 루프가 생긴다. 백오프 + 상한을 Agent 쪽에 둔다.
5. **결과 미도달 = 정상일 수 있다.** §14.1의 만료 규칙 때문이다. 타임아웃을 즉시 장애로 판정하지 않는다.
6. **"미도달 명령" 감지 책임이 여기에 있다.** MQTT 경로는 PUBACK(브로커 수신)까지만 보장하므로, 결과가 오지 않는 것을 관측할 수 있는 유일한 지점이 결과 경로다.

### 14.4 Agent 측 접점

§8.2의 `ResultReporter` 인터페이스가 경계선이다. 본 설계는 이 인터페이스를 호출하는 것까지만 정의하고, 구현체(HTTP 클라이언트, 재시도, 자격증명)는 결과 경로 담당이 제공한다.

```java
public interface ResultReporter {
    void report(Command cmd, State state, String payload);   // → POST /api/v1/results
}
```

권장 구현 지침: OkHttp **커넥션 풀 + keep-alive 재사용**(매 결과마다 TCP/TLS 핸드셰이크를 새로 맺으면 300대 동시 전송 시 서버 CPU를 태운다), 호출은 §8.2의 워커 풀 스레드에서 수행(MQTT 콜백 스레드 금지).

---

## 15. 폐쇄망(air-gapped) 운영 가이드

운영 환경은 인터넷이 차단된 폐쇄망이다. 선행 과제는 **방화벽 신청**과 **사내 CA 인증서 발급** 두 가지이며, 둘 다 리드타임이 길다. **구현보다 먼저 착수한다.**

> 아티팩트(이미지·라이브러리) 반입 절차는 이 문서의 범위가 아니다. 사내 표준 반입 프로세스를 따른다.

### 15.1 통신 흐름 — 방화벽 신청의 근거

MQTT의 가장 중요한 성질부터 짚는다.

> **모든 TCP 연결은 클라이언트(Agent·중앙서버)가 브로커로 맺는다. 브로커가 Agent에게 connect back 하는 일은 없다.**

명령이 서버 → Agent 방향으로 흐르지만, 그것은 **Agent가 이미 맺어둔 상시 연결 위로 내려가는 것**이다. 따라서 방화벽은 **Agent → 브로커 단방향만** 열면 된다. 역방향(브로커 → Agent) 정책은 **신청하지 않는다.** 이 점을 신청서에 명시하지 않으면 보안팀이 불필요한 역방향 정책을 요구하거나, 반대로 "서버가 Agent에 명령을 보내는데 왜 단방향이냐"는 반려 사유가 된다.

```
                       ┌──────────────────────────────────────┐
                       │  전부 클라이언트 → 브로커 방향 연결    │
                       └──────────────────────────────────────┘

 Python 중앙서버 ──(1) TCP 8883 ──▶ ┌───────────┐
                                    │  Broker   │
 Java Agent × 300 ─(2) TCP 8883 ──▶ │  :8883    │
        │                           └───────────┘
        │
        ├─(3) TCP 443 ──▶ Python 중앙서버    ※ 결과 전송(REST, 범위 밖)
        │
        └─(4) TCP/UDP 53 ──▶ 사내 DNS        ※ /etc/hosts 로 대체 가능
```

### 15.2 방화벽 신청서

그대로 복사해 쓸 수 있는 형태다. 출발지는 개별 IP 300개가 아니라 **대역(CIDR)으로 신청**한다.

| # | 출발지 | 목적지 | 프로토콜/포트 | 방향 | 용도 | 비고 |
|---|---|---|---|---|---|---|
| 1 | 중앙서버 IP | 브로커 IP | **TCP 8883** | 단방향 | MQTT over TLS — 명령 발행 | 상시 연결 1개 |
| 2 | Agent 대역 `10.x.y.0/24` | 브로커 IP | **TCP 8883** | 단방향 | MQTT over TLS — 명령 수신 | **상시 연결 300개** |
| 3 | Agent 대역 `10.x.y.0/24` | 중앙서버 IP | TCP 443 | 단방향 | 실행 결과 REST 전송 | 범위 밖(§14)이나 **함께 신청** |
| 4 | 중앙서버 + Agent 대역 | 사내 DNS | TCP/UDP 53 | 단방향 | 브로커 FQDN 해석 | `/etc/hosts` 고정 시 생략 가능 |
| 5 | 운영 단말 | 브로커 IP | TCP 8883 | 단방향 | CLI 점검(`mosquitto_sub`) | 운영자 대역만. 선택 |
| 6 | 배포 서버 | 사내 레지스트리 | TCP 443 | 단방향 | 이미지 pull | 초기 배포 시 1회 |

신청서 특기사항란에 넣을 문구:

```
- 본 통신은 클라이언트(Agent/중앙서버) → 브로커 방향의 단방향 TCP 세션입니다.
  브로커에서 Agent로 신규 연결을 시도하지 않으므로 역방향 정책은 불필요합니다.
- MQTT 프로토콜 특성상 세션을 장시간 유지하는 상시 연결(persistent connection)입니다.
  세션당 keepAlive 30초 간격으로 2바이트 PINGREQ가 발생합니다.
- 방화벽 TCP idle timeout 을 60초 이상으로 설정 요청드립니다. (권장 3600초)
- 평문 1883 이 아닌 TLS 8883 만 사용합니다.
```

> 평문 1883으로 먼저 PoC를 한다면 1883도 함께 신청해두고, 운영 전환 시 닫는다. **방화벽 재신청은 리드타임이 다시 발생**하므로 처음에 두 포트를 함께 올리는 편이 빠르다.

### 15.3 방화벽 설정 함정 3가지

**(a) TCP idle timeout — 가장 흔한 사고**

MQTT 연결은 명령이 없으면 30초에 한 번 2바이트 PINGREQ만 흐른다. 방화벽·L4가 이를 유휴로 보고 세션을 끊으면, **양쪽 모두 끊긴 줄 모르는 half-open 상태**가 된다. 서버는 정상 publish하고 PUBACK도 받지만 Agent에는 영영 도달하지 않는다.

- keepAlive 30s(§4) < idle timeout 이어야 한다. 방화벽 기본값이 300~3600초면 안전하다.
- 단, **세션을 끊을 때 RST를 보내지 않고 조용히 버리는(silent drop) 장비**가 있다. 이 경우 Agent는 keepAlive 타임아웃(30s × 1.5 ≈ 45초)으로 스스로 감지해 재접속한다. §4의 `setAutomaticReconnect(true)` 가 이 상황의 안전장치다.
- 검증: Agent를 붙여놓고 **1시간 이상 무명령 방치 후** 명령이 정상 도달하는지 확인한다. §9에 없던 폐쇄망 전용 항목이다.

**(b) NAT 세션 테이블 고갈**

Agent 300대가 NAT를 경유하면 상시 세션 300개가 테이블에 영구 점유된다. 일반 트래픽과 달리 **회수되지 않는다.** NAT 장비의 세션 상한과 현재 사용률을 신청 단계에서 확인한다. 300개는 보통 문제없지만, 장비를 공유하는 다른 시스템이 있으면 고지가 필요하다.

**(c) IPS/DPI 오탐**

1883/8883은 IANA 등록 포트(`mqtt`/`secure-mqtt`)라 대부분의 장비가 인식한다. 다만 애플리케이션 검사가 켜져 있으면 다음이 발생할 수 있다.

- MQTT 시그니처가 없는 장비가 미상 프로토콜로 분류해 차단
- 페이로드 재조립 과정에서 지연 발생

**TLS(8883)를 쓰면 DPI가 내용을 못 보므로 대부분 우회된다.** 폐쇄망이라도 TLS를 권장하는 실무적 이유가 하나 더 있는 셈이다. 차단이 의심되면 브로커 로그의 `connection_messages`(§7.1)와 Agent 측 `connectionLost` 로그를 대조한다.

### 15.4 TLS 인증서 — 폐쇄망에서는 사내 CA

Let's Encrypt 등 공인 CA의 자동 발급이 불가능하다. 두 가지 중 택일한다.

| 방식 | 판단 |
|---|---|
| **사내 CA 발급** | 권장. 이미 사내 PKI가 있으면 브로커 서버 인증서만 발급받으면 된다 |
| self-signed | PoC용. 만료 관리가 수동이라 300대 배포 시 갱신이 지옥이 된다 |

발급 시 **SAN(Subject Alternative Name)에 브로커의 FQDN과 IP를 모두 포함**시킨다. 폐쇄망에서는 DNS 없이 IP로 접속하는 경우가 잦은데, SAN에 IP가 없으면 인증서 검증이 실패한다.

Agent 측 truststore 등록:
```bash
# Java — 사내 CA 를 truststore 에 import
keytool -importcert -alias corp-ca -file corp-ca.crt \
        -keystore agent-truststore.jks -storepass '<pw>' -noprompt
```
```python
# Python — paho
self.c.tls_set(ca_certs="/etc/pki/corp-ca.crt")
```

인증서 만료일을 **배포 시점에 자산 목록으로 기록**한다. 폐쇄망은 만료 알림이 오지 않아 그대로 서비스가 멈춘다.

### 15.5 배포 순서 (폐쇄망 반영)

§9의 검증 절차 앞에 붙는 선행 단계다.

1. 방화벽 신청 (§15.2) — **리드타임 확인. 가장 먼저 착수**
2. 사내 CA 인증서 발급 신청 (§15.4) — SAN에 FQDN + IP
3. 브로커 기동 → **방화벽 소통 확인**
   ```bash
   # Agent 호스트에서 — 애플리케이션 없이 경로만 먼저 확인
   nc -zv <broker-ip> 8883
   openssl s_client -connect <broker-ip>:8883 -CAfile corp-ca.crt </dev/null
   ```
4. §9 검증 절차 수행
5. **§15.3-(a) 장시간 유휴 테스트** — 1시간 방치 후 명령 도달 확인

### 15.6 착수 전 체크리스트

- [ ] 방화벽 신청 6개 항목 접수 (§15.2) — DNS 항목 필요 여부 확인
- [ ] 방화벽 TCP idle timeout ≥ 60초 확인 (§15.3-a)
- [ ] NAT 경유 여부 및 세션 테이블 여유 확인 (§15.3-b)
- [ ] Agent 대역 CIDR 확정 (개별 IP 300개 신청 금지)
- [ ] 사내 CA 인증서 발급 — SAN에 FQDN **및 IP** 포함 (§15.4)
- [ ] `corp-ca.crt` — Java truststore / Python ca_certs 양쪽 배포
- [ ] 인증서 만료일 자산 등록

---

## 16. 브로커 장애 시 동작

### 16.1 장애를 두 가지로 나눈다

| | **(A) 브로커 재시작** | **(B) 브로커 데이터 유실** |
|---|---|---|
| 상황 | 프로세스/컨테이너 재기동. `broker/data/` 온전 | 볼륨 손상·삭제, 신규 인스턴스로 재구축 |
| 세션 | `persistence true` 로 **복원됨** | **전부 소멸** |
| 오프라인 큐 | 복원됨 | 소멸 |
| 복구 후 발행 | 정상 큐잉 | **세션이 없어 조용히 폐기** (§4 경고) |
| 빈도 | 흔함 (패치·재기동) | 드묾 (디스크 장애·운영 실수) |

(A)는 **무대책으로 통과한다.** 설계상 이미 대비돼 있다. 판단이 필요한 것은 (B)와, 두 경우 공통인 **다운 구간 동안의 발행 처리**다.

> (B)를 예방하는 것이 가장 싸다. `broker/data/` 를 반드시 **영속 볼륨**에 두고 백업 대상에 포함한다. 이 디렉터리를 잃으면 300대의 세션이 동시에 사라진다.

### 16.2 Java Agent — 판단할 것이 거의 없다

Agent는 **순수 소비자**라서 다운 중 할 일이 없다. §4·§8.2 설정으로 이미 충족된다.

| 상황 | 동작 | 근거 |
|---|---|---|
| 연결 끊김 감지 | `disconnected()` 콜백. silent drop이면 keepAlive 타임아웃(~45초) | §4 keepAlive 30s |
| 재접속 | 지수 백오프(1s→60s) **+ jitter** 자동 | §4 `setAutomaticReconnect` |
| 재접속 성공 | `connectComplete`에서 **매번 재구독** + birth 발행 | §8.2 |
| **실행 중이던 명령** | **그대로 완료된다.** 결과는 REST로 나가므로 브로커와 무관 | §14 |

마지막 항목이 채널 분리 설계의 뜻밖의 이점이다. 결과가 MQTT로 회신되는 구조였다면 브로커 다운 시 **실행은 끝났는데 결과를 보낼 수 없는** 상태가 됐을 것이다. 지금은 브로커가 죽어도 진행 중인 작업과 결과 보고가 멈추지 않는다.

**추가로 구현할 것 하나** — 재접속 시 세션 존재 여부를 확인해 경고를 남긴다.

```java
// 재접속했는데 브로커에 세션이 없었다 = (B) 상황 = 명령을 놓쳤을 수 있다
if (reconnect && !token.getSessionPresent()) {
    log.warn("session was lost on broker side — commands may have been missed");
    // 필요 시 REST 로 재동기화 요청 (범위 밖)
}
```

Agent가 (B)를 인지할 수 있는 **유일한 지점**이다. 중앙서버는 구독을 하지 않아 알 방법이 없다.

### 16.3 Python 중앙서버 — 여기만 판단이 필요하다

핵심 질문: **브로커가 죽어 있는 동안 `send()` 가 호출되면 어떻게 하는가?**

먼저 하지 말아야 할 것부터. 현재 §8.1 코드는 이 상황에서 잘못 동작한다.

```python
info = self.c.publish(...)
info.wait_for_publish(timeout=5)     # ← 브로커 다운 시 5초를 그냥 버린다
```

`paho`는 연결이 없으면 `MQTT_ERR_NO_CONN`(rc=4)을 담은 `MQTTMessageInfo`를 돌려주고, `wait_for_publish()`는 타임아웃까지 매달린다. 브로드캐스트 300건이면 **25분간 블로킹**된다. 게다가 paho의 내부 재전송 큐는 **메모리에만 있어 프로세스가 죽으면 사라진다.** 여기에 durability를 기대하면 안 된다.

**판단**: 연결 상태를 먼저 확인해 **즉시 실패시키고**, 명령은 DB 아웃박스에 남긴다.

```python
class BrokerUnavailable(RuntimeError): pass

def send(self, agent_id: str, type_: str, args: dict) -> str:
    cmd_id = str(uuid.uuid4())
    cmd = {"cmdId": cmd_id, "type": type_, "args": args,
           "issuedAt": datetime.now(timezone.utc).isoformat()}

    db.insert_command(cmd_id, agent_id, cmd, status="PENDING")   # ★ 발행 전에 기록
    if not self._publishable():
        return cmd_id                    # 아웃박스에 남음. 복구 워커가 발행한다
    self._publish(agent_id, cmd, expiry=CMD_EXPIRY_SEC)
    db.mark(cmd_id, "PUBLISHED")
    return cmd_id

def _publishable(self) -> bool:
    return self.c.is_connected() and time.monotonic() >= self._grace_until
```

§10이 이미 *"컨트롤러가 발행 시점에 DB 기록"* 을 요구하고 있으므로, `status` 컬럼 하나와 복구 워커만 추가하면 된다. 새 컴포넌트가 아니다.

> **구독하지 않는 것과 연결 상태를 아는 것은 다르다.** `is_connected()`·`on_disconnect`는 구독이 아니라 커넥션 상태다. ACL(§5.2)에도 저촉되지 않는다. 발행 전용 원칙을 지키면서 브로커 다운을 감지할 수 있다.

### 16.4 아웃박스 복구 — 만료를 잔여 시간으로 재계산

복구 워커가 `PENDING` 을 재발행할 때, 1시간 만료를 **처음부터 다시 세면 안 된다.**

```python
def _replay_outbox(self):
    for row in db.fetch_pending():
        elapsed = time.time() - row.issued_at_epoch      # 컨트롤러 자기 시계만 사용
        remaining = CMD_EXPIRY_SEC - int(elapsed)
        if remaining <= 0:
            db.mark(row.cmd_id, "EXPIRED_BEFORE_PUBLISH")   # 발행조차 하지 않는다
            continue
        self._publish(row.agent_id, row.payload, expiry=remaining)
        db.mark(row.cmd_id, "PUBLISHED")
```

이렇게 해야 "발행 후 1시간" 이 아니라 **"명령 생성 후 1시간"** 이라는 의미가 유지된다. 브로커가 2시간 다운됐다면 밀린 명령은 재발행 없이 `EXPIRED_BEFORE_PUBLISH` 로 정리된다.

시각 비교가 등장하지만 **컨트롤러 단일 호스트의 자기 시계 안에서만** 이뤄진다(자기가 기록한 `issuedAt` vs 자기 `now`). §3.1이 제거한 것은 *호스트 간* 시계 비교이므로 원칙에 어긋나지 않는다.

### 16.5 복구 직후 유예 시간 — (B) 대비

브로커가 살아나면 컨트롤러가 먼저 붙는다(재접속 1개 vs 300개). 이때 (B) 상황이면 **Agent 세션이 아직 없어 발행분이 조용히 폐기된다.**

컨트롤러는 구독을 하지 않으므로 Agent 복귀를 확인할 수단이 없다. 따라서 **시간으로 방어한다.**

```python
def _on_connect(self, client, userdata, flags, rc, props=None):
    self._grace_until = time.monotonic() + RECONNECT_GRACE_SEC   # 기본 60초
```

Agent 백오프 상한이 60초 + jitter(§4)이므로 대부분 이 안에 복귀한다. 유예 중 들어온 `send()` 는 아웃박스에 쌓였다가 §16.4로 발행된다.

(A)에서는 세션이 살아 있어 유예가 불필요하지만, 컨트롤러는 (A)와 (B)를 구분할 수 없다. 60초는 싼 보험이다. 명령 지연이 문제되는 워크로드라면 `RECONNECT_GRACE_SEC` 를 낮추되 0으로 두지는 않는다.

### 16.6 정리 — 역할 분담

| 주체 | 다운 중 | 복구 시 |
|---|---|---|
| **브로커** | — | `persistence` 로 세션·큐 복원 (A) |
| **Agent** | 하던 작업 계속, 결과는 REST로 정상 전송, 백오프 재접속 | 재구독 + birth. `sessionPresent=false` 면 경고 |
| **중앙서버** | `send()` 즉시 실패 대신 **아웃박스 적재**. 5초 블로킹 금지 | 60초 유예 후 잔여 만료로 재발행 |

**단일 장애점은 브로커 하나다.** Agent 300대·중앙서버는 브로커 없이도 자기 상태를 잃지 않는다. Mosquitto는 클러스터링이 없으므로(§10), 가용성이 요구 수준에 못 미치면 그때가 EMQX 전환 시점이다.

### 16.7 장기 다운 시 로그 폭증 — 디스크 고갈

**실재하는 위험이다.** 브로커가 오래 죽어 있으면 Agent 300대가 각자 재접속 실패 로그를 계속 쏟아낸다. §8.2 콜백은 한 사이클에 `disconnected` → `mqttErrorOccurred` → (성공 시) `connectComplete` 세 곳에서 로그를 남기며, 그중 둘은 **예외 객체를 함께 넘겨 스택트레이스 전체를 출력**한다.

#### 얼마나 쌓이는가

`MqttException` 스택트레이스를 ~30줄 / ~120 B 로 잡으면:

| 상황 | 이벤트 빈도 | 호스트당 | **중앙 수집 시 (×300)** |
|---|---|---|---|
| 정상 백오프 (상한 60s 도달) | 1,440 /일 | ~5 MB/일 | 1.5 GB/일 |
| **플래핑 (2초 주기)** | 43,200 /일 | **~155 MB/일** | **46 GB/일** |

**정상 백오프는 감당 가능하다. 문제는 플래핑이다.** 백오프가 상한 60초에 도달하지 못하고 짧은 주기로 도는 경우가 실제로 더 흔하다.

- 브로커가 **살아 있으나 불안정** — TCP 연결은 맺어지고 CONNACK 직후 끊김 → 백오프가 매번 1초로 리셋
- **방화벽 idle timeout / silent drop** (§15.3-a) — 브로커는 멀쩡한데 세션만 계속 잘림
- 인증 실패·ACL 오설정 — 연결은 되는데 구독이 거부되어 재시도 루프

이 경우 **브로커는 정상으로 보이는데 300대의 디스크가 조용히 차오른다.** 모니터링이 브로커만 보고 있으면 놓친다.

#### 연쇄 효과

디스크가 차면 로그만 멈추는 게 아니다.

1. Paho `MqttDefaultFilePersistence`(§8.2) 쓰기 실패 → `MqttPersistenceException` → **로그를 더 남기려 시도**
2. 명령 핸들러의 작업(파일 생성·서비스 재시작) 실패
3. 결과 REST 전송 실패 → 재시도 → 또 로그

**로그 파티션은 애플리케이션·Paho 영속화 디렉터리와 분리**하거나, 최소한 상한을 고정한다.

#### 대책의 순서 — 상한은 맨 마지막이다

크기 상한만 걸면 **노이즈가 신호를 밀어낸다.** 플래핑이 하루 155 MB인데 `totalSizeCap` 이 100 MB면 보존 기간이 15시간으로 무너지고, 정작 필요한 명령 실행 기록이 재접속 실패 로그에 덮여 사라진다. 상한은 문제를 디스크 고갈에서 **증거 인멸로 바꿀 뿐**이다.

순서는 이렇다.

1. **노이즈를 만들지 않는다** (근본)
2. **노이즈와 신호를 분리해 서로 밀어내지 못하게 한다** (핵심)
3. 상한은 그 위에 얹는 최후 방어선

#### 대책 1 — 시간 기준 억제 (근본)

> ⚠️ **횟수 기준 카운터는 플래핑을 못 막는다.** "연속 실패 N회마다 1줄" 방식은 재접속이 성공할 때마다 카운터가 0으로 리셋되므로, 끊김→연결→끊김이 반복되면 **매 사이클이 '첫 실패'로 취급되어 스택트레이스가 그대로 찍힌다.** 2초 주기 플래핑에서는 억제 효과가 사실상 없다.

기준을 **횟수가 아니라 시간**으로 잡는다. 그리고 짧게 붙었다 끊기는 연결은 **복구로 인정하지 않는다.**

```java
private static final long STABLE_MS = 60_000;      // 이만큼 유지돼야 "복구"로 인정
private static final long REPORT_MS = 3_600_000;   // 불안정 지속 중 보고 주기(1시간)

private volatile long connectedAt   = 0;
private volatile long unstableSince = 0;           // 0 이면 안정 상태
private volatile long lastReport    = 0;
private final AtomicInteger events  = new AtomicInteger();

@Override public void disconnected(MqttDisconnectResponse r) {
    long now = System.currentTimeMillis();
    events.incrementAndGet();

    if (unstableSince == 0) {                      // 안정 → 불안정 최초 전환. 여기서만 상세히
        unstableSince = now;
        lastReport    = now;
        log.warn("MQTT unstable: disconnected ({})", r.getReasonString(), r.getException());
    } else if (now - lastReport >= REPORT_MS) {    // 지속 중이면 1시간에 딱 한 줄
        lastReport = now;
        log.warn("MQTT still unstable for {}m — {} events, last: {}",
                 (now - unstableSince) / 60_000, events.get(), r.getReasonString());
    }
    // 그 외에는 아무것도 하지 않는다. log.debug() 호출조차 없다
}

@Override public void connectComplete(boolean reconnect, String uri) {
    connectedAt = System.currentTimeMillis();
    resubscribe();                                  // §8.2 — 재구독은 매번 해야 한다
    // ★ 복구 로그를 여기서 찍지 않는다. 곧 다시 끊길 수 있다(플래핑)
}

// 연결이 STABLE_MS 이상 유지됐을 때만 복구로 판정한다
scheduler.scheduleAtFixedRate(() -> {
    long now = System.currentTimeMillis();
    if (unstableSince != 0 && client.isConnected() && now - connectedAt >= STABLE_MS) {
        log.warn("MQTT recovered — was unstable {}m, {} events",
                 (now - unstableSince) / 60_000, events.getAndSet(0));
        unstableSince = 0;
    }
}, 30, 30, TimeUnit.SECONDS);
```

**핵심 두 가지**

1. **`connectComplete` 에서 로그를 찍지 않는다.** 플래핑의 로그량 절반이 "재접속 성공" 메시지에서 나온다. 60초 이상 안정됐을 때만 복구로 보고한다.
2. **억제된 경로는 `log.debug()` 도 호출하지 않는다.** DEBUG 레벨이 꺼져 있어도 문자열 포매팅과 호출 비용은 남고, 실수로 DEBUG를 켜는 순간 폭증한다. 아예 분기 밖으로 뺀다.

#### 실제로 남는 로그

브로커가 **3일간 다운되거나 3일간 2초 주기로 플래핑**해도 결과는 같다.

```
2026-09-15 02:10:03 WARN  MQTT unstable: disconnected (Connection lost)
    org.eclipse.paho.mqttv5.common.MqttException: Connection lost
    ... (스택트레이스 — 최초 1회뿐)
2026-09-15 03:10:07 WARN  MQTT still unstable for 60m — 1823 events, last: Connection lost
2026-09-15 04:10:11 WARN  MQTT still unstable for 120m — 3644 events, last: Connection lost
...
2026-09-18 02:11:40 WARN  MQTT recovered — was unstable 4321m, 129604 events
```

| 상황 (3일 지속) | 억제 전 | **억제 후** |
|---|---|---|
| 정상 백오프 (60s) | ~15 MB | **~74줄 / ~10 KB** |
| 플래핑 (2초 주기) | ~465 MB | **~74줄 / ~10 KB** |

시간 기준이므로 **이벤트가 아무리 잦아져도 로그량이 늘지 않는다.** 그러면서 `events` 카운터가 실제 발생 횟수를 보존하므로 "1시간에 1,823회 끊겼다 = 플래핑" 이라는 진단 정보는 그대로 남는다. 횟수 기준 억제로는 얻을 수 없는 정보다.

#### Paho 자체 로거

위는 우리 래퍼의 로그다. Paho 라이브러리도 내부적으로 재접속 시도를 로깅하므로 별도로 막아야 한다.

```xml
<logger name="org.eclipse.paho" level="ERROR" additivity="false">
  <appender-ref ref="MQTTCONN"/>
</logger>
```

`WARN` 이 아니라 `ERROR` 로 내린다. 재접속 실패는 우리 래퍼가 이미 정확히 보고하므로 Paho의 중복 보고는 가치가 없다. 완전히 끄려면 `OFF` 도 무방하다.

> Paho는 기본적으로 JUL(`java.util.logging`)을 쓴다. logback 설정만으로 안 잡히면 `jul-to-slf4j` 브리지를 추가하거나 `logging.properties` 에서 직접 레벨을 내린다. **설정했다고 가정하지 말고 실제로 억제되는지 확인한다.**

#### 완전히 끄는 선택지

`com.example.agent.MqttConnector` 로거를 `OFF` 로 두면 연결 로그가 **한 줄도 남지 않는다.** 다만 권장하지 않는다 — 위 구성의 비용이 3일에 74줄(~10 KB)이고, 그 74줄이 "언제부터 불안정했고 몇 번 끊겼는지"라는 **장애 분석의 유일한 단서**다. 이걸 지우면 디스크는 아끼지만 원인 규명이 불가능해진다.

#### 대책 2 — 노이즈와 신호를 분리 (핵심)

억제를 해도 예상 못 한 경로에서 노이즈가 터질 수 있다. 그때도 **명령 실행 기록은 살아남아야 한다.** 파일과 상한을 따로 준다.

```xml
<!-- logback.xml -->

<!-- (1) 애플리케이션 로그 — 진단 가치가 높다. 길게 보존 -->
<appender name="APP" class="ch.qos.logback.core.rolling.RollingFileAppender">
  <file>/var/log/agent/agent.log</file>
  <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
    <fileNamePattern>/var/log/agent/agent.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
    <maxFileSize>10MB</maxFileSize>
    <maxHistory>30</maxHistory>
    <totalSizeCap>500MB</totalSizeCap>
  </rollingPolicy>
</appender>

<!-- (2) MQTT 연결 수명주기 — 노이즈. 작게 가두고 따로 버린다 -->
<appender name="MQTTCONN" class="ch.qos.logback.core.rolling.RollingFileAppender">
  <file>/var/log/agent/mqtt-conn.log</file>
  <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
    <fileNamePattern>/var/log/agent/mqtt-conn.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
    <maxFileSize>10MB</maxFileSize>
    <maxHistory>3</maxHistory>
    <totalSizeCap>50MB</totalSizeCap>
  </rollingPolicy>
</appender>

<!-- ★ additivity="false" — 연결 로그가 APP 으로 넘어가지 않게 한다. 이것이 분리의 핵심 -->
<logger name="com.example.agent.MqttConnector" level="INFO" additivity="false">
  <appender-ref ref="MQTTCONN"/>
</logger>
<logger name="org.eclipse.paho" level="ERROR" additivity="false">   <!-- 대책 1 참조 -->
  <appender-ref ref="MQTTCONN"/>
</logger>

<root level="INFO">
  <appender-ref ref="APP"/>
</root>
```

`additivity="false"` 가 빠지면 연결 로그가 **양쪽 파일에 모두 쌓여** 분리가 무의미해진다. 가장 흔한 실수다.

이렇게 두면:

- 플래핑이 아무리 심해도 차는 것은 `mqtt-conn.log` 뿐이고, 그건 **버려도 되는 로그**다
- `agent.log` 는 명령 실행 기록만 담으므로 증가량이 **명령 발행량에 비례**한다. 유휴 시 거의 0, 브로드캐스트 300건이라야 수백 줄. 30일 보존이 현실적이다
- 장애 분석 시 두 파일을 시각으로 대조하면 "언제 끊겼고 그동안 무슨 명령이 밀렸는지"가 오히려 더 잘 보인다

#### 대책 3 — 상한은 최후 방어선

위 두 대책이 동작하면 `totalSizeCap` 에는 도달하지 않는다. 그래도 남겨두는 이유는 **예상하지 못한 로그 경로**(서드파티 라이브러리, 핸들러 구현) 때문이다. 상한이 없으면 디스크 고갈로 §16.7 서두의 연쇄 효과가 발생한다.

stdout → systemd journald 구성이라면 `journald.conf` 의 `SystemMaxUse=` 가 같은 역할을 한다(기본 `/var` 의 10%). 다만 journald는 **단일 저장소라 노이즈/신호 분리가 안 되므로**, 위 (1)(2) 분리를 쓰려면 파일 출력이 낫다.

#### 대책 4 — 디스크 사용률 감시

위 대책들은 Agent가 **자기 로그로** 디스크를 채우는 것만 막는다. 다른 프로세스가 채우는 것까지는 못 막으므로, 호스트 디스크 사용률은 별도 감시 대상으로 둔다. 이 설계에서는 중앙서버가 Agent 상태를 구독하지 않으므로 **디스크 경고를 받을 경로가 없다** — 기존 사내 인프라 모니터링에 위임한다.

### 16.8 운영 점검 항목

- [ ] `broker/data/` 가 영속 볼륨이며 백업 대상인가 — **(B) 예방이 최우선**
- [ ] 컨트롤러가 브로커 연결 상태를 헬스체크/메트릭으로 노출하는가
- [ ] 아웃박스 `PENDING` 건수·최고 적체 시간을 메트릭으로 노출하는가
- [ ] `EXPIRED_BEFORE_PUBLISH` 발생 시 알림이 가는가 (명령이 실행되지 않았다는 뜻)
- [ ] 브로커 재시작 후 **명령 도달**까지 확인하는 런북이 있는가 (컨테이너 기동 확인만으로는 부족)
- [ ] (B) 복구 런북 — ops 계정으로 `$SYS/broker/clients/connected` 가 300에 도달했는지 확인 후 발행 재개
- [ ] **재접속 실패 로그가 시간 기준으로 억제되는가** (§16.7 대책 1) — 횟수 기준은 플래핑에서 무력하다
- [ ] `connectComplete` 에서 복구 로그를 찍지 않는가 — 60초 안정 후에만 보고
- [ ] **MQTT 연결 로그가 별도 appender로 분리돼 있는가** (§16.7 대책 2) — `additivity="false"` 확인
- [ ] `agent.log` 보존 기간이 노이즈에 의해 잠식되지 않는가 (30일 유지)
- [ ] `org.eclipse.paho` 로거가 **ERROR 이상**이며 MQTTCONN 으로만 나가는가 (JUL 브리지 확인)
- [ ] 두 appender 모두 `totalSizeCap` 이 설정돼 있는가 — rotate 없는 파일 출력 금지
- [ ] 로그 파티션이 Paho 영속화 디렉터리·애플리케이션 작업 영역과 분리돼 있는가
- [ ] 호스트 디스크 사용률이 사내 모니터링 대상인가 — **중앙서버는 이를 알 수 없다**
