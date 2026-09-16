# MQTT 브로커 운영 설치 가이드

대상: **Docker Compose 단일 노드 / 폐쇄망 / Agent 300대 / 평문 1883**
이 문서만으로 설치가 끝나도록 썼다. 각 절 머리에 **실행 위치**가 있고, 명령마다 **기대 출력**을 함께 적었다.

---

## 0. 전제와 범위

| 항목 | 값 |
|---|---|
| 브로커 | `eclipse-mosquitto:2.0` (2.0.22 검증) |
| 프로토콜 | MQTT 5.0 |
| 리스너 | **평문 1883 단일** (TLS 미사용 — §0.1) |
| 인증 | `password_file` + ACL |
| 실행 | Docker Compose, 단일 노드 (Swarm 미사용) |
| 인스턴스 | **1개.** Mosquitto 는 클러스터링을 지원하지 않는다 |
| 네트워크 | 인터넷 불가 폐쇄망 |

### 0.1 TLS 미적용에 따른 전제

운영 결정으로 TLS를 쓰지 않는다. 설치·운영 시 다음을 사실로 인지하고 간다.

- **MQTT 비밀번호는 CONNECT 패킷에 평문으로 전송된다.** MQTT에는 challenge-response가 없다. 경로상에서 패킷을 캡처하면 자격증명이 그대로 노출되고, 그 계정으로 접속하면 ACL도 의미가 없다.
- 토픽명(`cmd/{agentId}/req`)과 명령 페이로드도 평문으로 흐른다.
- 따라서 **`allow_anonymous false` + ACL은 "외부 침입 방어"가 아니라 "내부 오작동 방지"로 기능한다.** 잘못 짠 Agent가 남의 토픽을 구독하는 것은 막지만, 트래픽을 볼 수 있는 상대는 막지 못한다.

**평문 전제에서 실효가 있는 완화책** — 아래 셋은 이 가이드에 반영되어 있다.

| 완화책 | 위치 |
|---|---|
| 브로커를 Agent 대역과 중앙서버에서만 접근 가능하게 제한 | §1.1 방화벽 |
| Agent 별 계정 분리 — 하나가 새도 그 Agent 하나만 위험 | §5.1 |
| `central` 자격증명을 가장 엄격히 관리 (300대 전체 명령 권한) | §5.8 |
| 헬스체크 계정을 무력화 — 노출돼도 할 수 있는 게 없음 | §4.2, §6.2 |

나중에 TLS 를 켜는 절차는 **부록 A** 에 있다. 8883 리스너를 추가하고 전환 기간에 두 포트를 함께 열면 무중단으로 넘어갈 수 있다.

### 0.2 소요 시간

| 단계 | 시간 |
|---|---|
| §1 방화벽 신청 | **수일~수주** (병목) |
| §2~§6 설치 | ~1시간 |
| §7 검증 | ~1.5시간 (유휴 테스트 1시간 포함) |

---

### 0.3 설치 디렉터리 구조

모든 파일은 **`/sw/docker/mqtt`** 아래에 둔다. 이 문서의 경로는 전부 절대경로이며, 각 절 머리에 **실행 위치**를 명시한다.

> `/sw/docker` 를 다른 컨테이너와 공유하더라도 이 가이드가 건드리는 것은 **`mqtt` 하위뿐**이다. 소유자 변경(§2.2)도 `/sw/docker/mqtt` 에만 적용한다.

```
/sw/docker/mqtt/             ← 설치 루트
├── docker-compose.yml          §6.1   docker:docker  644
├── .env                        §6.2   docker:docker  600   ← health 비밀번호, UID/GID
├── config/                     §2.1   docker:docker  700
│   ├── mosquitto.conf          §4.1   docker:docker  600
│   ├── acl                     §4.2   docker:docker  600
│   └── passwd                  §5.7   docker:docker  600   ← 해시 목록
├── data/                       §2.1   docker:docker  700   ← 세션·오프라인 큐. 백업 대상
│   └── mosquitto.db                   컨테이너가 생성한다
└── log/                        §2.1   docker:docker  700
```

설치 중에만 존재하는 작업 디렉터리 — **메모리 파일시스템이며 §5.9 에서 파기한다**:

```
/dev/shm/mqtt-prov/             §5.2   docker:docker  700
├── inventory.csv                      호스트/계정 목록
├── gen-accounts.sh                    계정 생성 스크립트
├── passwd                             생성 결과(해시) → /sw/docker/mqtt/config/ 로 이동
└── creds.csv                          ★ 평문 비밀번호. 배포 후 shred
```

컨테이너 안에서의 대응 경로(§6.1 볼륨 마운트):

| 호스트 | 컨테이너 | 모드 |
|---|---|---|
| `/sw/docker/mqtt/config` | `/mosquitto/config` | **읽기 전용** |
| `/sw/docker/mqtt/data` | `/mosquitto/data` | 쓰기 |
| `/sw/docker/mqtt/log` | `/mosquitto/log` | 쓰기 |

> 로그에 나오는 `/mosquitto/config/...` 는 **컨테이너 안 경로**다. 호스트에서 고칠 때는 `/sw/docker/mqtt/config/...` 로 바꿔 읽는다.

#### 실행 사용자 — root 도 uid 1883 도 쓰지 않는다

**이 가이드는 `sudo` 를 한 번도 쓰지 않는다.** 설치·운영 전 과정을 `docker` 사용자 권한만으로 수행한다.

mosquitto 이미지는 기본적으로 컨테이너 안의 `mosquitto`(uid 1883)로 동작하지만, 그러면 호스트 파일 소유자도 1883 으로 맞춰야 하고 그 `chown` 에 root 가 필요하다. 대신 **컨테이너를 `docker` 사용자의 uid 로 실행한다**(§6.1 `user:`).

```bash
id -u    # 예: 1001  ← 이 값을 §6.2 .env 에 넣는다
id -g
```

그 결과:

- 모든 파일을 `docker` 사용자가 직접 소유한다 → `vi`, `sed` 로 **sudo 없이 수정**된다
- bind mount 는 uid 를 번역하지 않고 숫자만 비교하므로, 컨테이너 안 프로세스도 같은 uid 라 그대로 읽는다
- `chmod 600` 이 그대로 유효해 mosquitto 의 권한 경고가 뜨지 않는다 (실측 확인 — 기동 로그에 경고 없음)

> `docker` 사용자가 `docker` 그룹에 속해 있어야 한다. `docker ps` 가 `permission denied` 없이 실행되면 된다.

---


### 0.4 실행 위치 표기 — 호스트 작업과 컨테이너 작업 구분

**결론부터: 이 가이드의 작업은 거의 전부 리눅스 호스트에서 한다.** 돌고 있는 컨테이너 안에서 실행하는 명령은 접속 수 조회(§8.5, §9) 하나뿐이다.

각 절 머리에 아래 세 가지 중 하나를 표시한다.

| 표기 | 의미 | 프롬프트가 있는 곳 |
|---|---|---|
| **[호스트]** | 리눅스 호스트 셸에서 직접 실행 | 브로커 호스트 |
| **[호스트 → 일회용 컨테이너]** | 호스트 파일을 다루되, **도구를 이미지에서 빌려 쓴다** | 브로커 호스트 |
| **[컨테이너 내부]** | 돌고 있는 브로커 컨테이너 안에서 실행 | 컨테이너 |

#### 가장 헷갈리는 것 — "일회용 컨테이너"는 컨테이너 작업이 아니다

계정 생성(§5)과 갱신(§8.1)에 이런 명령이 나온다.

```bash
docker run --rm --user "$(id -u):$(id -g)" -v /sw/docker/mqtt/config:/w <이미지> \
  mosquitto_passwd -b /w/passwd 'myhost01_wasadm_J' '<pw>'
```

`docker run` 이 보이니 컨테이너 작업처럼 읽히지만, **실제로 바뀌는 것은 호스트의 `/sw/docker/mqtt/config/passwd` 파일**이다. 컨테이너는 `mosquitto_passwd` 라는 바이너리를 꺼내 쓰기 위해 1초 떴다가 사라진다(`--rm`). 호스트에 mosquitto 를 설치하지 않으려는 것뿐이다.

- 돌고 있는 브로커는 **건드리지 않는다.** `mosquitto_passwd` 는 브로커 데몬과 별개의 CLI 이므로 브로커가 꺼져 있어도 동작한다.
- `-v /sw/docker/mqtt/config:/w` 가 호스트 디렉터리를 컨테이너의 `/w` 로 연결한다. 그래서 명령 안의 경로는 `/w/passwd` 지만 **결과는 호스트 `/sw/docker/mqtt/config/passwd`** 에 남는다.

#### 경로 읽는 법

같은 파일이 호스트와 컨테이너에서 다른 경로로 불린다. **로그는 컨테이너 경로로 찍힌다.**

| 로그·설정에 보이는 경로 | 호스트에서 고칠 경로 |
|---|---|
| `/mosquitto/config/mosquitto.conf` | `/sw/docker/mqtt/config/mosquitto.conf` |
| `/mosquitto/config/passwd` | `/sw/docker/mqtt/config/passwd` |
| `/mosquitto/config/acl` | `/sw/docker/mqtt/config/acl` |
| `/mosquitto/data/mosquitto.db` | `/sw/docker/mqtt/data/mosquitto.db` |

`mosquitto.conf` 안의 `password_file /mosquitto/config/passwd` 도 **컨테이너 기준 경로**다. 호스트 경로로 바꿔 쓰면 브로커가 파일을 찾지 못한다.

#### 장별 실행 위치 요약

| 장 | 실행 위치 | 작업 디렉터리 |
|---|---|---|
| §1 방화벽 신청 | 실행 명령 없음 (결재 문서) | — |
| §2 호스트 준비 | **[호스트]** | `/sw/docker/mqtt`, `/etc/docker` |
| §3 이미지 반입 | **[호스트]** 인터넷 구간 + 폐쇄망 호스트 | 임의 (예: `~/`) |
| §4 설정 파일 | **[호스트]** | `/sw/docker/mqtt/config` |
| §5 계정 생성 | **[호스트]** + **[호스트 → 일회용 컨테이너]** | `/dev/shm/mqtt-prov` → `/sw/docker/mqtt/config` |
| §6 배포 | **[호스트]** | `/sw/docker/mqtt` |
| §7 설치 검증 | **[운영자 단말]·[Agent 호스트]** — 브로커 **밖**에서 | 임의 |
| §8 운영 절차 | **[호스트]** + **[호스트 → 일회용 컨테이너]** | `/sw/docker/mqtt` |
| §9 트러블슈팅 | **[호스트]** (접속 수 조회만 **[컨테이너 내부]**) | `/sw/docker/mqtt` |

---


## 1. 선행 작업

> **실행 위치: 없음.** 방화벽 신청서 작성·접수만 하는 장이다. 리드타임이 길어 가장 먼저 착수한다.

### 1.1 방화벽 신청

**포트를 8883 이 아닌 1883 으로** 신청한다. 출발지는 개별 IP 300개가 아니라 **대역(CIDR)**으로 낸다.

| # | 출발지 | 목적지 | 포트 | 용도 |
|---|---|---|---|---|
| 1 | 중앙서버 IP | 브로커 IP | TCP 1883 | 명령 발행 (상시 연결 1개) |
| 2 | Agent 대역 `10.x.y.0/24` | 브로커 IP | TCP 1883 | 명령 수신 (**상시 연결 300개**) |
| 3 | Agent 대역 | 중앙서버 IP | TCP 443 | 실행 결과 REST 전송 |
| 4 | 중앙서버 + Agent 대역 | 사내 DNS | TCP/UDP 53 | FQDN 해석 (`/etc/hosts` 시 생략) |
| 5 | 운영 단말 | 브로커 IP | TCP 1883 | CLI 점검. 운영자 대역만 |
| 6 | 배포 서버 | 사내 레지스트리 | TCP 443 | 이미지 pull (초기 1회) |

**평문이므로 출발지 제한이 유일한 접근 통제다.** 1·2·5번의 출발지를 넓게 잡지 않는다.

특기사항란 필수 문구:

```
- 클라이언트 → 브로커 방향 단방향 TCP 세션입니다. 역방향 정책 불필요.
- MQTT 특성상 상시 연결(persistent connection)이며 keepAlive 30초 간격
  2바이트 PINGREQ 가 발생합니다.
- 방화벽 TCP idle timeout 을 60초 이상으로 설정 요청드립니다. (권장 3600초)
```

> **idle timeout이 이 설치의 최대 사고 요인이다.** 값이 짧으면 양쪽 다 끊긴 줄 모르는 half-open 상태가 되어, 서버는 정상 publish + PUBACK을 받지만 Agent에는 영영 도달하지 않는다.

체크리스트:
- [ ] 6개 항목 접수, Agent 대역 CIDR 확정
- [ ] idle timeout ≥ 60초 확인 (권장 3600)
- [ ] NAT 경유 시 세션 테이블 여유 확인 — 상시 300세션이 **회수되지 않고** 점유된다
- [ ] IPS/DPI 애플리케이션 검사 여부 확인 — MQTT 시그니처가 없는 장비는 미상 프로토콜로 차단할 수 있다. **TLS 가 없으므로 DPI가 페이로드를 그대로 본다**

### 1.2 이미지 반입 경로

사내 레지스트리 등록이 가능한지, tar 반입인지 확인한다. 이미지는 **10.6 MB**라 tar 반입도 부담이 없다.

---

## 2. 호스트 준비

> **[호스트] 작업 디렉터리: `/sw/docker/mqtt`, `/etc/docker`**
> 이 시점에는 컨테이너가 **아직 존재하지 않는다**(기동은 §6). 들어갈 컨테이너가 없으므로 전부 호스트 작업이다.
> 나중에도 설정 디렉터리는 읽기 전용으로 마운트하므로, 권한은 **항상 호스트에서** 잡는다.

### 2.1 디렉터리 생성

```bash
mkdir -p /sw/docker/mqtt/{config,data,log}
```

### 2.2 소유자와 권한

```bash
chmod 700 /sw/docker/mqtt/{config,data,log}
```

확인:
```bash
ls -ld /sw/docker/mqtt /sw/docker/mqtt/{config,data,log}
id -u && id -g        # §6.2 에 넣을 값. 기록해 둔다
```

기대 출력 — 소유자가 **`docker`** 여야 한다:
```
drwxr-xr-x 5 docker docker 4096 ... /sw/docker/mqtt
drwx------ 2 docker docker 4096 ... /sw/docker/mqtt/config
drwx------ 2 docker docker 4096 ... /sw/docker/mqtt/data
drwx------ 2 docker docker 4096 ... /sw/docker/mqtt/log
```

`/sw/docker` 에 쓰기 권한이 없어 `mkdir` 이 실패하면 **인프라 담당에게 `docker` 사용자 소유의 `/sw/docker/mqtt` 생성을 요청**한다. 이 가이드에서 root 가 필요한 유일한 지점이다.

#### 왜 이 단계를 건너뛰면 안 되는가

mosquitto 2.0 은 `passwd`·`acl` 이 world-readable 이거나 소유자가 다르면 다음을 출력한다:

```
Warning: File /mosquitto/config/acl has world readable permissions.
         Future versions will refuse to load this file.
Warning: File /mosquitto/config/acl owner is not mosquitto.
         Future versions will refuse to load this file.
```

**"Future versions will refuse to load" 는 경고가 아니라 예고다.** 상위 버전으로 올리는 순간 브로커가 뜨지 않는다. 폐쇄망에서는 복구가 번거로우니 처음부터 맞춘다.

컨테이너 entrypoint 가 기동 시 `chown` 을 시도하지만, `:ro` 마운트라 실패한다:
```
chown: /mosquitto/config/passwd: Read-only file system
```
`:ro` 를 떼면 컨테이너가 고칠 수 있으나, **브로커가 자기 인증 설정을 덮어쓸 수 있게** 되므로 읽기 전용을 유지한다.

### 2.3 로그 로테이션

Docker 의 `json-file` 드라이버는 **기본적으로 로테이션을 하지 않는다.** 컨테이너를 지우기 전까지 `/var/lib/docker/containers/<id>/<id>-json.log` 가 무한정 커진다. 설정 여부는 이렇게 확인한다:

```bash
docker info --format '{{.LoggingDriver}}'
docker inspect <컨테이너> --format '{{.HostConfig.LogConfig.Config}}'   # map[] 이면 무제한
```

#### 무엇이 로그를 채우는가

**정상 운영 중에는 거의 늘지 않는다.** `connection_messages true` 는 접속/해제 시에만 기록하므로, 300대가 붙어서 유지되는 동안에는 추가 로그가 없다. 문제는 **Agent 가 재접속을 반복하는 상황**이다. 접속 1회당 브로커가 남기는 로그는 실측 기준 약 300~550 bytes 다:

```
New connection from 10.x.y.21:33200 on port 1883.
New client connected from 10.x.y.21:33200 as myhost01_wasadm_J (p2, c1, k30, u'myhost01_wasadm_J').
Client myhost01_wasadm_J disconnected.
```

| 상황 | 브로커 호스트 하루 로그량 |
|---|---|
| 정상 — 300대 접속 유지 | 최초 ~100 KB, 이후 거의 증가 없음 |
| **방화벽 idle timeout** — 45초마다 재접속(§7.8) | **~150 MB/일** |
| **clientId 충돌** — 초당 재접속 루프(§9) | **~7 GB/일** |
| **브로커 기동 실패 반복** — `restart: unless-stopped` 로 재시작 루프 | 기동 배너가 매회 누적 |

> ⚠️ **브로커가 꺼져 있는 동안에는 브로커 로그가 늘지 않는다.** 그때 재접속 실패 로그를 쏟아내는 것은 Agent 300대이고, 그것은 **각 Agent 호스트의 디스크**다. 이 절의 설정으로는 막을 수 없으며, Agent 측 로깅 정책으로 따로 다뤄야 한다.

위험한 쪽은 **브로커가 살아 있는데 Agent 가 붙었다 끊기를 반복하는** 경우다. 둘 다 발견이 늦는 장애라 주말을 끼면 그대로 쌓인다. 차는 곳이 `/var/lib/docker` 라 브로커뿐 아니라 **호스트의 다른 컨테이너까지 영향을 받는다.**

#### 이 가이드에서의 조치 — compose 에서 해결한다

데몬 기본값(`/etc/docker/daemon.json`)을 바꾸려면 root 와 `systemctl restart docker` 가 필요하다. **`docker` 사용자 권한으로는 할 수 없고, 필요하지도 않다.**

§6.1 compose 파일의 `logging:` 블록이 브로커 컨테이너를 최대 **250 MB** 로 묶는다:

```yaml
logging:
  driver: json-file
  options: { max-size: "50m", max-file: "5" }
```

컨테이너별 설정이 데몬 기본값보다 우선하므로 이것만으로 충분하다. **§2 에서 할 일은 없다.** 기동 후 §6.3 에서 적용 여부를 확인한다:

```bash
docker inspect mqtt-broker --format '{{.HostConfig.LogConfig.Config}}'
# → map[max-file:5 max-size:50m]     ← map[] 이면 미적용
```

> 호스트의 **다른 컨테이너**까지 보호하려면 데몬 설정이 필요하다. 이 호스트에 브로커 외 컨테이너가 있고 로그 관리가 안 되고 있다면 **인프라 담당에게 `daemon.json` 설정을 요청**한다. 브로커 설치의 선행 조건은 아니다.

---


## 3. 이미지 반입

> **[호스트] 작업 디렉터리: 임의 (예: `~/`)**
> 앞부분은 **인터넷 가능 구간**, 뒷부분은 **폐쇄망 브로커 호스트**에서 실행한다. 각 블록에 표시했다.


### 3.1 사내 레지스트리

**[호스트 — 인터넷 가능 구간]**

```bash
docker pull eclipse-mosquitto:2.0
docker tag  eclipse-mosquitto:2.0 registry.corp.local/mqtt/eclipse-mosquitto:2.0.22
docker push registry.corp.local/mqtt/eclipse-mosquitto:2.0.22
```

### 3.2 tar 반입

**[호스트 — 인터넷 가능 구간]**

```bash
docker pull eclipse-mosquitto:2.0
docker save eclipse-mosquitto:2.0 -o mosquitto-2.0.22.tar     # ~10 MB
sha256sum mosquitto-2.0.22.tar > mosquitto-2.0.22.sha256

```

**[호스트 — 폐쇄망 브로커 호스트]**

```bash
sha256sum -c mosquitto-2.0.22.sha256
docker load -i mosquitto-2.0.22.tar
```

### 3.3 검증

**[호스트 — 폐쇄망 브로커 호스트]**

```bash
docker run --rm --entrypoint mosquitto <이미지> -h | head -3
# → mosquitto is an MQTT v5.0/v3.1.1/v3.1 broker.
```

> **태그는 `2.0`이 아니라 `2.0.22`로 고정한다.** `2.0`은 이동 태그라 반입 시점에 따라 다른 바이너리가 들어올 수 있다.

---

## 4. 설정 파일

> **[호스트] 작업 디렉터리: `/sw/docker/mqtt/config`**
> 파일을 호스트에 만든다. 컨테이너는 아직 없다.
> 파일 안에 적는 `/mosquitto/config/...` 경로는 **컨테이너 기준 경로**다(§0.4). 호스트 경로로 바꿔 쓰면 브로커가 파일을 찾지 못한다.

만들 파일은 두 개다. `passwd` 는 §5 에서 생성한다.

```
/sw/docker/mqtt/config/mosquitto.conf     ← §4.1
/sw/docker/mqtt/config/acl                ← §4.2
/sw/docker/mqtt/config/passwd             ← §5.7 에서 설치
```

### 4.1 `mosquitto.conf` 생성

아래 블록을 **통째로** 붙여넣는다.

```bash
tee /sw/docker/mqtt/config/mosquitto.conf >/dev/null <<'CONF'
listener 1883
protocol mqtt

allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl

# 세션/메시지 영속화 — 컨테이너 재시작에도 오프라인 큐 유지
persistence true
persistence_location /mosquitto/data/
autosave_interval 30

# 오프라인 Agent 세션 보존 기간. 명령 만료(1h)보다 충분히 길게
persistent_client_expiration 7d

# 세션당 큐 상한 — 명령은 1h 에 자동 소멸하므로 1000건을 쌓을 이유가 없다
max_queued_messages 100
max_inflight_messages 20
memory_limit 512MB

log_dest stdout
log_type error
log_type warning
log_type notice
log_type information
connection_messages true
CONF
```

권한 적용:

```bash
chmod 600 /sw/docker/mqtt/config/mosquitto.conf
```

주요 값의 의미:

| 설정 | 의미 |
|---|---|
| `allow_anonymous false` | 익명 접속 차단. `password_file` 인증 강제 |
| `persistence true` | 컨테이너 재시작에도 세션·오프라인 큐 유지 |
| `persistent_client_expiration 7d` | 오프라인 Agent 세션 보존 기간. 명령 만료(1h)보다 길어야 한다 |
| `max_queued_messages 100` | 세션당 큐 상한. 명령이 1h 에 소멸하므로 충분하다 |
| `memory_limit 512MB` | 큐 폭주 시 브로커 자체를 보호 |
| `connection_messages true` | 접속/해제 로그. 장애 추적에 필요하다 |

> **MQTT 5 관련 설정은 없다.** mosquitto 2.0 은 3.1.1 과 5.0 을 같은 리스너에서 동시에 받고, `Message Expiry Interval` 처리는 기본 동작이다. 프로토콜 버전은 클라이언트가 CONNECT 시점에 결정한다(Python `protocol=mqtt.MQTTv5`, Java `org.eclipse.paho.mqttv5.client`).

### 4.2 `acl` 생성

```bash
tee /sw/docker/mqtt/config/acl >/dev/null <<'ACL'
# 컨트롤러: 명령 발행 전용. 읽기 권한 없음
user central
topic write cmd/#

# 운영자 — 브로커 지표 조회만
user ops
topic read  $SYS/#

# 헬스체크 전용 — 토픽 하나. 노출돼도 할 수 있는 게 없다
user health
topic read  $SYS/broker/uptime

# Agent 공통 패턴 — %u 는 접속 username 으로 치환
pattern read  cmd/%u/req
pattern read  cmd/broadcast/req
ACL
```

권한 적용:

```bash
chmod 600 /sw/docker/mqtt/config/acl
```

> ⚠️ **`tee` 로 만든 파일은 umask 에 따라 `644` 가 된다.** `chmod 600` 을 빼먹으면 §6 기동 시 `world readable permissions` 경고가 뜨고, mosquitto 상위 버전에서는 브로커가 아예 뜨지 않는다.
> 소유자는 `docker` 사용자 그대로이므로 `chown` 은 필요 없다.

이 ACL 이 강제하는 것:

- `central` 에 **읽기 권한이 없다** → 중앙서버는 발행만 할 수 있다. 실수로 구독 코드가 들어가도 브로커가 거부한다.
- Agent 계정에 **쓰기 권한이 없다** → Agent 는 구독만 할 수 있다. 실수로 발행 코드가 들어가도 거부된다.
- `pattern read cmd/%u/req` 의 `%u` 가 접속 username 으로 치환되므로, **`myhost01_wasadm_J` 는 `myhost02_wasadm_J` 의 명령 토픽을 구독할 수 없다.** §5 에서 계정명을 agentId 와 일치시켜야 하는 이유다.

기동 시 다음 경고가 뜨는데 **정상이다.** `pattern` 행에 치환 문자가 없어서 나며, **모든 인증 사용자에게 적용되는 규칙으로 정상 동작한다**(실측 확인):

```
Warning: ACL pattern 'cmd/broadcast/req' does not contain '%c' or '%u'.
```

### 4.3 배치 확인

```bash
ls -l /sw/docker/mqtt/config/
```

기대 출력 — **소유자 `docker`, 권한 `-rw-------`**:
```
-rw------- 1 docker docker  426 ... acl
-rw------- 1 docker docker  692 ... mosquitto.conf
```

`passwd` 는 아직 없다. §5 에서 만든다.

---


## 5. 계정 생성

> **[호스트] 작업 디렉터리: `/dev/shm/mqtt-prov` → 결과를 `/sw/docker/mqtt/config` 로 설치**
> §2 에서 디렉터리를 만든 **브로커 호스트**에서 한다. 이유는 두 가지다. ① 생성 결과 `passwd` 를 `/sw/docker/mqtt/config/` 에 바로 넣어야 한다. ② 평문 비밀번호 목록이 네트워크를 타지 않는다.
>
> 계정 생성 명령(§5.4~§5.5)은 **[호스트 → 일회용 컨테이너]** 다. `docker run` 이 보이지만 컨테이너에 들어가는 것이 아니라, 이미지에서 `mosquitto_passwd` 만 빌려 호스트 파일을 만드는 것이다(§0.4).

### 5.0 사전 확인

```bash
docker --version          # Docker 20.10 이상
openssl version           # 비밀번호 생성에 사용
id -u; id -g              # §6.2 에 넣을 UID/GID
docker ps >/dev/null && echo 'docker 그룹 OK'
ls -ld /sw/docker/mqtt/config   # §2 에서 만든 디렉터리
```

마지막 명령의 기대 출력:
```
drwx------ 2 docker docker 4096 ... /sw/docker/mqtt/config
```

`No such file or directory` 가 나오면 §2 를 먼저 수행한다.

### 5.1 무엇을 만드는가

총 **303개** 계정을 만든다.

| 계정 | 개수 | ACL 권한 | 받는 쪽 |
|---|---|---|---|
| `central` | 1 | `topic write cmd/#` — 300대 전체 명령 권한 | Python 중앙서버 |
| `health` | 1 | `$SYS/broker/uptime` 읽기만 | 브로커 호스트 `.env` |
| `ops` | 1 | `$SYS/#` 읽기 | 운영자 단말 |
| Agent | 300 | 자기 토픽만 (`%u` 치환) | 각 Agent 호스트 |

**Agent 계정명은 반드시 agentId 와 같아야 한다.** §5.2 ACL 이 `%u`(접속 username) 치환에 의존하기 때문이다. agentId 는 Agent 가 스스로 만들지 않고, 기동 스크립트가 채운 환경변수에서 다음 규칙으로 조립된다:

```
agentId = ${HOSTNAME}_${USER}_J
```

호스트가 `myhost01`, 구동 계정이 `wasadm` 이면 → **`myhost01_wasadm_J`**

> ⚠️ `agent-001` 같은 **일련번호 형식으로 만들면 안 된다.** Agent 는 `myhost01_wasadm_J` 로 접속하므로 전원 인증에 실패한다. 설계 문서나 예제에서 `agent-001` 표기를 봤다면 그것은 가독성을 위한 약칭이다.

계정을 공유해서도 안 된다. `pattern read cmd/%u/req` 가 모두 같은 토픽으로 풀려 아무 Agent 나 남의 명령을 구독할 수 있다. 평문 구간(§0.1)에서는 이 분리가 피해 범위를 그 Agent 하나로 묶어주는 유일한 장치다.

### 5.2 작업 디렉터리 만들기

평문 비밀번호를 다루므로 **디스크에 쓰지 않는다.** `/dev/shm` 은 메모리 파일시스템이라 재부팅 시 사라진다.

```bash
export WORK=/dev/shm/mqtt-prov
mkdir -p "$WORK" && chmod 700 "$WORK" && cd "$WORK"
pwd
```

기대 출력:
```
/dev/shm/mqtt-prov
```

> 이후 명령은 **전부 이 디렉터리에서** 실행한다. 중간에 터미널을 닫았다면 `export WORK=/dev/shm/mqtt-prov && cd "$WORK"` 로 다시 들어온다.

### 5.3 인벤토리 파일 작성

Agent 300대의 **(호스트명, 구동 계정)** 쌍이 필요하다. 이 두 값이 agentId 를 결정한다.

**방법 A — 직접 작성**

```bash
cat > inventory.csv <<'CSV'
# hostname,user
myhost01,wasadm
myhost02,wasadm
myhost03,appadm
CSV
```

**방법 B — 기존 자산 목록에서 변환**

호스트명 목록만 있고 계정이 전부 동일하다면:

```bash
awk '{print $1",wasadm"}' hostlist.txt > inventory.csv
```

**방법 C — Agent 호스트에서 직접 수집** (SSH 가능한 경우)

```bash
while read -r h; do
  echo "$h,$(ssh "$h" 'whoami')"
done < hostlist.txt > inventory.csv
```

#### 검증 — 여기서 틀리면 전부 다시 만들어야 한다

```bash
grep -vc '^#' inventory.csv                                    # 300 이어야 한다
awk -F, '!/^#/{print $1"_"$2"_J"}' inventory.csv | sort | uniq -d   # 출력이 없어야 한다
```

두 번째 명령은 **agentId 중복 검사**다. 출력이 있으면 같은 호스트에 같은 계정이 두 번 들어간 것이다. 그대로 두면 두 Agent 가 같은 clientId 로 접속해 서로를 강제 종료시키며 무한 재접속 루프에 빠진다. agentId 는 clientId 로도 쓰이며, mosquitto 는 clientId 가 겹치면 뒤에 들어온 쪽이 앞의 세션을 강제 종료(session takeover)시킨다.

### 5.4 생성 스크립트 저장

한 줄씩 복사하지 말고 **파일로 저장해 실행**한다. 아래 블록을 통째로 붙여넣는다.

```bash
cat > gen-accounts.sh <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

WORK="$(cd "$(dirname "$0")" && pwd)"
INV="$WORK/inventory.csv"
: "${IMG:?IMG 환경변수에 이미지 태그를 지정하세요}"

[ -f "$INV" ] || { echo "인벤토리 파일이 없습니다: $INV" >&2; exit 1; }

: > "$WORK/passwd";    chmod 600 "$WORK/passwd"
: > "$WORK/creds.csv"; chmod 600 "$WORK/creds.csv"

add() {
  docker run --rm --user "$(id -u):$(id -g)" -v "$WORK:/w" "$IMG" \
    mosquitto_passwd -b /w/passwd "$1" "$2"
  printf '%s,%s\n' "$1" "$2" >> "$WORK/creds.csv"
}

echo "[1/2] 운영 계정 3종 생성"
for u in central health ops; do add "$u" "$(openssl rand -base64 24)"; done

echo "[2/2] Agent 계정 생성"
n=0
while IFS=, read -r host user; do
  host="$(echo "$host" | tr -d ' \r')"; user="$(echo "$user" | tr -d ' \r')"
  [ -z "$host" ] && continue
  case "$host" in \#*) continue ;; esac
  add "${host}_${user}_J" "$(openssl rand -base64 24)"
  n=$((n+1))
done < "$INV"

echo
echo "완료: 운영 3 + Agent $n = $((n+3)) 계정"
SCRIPT
chmod +x gen-accounts.sh
```

스크립트가 하는 일:

- 비밀번호는 `openssl rand -base64 24` (~144 bit). 사람이 외울 일이 없으므로 길게 잡는다.
- `mosquitto_passwd` 는 **브로커 데몬과 별개의 CLI** 다. 브로커가 떠 있지 않아도 동작하므로 설치 전에 실행할 수 있다.
- `--user "$(id -u):$(id -g)"` 와 `chmod 600` 이 핵심이다. 빼면 호출마다 권한 경고 3줄이 나와 **303회에 900줄**이 쌓이고 실제 실패를 놓친다.
- 인벤토리의 `#` 주석 줄과 윈도우 개행(`\r`)을 걸러낸다.

### 5.5 실행

```bash
export IMG=registry.corp.local/mqtt/eclipse-mosquitto:2.0.22
./gen-accounts.sh
```

기대 출력 — **이 네 줄 외에 아무것도 나오지 않아야 한다**:
```
[1/2] 운영 계정 3종 생성
[2/2] Agent 계정 생성

완료: 운영 3 + Agent 300 = 303 계정
```

| 실패 | 원인 | 조치 |
|---|---|---|
| `IMG 환경변수에...` | `export IMG=` 누락 | §5.5 첫 줄 |
| `Unable to find image` | 이미지 미반입 | §3 |
| `permission denied ... docker.sock` | `docker` 그룹 미포함 | 인프라 담당에 `usermod -aG docker docker` 요청 후 재로그인 |
| `Warning: File /w/passwd ...` 가 반복 | 스크립트를 수정해 실행 | 원문 그대로 사용 |

### 5.6 결과 확인

```bash
wc -l < passwd                   # 303
cut -d: -f1 passwd | head -5     # 계정명 확인
cut -d: -f1 passwd | tail -3     # Agent 계정명 형식 확인
head -c 40 passwd; echo          # 해시 형식 확인
```

기대 출력:
```
303
central
health
ops
myhost01_wasadm_J
myhost02_wasadm_J
...
myhost300_wasadm_J
central:$7$101$........
```

`$7$` 로 시작하면 정상이다. **평문은 `passwd` 에 들어가지 않는다** — 브로커는 평문을 가진 적이 없다.

### 5.7 브로커에 설치

```bash
install -m 600 passwd /sw/docker/mqtt/config/passwd
ls -l /sw/docker/mqtt/config/passwd
```

기대 출력:
```
-rw------- 1 docker docker 22134 ... /sw/docker/mqtt/config/passwd
```

### 5.8 자격증명 배포

평문 비밀번호는 `creds.csv` 에 `계정명,비밀번호` 형식으로 들어 있다. 해시(`passwd`)와 평문(`creds.csv`)은 **서로 다른 경로로** 나간다.

#### (1) `health` → 브로커 호스트 `.env`

```bash
HPW=$(grep '^health,' creds.csv | cut -d, -f2-)
printf 'MQTT_HEALTH_PW=%s\n' "$HPW" >> /sw/docker/mqtt/.env
chmod 600 /sw/docker/mqtt/.env
ls -l /sw/docker/mqtt/.env
```

#### (2) `central` → 중앙서버

```bash
grep '^central,' creds.csv | cut -d, -f2-        # 값 확인 후 중앙서버로 옮긴다
```

중앙서버에서:
```bash
printf 'MQTT_CENTRAL_PW=%s\n' '<위 값>' >> /opt/controller/.env
chmod 600 /opt/controller/.env
```

`central` 은 계정 1개지만 **가장 강력하다.** `topic write cmd/#` 는 300대 전체에 임의 명령을 보낼 수 있다. Agent 자격증명 하나가 새면 그 Agent 가 위험하지만, `central` 이 새면 전체가 위험하다.

#### (3) Agent 300대 → `agent.properties`

Agent 는 자기 계정 하나만 받는다. 파일 형식:

```properties
mqtt.host=10.x.y.10
mqtt.port=1883
mqtt.username=myhost01_wasadm_J
mqtt.password=<해당 Agent 비밀번호>
```

`username` 은 Agent 가 `${HOSTNAME}_${USER}_J` 로 조립한 값과 같아야 한다. 배포 예시:

```bash
while IFS=, read -r aid pw; do
  case "$aid" in central|health|ops) continue ;; esac
  host="${aid%%_*}"
  ssh "$host" "umask 077 && printf 'mqtt.username=%s\nmqtt.password=%s\n' '$aid' '$pw' \
                 >> /opt/agent/agent.properties && chmod 600 /opt/agent/agent.properties"
done < creds.csv
```

> 폐쇄망 정책상 SSH 일괄 배포가 불가능하면 배포 파이프라인이나 구성관리 도구를 쓴다. **어느 경우든 파일 권한 600, 서비스 계정 소유**가 조건이다.

배포 시 금지 사항:

- **환경변수 주입 금지.** `/proc/<pid>/environ`, `ps e`, 코어덤프, 자식 프로세스로 전파된다.
- **이미지·jar 하드코딩 금지.** 300대에 같은 값이 박히고 회수가 불가능하다.
- Java 쪽에서 `MqttConnectionOptions` 를 통째로 로그에 찍으면 자격증명이 샌다. `reasonString` 만 찍는다.

### 5.9 평문 목록 파기

**배포가 끝나면 즉시 수행한다.**

```bash
cd "$WORK" && shred -u creds.csv inventory.csv
cd / && rm -rf "$WORK"
ls /dev/shm/mqtt-prov      # No such file or directory 여야 한다
```

`passwd`(해시)는 `/sw/docker/mqtt/config/` 에 설치되어 있으므로 작업 디렉터리를 통째로 지워도 된다.

> 비밀번호를 분실하면 복구할 수 없다. 해당 계정을 §8.1 절차로 재발급한다.

---


## 6. 배포

> **[호스트] 작업 디렉터리: `/sw/docker/mqtt`**

### 6.1 `/sw/docker/mqtt/docker-compose.yml`

```yaml
services:
  broker:
    image: registry.corp.local/mqtt/eclipse-mosquitto:2.0.22
    container_name: mqtt-broker
    user: "${MQTT_UID}:${MQTT_GID}"   # ★ docker 사용자로 실행. root/1883 을 쓰지 않는다
    restart: unless-stopped
    ports:
      - "1883:1883"
    volumes:
      - /sw/docker/mqtt/config:/mosquitto/config:ro
      - /sw/docker/mqtt/data:/mosquitto/data
      - /sw/docker/mqtt/log:/mosquitto/log
    healthcheck:
      # central 은 읽기 권한이 없다(§4.2). health 전용 계정을 쓴다
      test: ["CMD", "mosquitto_sub", "-h", "localhost", "-p", "1883",
             "-u", "health", "-P", "${MQTT_HEALTH_PW}",
             "-t", "$$SYS/broker/uptime", "-C", "1", "-W", "3"]
      interval: 30s
      timeout: 5s
      retries: 3
    stop_grace_period: 30s
    logging:
      driver: json-file
      options: { max-size: "50m", max-file: "5" }
    deploy:
      resources:
        limits: { cpus: "2", memory: 2G }
```

Agent 300대는 Mosquitto 단일 노드 용량(~10k 커넥션)의 **3% 수준**이다. 1 vCPU / 1 GB 로 충분하며 위 값은 여유분이다.

### 6.2 `.env` (권한 600)

`health` 비밀번호는 §5.8 에서 이미 넣었다. 여기에 **실행 UID/GID** 를 추가한다.

```bash
cd /sw/docker/mqtt
printf 'MQTT_UID=%s\nMQTT_GID=%s\n' "$(id -u)" "$(id -g)" >> .env
chmod 600 .env
cat .env
```

기대 출력:
```
MQTT_HEALTH_PW=...
MQTT_UID=1001
MQTT_GID=1001
```

compose 가 같은 디렉터리의 `.env` 를 자동으로 읽어 `user:` 와 healthcheck 에 채운다.

> ⚠️ **`MQTT_UID`/`MQTT_GID` 가 비어 있으면 컨테이너가 root 로 뜬다.** 그러면 `data/` 에 root 소유 파일이 생겨 이후 `docker` 사용자가 백업·삭제할 수 없게 된다. 기동 후 §6.3 에서 반드시 확인한다.

> 이 값은 compose 파싱 시점에 컨테이너 설정에 박혀 **`docker inspect`로 평문 노출된다.** 숨기려 애쓰는 대신 **노출돼도 아무것도 못 하는 계정**을 쓴다 — `health` 는 `$SYS/broker/uptime` 읽기 권한 하나뿐이다(§4.2 ACL).

### 6.3 기동

```bash
cd /sw/docker/mqtt && docker compose up -d
docker compose ps                 # STATUS: Up (healthy)
docker compose logs --tail 30
```

기대 로그:
```
mosquitto version 2.0.22 starting
Config loaded from /mosquitto/config/mosquitto.conf.
Warning: ACL pattern 'cmd/broadcast/req' does not contain '%c' or '%u'.   ← 정상
Opening ipv4 listen socket on port 1883.
Opening ipv6 listen socket on port 1883.
mosquitto version 2.0.22 running
```

`Warning: ... world readable permissions` 가 보이면 §4 의 `chmod 600` 을 확인한다.

#### 실행 사용자와 로그 설정 확인

```bash
docker inspect mqtt-broker --format 'User={{.Config.User}}'
docker inspect mqtt-broker --format 'Log={{.HostConfig.LogConfig.Config}}'
ls -l /sw/docker/mqtt/data/
```

기대 출력:
```
User=1001:1001                      ← 비어 있으면 root 로 뜬 것이다 (§6.2)
Log=map[max-file:5 max-size:50m]    ← map[] 이면 미적용 (§2.3)
```

`data/` 는 기동 직후에는 비어 있는 것이 정상이다. `mosquitto.db` 는 `autosave_interval`(30초) 경과 또는 정지 시점에 기록된다. 생성 후 소유자를 확인한다:

```bash
ls -l /sw/docker/mqtt/data/
# -rw------- 1 docker docker ... mosquitto.db
```

`root` 소유로 생겼다면 `.env` 의 UID/GID 가 비어 있었던 것이다. 컨테이너를 내리고(`docker compose down`) **root 소유 파일 삭제를 인프라 담당에 요청한 뒤** §6.2 부터 다시 한다.

**설정 파일은 compose에 파일명으로 나타나지 않는다.** 디렉터리를 통째로 마운트하고(`/sw/docker/mqtt/config:/mosquitto/config:ro`), 이미지 기본 CMD가 `mosquitto -c /mosquitto/config/mosquitto.conf`이기 때문에 경로가 맞물려 적용된다. `Config loaded from ...` 로그로 확인한다.

---

## 7. 설치 검증

> **실행 위치: 브로커 호스트가 아니라 [운영자 단말] 또는 [Agent 호스트].**
> 방화벽·네트워크 경로까지 함께 검증해야 한다. 브로커 안에서 `localhost` 로 테스트하면 경로 검증이 빠져 의미가 없다.

**순서대로** 진행한다. 앞 단계가 통과해야 다음 단계의 결과를 신뢰할 수 있다.

### 7.0 검증 도구 준비

`mosquitto_sub`/`mosquitto_pub` 가 필요하다. 둘 중 하나를 쓴다.

**방법 A — 이미지에서 빌려 쓰기** (권장. 패키지 설치 권한이 필요 없다)
```bash
mqtt() { docker run --rm --network host <이미지> "$@"; }
mqtt mosquitto_sub --help | head -1
```

**방법 B — 패키지 설치** (root 권한이 있는 운영자 단말에서만)
```bash
yum install -y mosquitto           # 설치 권한 필요
```

이하 명령은 `mosquitto_sub ...` 로 적는다. 방법 A 를 쓴다면 앞에 `mqtt ` 를 붙인다.

#### 검증 전 변수 설정

```bash
export BROKER=10.x.y.10          # 브로커 IP 또는 FQDN
export A1=myhost01_wasadm_J      # 검증용 Agent 1
export A2=myhost02_wasadm_J      # 검증용 Agent 2
```

`central`·`ops`·Agent 비밀번호는 §5.8 에서 배포한 값을 쓴다.

---

### 7.1 네트워크 경로

애플리케이션 없이 TCP 도달만 먼저 본다.

```bash
nc -zv "$BROKER" 1883
```

기대 출력:
```
Connection to 10.x.y.10 1883 port [tcp/*] succeeded!
```

| 실패 | 원인 | 조치 |
|---|---|---|
| `Connection refused` | 브로커 미기동 또는 포트 미개방 | §6.3 `docker compose ps` 확인 |
| `Connection timed out` | 방화벽 미개방 | §1.1 신청 상태 확인 |
| `Name or service not known` | DNS 미해석 | IP 로 시도, `/etc/hosts` 고정 검토 |

**여기서 막히면 이후가 전부 막힌다.** 반드시 먼저 해결한다.

### 7.2 인증

```bash
# ① 정상 계정
mosquitto_sub -h "$BROKER" -p 1883 -u ops -P '<ops 비밀번호>' \
  -t '$SYS/broker/version' -C 1
```
기대 출력:
```
mosquitto version 2.0.22
```

```bash
# ② 익명 접속 — 거부되어야 한다
mosquitto_sub -h "$BROKER" -p 1883 -t '$SYS/#' -C 1
```
기대 출력:
```
Connection error: Connection Refused: not authorised.
```

```bash
# ③ 틀린 비밀번호 — 거부되어야 한다
mosquitto_sub -h "$BROKER" -p 1883 -u ops -P 'wrong' -t '$SYS/#' -C 1
```
기대 출력:
```
Connection error: Connection Refused: not authorised.
```

②가 성공하면 `allow_anonymous false` 가 적용되지 않은 것이다. §4.1 을 확인하고 §6 을 다시 기동한다.

### 7.3 토픽 격리 — 가장 중요한 검증

`$A2` 계정으로 `$A1` 의 명령 토픽을 구독해도 **명령이 오지 않아야** 한다.

**터미널 1** — 남의 토픽 구독 시도:
```bash
mosquitto_sub -h "$BROKER" -p 1883 -u "$A2" -P '<A2 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -W 10
```

**터미널 2** — 자기 토픽 구독:
```bash
mosquitto_sub -h "$BROKER" -p 1883 -u "$A1" -P '<A1 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -W 10
```

**터미널 3** — `central` 로 실제 발행:
```bash
mosquitto_pub -h "$BROKER" -p 1883 -V 5 -u central -P '<central 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -m '{"cmdId":"iso-test"}'
```

기대 결과:

| 터미널 | 기대 출력 | 의미 |
|---|---|---|
| 1 (남의 토픽) | `Timed out` — **수신 0건** | 격리 성공 |
| 2 (자기 토픽) | `{"cmdId":"iso-test"}` | 정상 수신 |

> ⚠️ **SUBACK 으로 판정하지 말 것.** mosquitto 는 ACL 위반 구독도 **SUBACK 0(성공)으로 응답**하고 거부는 **메시지 전달 시점**에 적용한다. `-d` 옵션에 `Subscribed (mid: 1): 1` 이 찍히는 것은 정상이며 격리 실패가 아니다.
> 터미널 2 를 함께 보는 이유는, ACL 이 **과하게 막고 있지 않은지**도 동시에 확인하기 위해서다. 둘 다 0건이면 격리가 아니라 설정 오류다.

터미널 1 에 메시지가 오면 **계정명이 agentId 와 다르거나** ACL 의 `%u` 패턴이 잘못된 것이다. §5.1 과 §4.2 를 확인한다.

### 7.4 발행 전용 / 구독 전용 강제

```bash
# ① central 은 읽기 권한이 없어야 한다
mosquitto_sub -h "$BROKER" -p 1883 -u central -P '<central 비밀번호>' -t 'evt/#' -W 5
```
기대 출력 — **`Timed out`, 수신 0건**:
```
Timed out
```

```bash
# ② Agent 는 발행 권한이 없어야 한다 — -V 5 가 필수다
mosquitto_pub -h "$BROKER" -p 1883 -V 5 -u "$A1" -P '<A1 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -m 'x' -d | grep PUBACK
```
기대 출력:
```
Client ... received PUBACK (Mid: 1, RC:135)
```

`RC:135` 가 **Not authorized** 다. 거부되었다는 뜻이며 이것이 정상이다.

> ⚠️ **`-V 5` 를 빼면 검증이 무효가 된다.** MQTT 3.1.1 에는 PUBACK 에 사유 코드가 없어, 거부된 발행도 다음과 같이 **성공처럼 보인다**:
> ```
> Client null received PUBACK (Mid: 1, RC:0)     ← 종료코드도 0
> ```
> 메시지는 실제로 버려지지만 발행자는 알 수 없다. 실측으로 확인된 함정이다.

### 7.5 오프라인 큐잉

Agent 가 꺼져 있을 때 발행한 명령이 재접속 시 전달되는지 본다.

```bash
# ① 세션 생성 — 접속했다가 끊는다
mosquitto_sub -h "$BROKER" -p 1883 -V 5 -u "$A1" -P '<A1 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -c -i "$A1" -x 300 -W 2

# ② 오프라인 상태에서 발행
mosquitto_pub -h "$BROKER" -p 1883 -V 5 -u central -P '<central 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -m '{"cmdId":"queued-1"}'

# ③ 재접속
mosquitto_sub -h "$BROKER" -p 1883 -V 5 -u "$A1" -P '<A1 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -c -i "$A1" -x 300 -W 5
```

기대 출력 (③):
```
{"cmdId":"queued-1"}
```

옵션의 의미 — **하나라도 빠지면 큐잉이 일어나지 않는다**:

| 옵션 | 의미 |
|---|---|
| `-q 1` | **구독 QoS.** QoS 0 구독은 오프라인 큐잉 대상이 아니다 |
| `-c` | clean session 해제. 끊어도 세션을 유지한다 |
| `-i "$A1"` | clientId 고정. 같은 세션으로 돌아오기 위해 필요하다 |
| `-x 300` | 세션 만료 300초. MQTT 5 에서 지정하지 않으면 기본 0(즉시 소멸)이다 |

> ⚠️ **`-q 1` 누락이 가장 흔한 실수다.** 발행을 QoS 1 로 해도 **구독이 QoS 0 이면 유효 QoS 는 0** 이라 큐에 쌓이지 않는다. ③에 아무것도 오지 않으면서 오류도 없다면 이것을 먼저 의심한다.

### 7.6 메시지 만료

명령이 1시간 뒤 자동 소멸하는지 확인한다. 실제로 1시간 기다릴 필요는 없다.

```bash
# ① 세션 생성
mosquitto_sub -h "$BROKER" -p 1883 -V 5 -u "$A1" -P '<A1 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -c -i "$A1" -x 300 -W 2

# ② 오프라인 상태에서 두 건 발행 — 만료 5초 / 300초
mosquitto_pub -h "$BROKER" -p 1883 -V 5 -u central -P '<central 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -D PUBLISH message-expiry-interval 5   -m '{"cmdId":"exp-A"}'
mosquitto_pub -h "$BROKER" -p 1883 -V 5 -u central -P '<central 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -D PUBLISH message-expiry-interval 300 -m '{"cmdId":"keep-B"}'

# ③ 8초 경과 후 재접속
sleep 8
mosquitto_sub -h "$BROKER" -p 1883 -V 5 -u "$A1" -P '<A1 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -c -i "$A1" -x 300 -W 5
```

기대 출력 (③) — **`keep-B` 만 오고 `exp-A` 는 오지 않는다**:
```
{"cmdId":"keep-B"}
Timed out
```

> `mosquitto_pub` 의 `-x` 는 **session-expiry-interval 이지 메시지 만료가 아니다.**
> 메시지 만료는 `-D PUBLISH message-expiry-interval <초>` 다. 실측으로 확인된 함정이다.

**세션 만료가 메시지 만료보다 짧으면 안 된다.** `-x` 를 10 으로 낮추면 세션 자체가 먼저 사라져 큐가 통째로 없어진다. 운영값은 세션 24시간 / 메시지 1시간이다.

### 7.7 브로커 재시작 내구성

**[호스트]** 에서:
```bash
cd /sw/docker/mqtt && docker compose restart
```

재시작 전에 §7.5 ②까지 수행해 큐에 메시지를 남겨두고, 재시작 후 ③으로 수신되는지 본다. `persistence true` 가 동작하면 큐가 살아남는다.

확인:
```bash
ls -l /sw/docker/mqtt/data/mosquitto.db
```
파일이 존재하고 크기가 0 이 아니어야 한다.

### 7.8 장시간 유휴 — 폐쇄망 필수 항목

**이 항목을 건너뛰면 운영 중에 발견하게 된다.**

```bash
# Agent 1대를 접속시킨 채 1시간 이상 방치한 뒤, 명령을 발행해 도달하는지 확인
mosquitto_sub -h "$BROKER" -p 1883 -V 5 -u "$A1" -P '<A1 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -c -i "$A1" -x 86400 -W 4000
```

1시간 이상 지난 뒤 다른 터미널에서:
```bash
mosquitto_pub -h "$BROKER" -p 1883 -V 5 -u central -P '<central 비밀번호>' \
  -t "cmd/$A1/req" -q 1 -m '{"cmdId":"idle-test"}'
```

구독 쪽에 `{"cmdId":"idle-test"}` 가 떠야 한다. 오지 않으면 방화벽·NAT 가 유휴 세션을 RST 없이 버린 것이다(half-open). 브로커는 정상 publish 하고 PUBACK 도 받지만 Agent 에는 도달하지 않는다. §1.1 의 idle timeout 값을 상향 요청한다.

### 7.9 체크리스트

- [ ] 7.1 `nc -zv` → `succeeded!`
- [ ] 7.2 익명·오류 비밀번호 → `not authorised`
- [ ] 7.3 토픽 격리 — 남의 토픽 0건 / 자기 토픽 수신
- [ ] 7.4 `central` 구독 0건 / Agent 발행 `RC:135`
- [ ] 7.5 오프라인 큐잉 — 재접속 시 수신
- [ ] 7.6 메시지 만료 — 경과분 폐기, 이내분 전달
- [ ] 7.7 브로커 재시작 후에도 큐 유지
- [ ] 7.8 **1시간 유휴 후 명령 도달**
- [ ] `docker compose ps` → `Up (healthy)`

---

## 8. 운영 절차

> **[호스트] 작업 디렉터리: `/sw/docker/mqtt`** — 계정 조작은 **[호스트 → 일회용 컨테이너]**(§0.4)

### 8.1 비밀번호 갱신 — 무중단

mosquitto 는 **SIGHUP 으로 `password_file` 을 다시 읽으며 기존 연결을 끊지 않는다.** 인증은 CONNECT 시점에만 일어나기 때문이다. 300대가 붙어 있는 채로 갱신할 수 있다.

**① 브로커 쪽 갱신**

```bash
docker run --rm --user "$(id -u):$(id -g)" -v /sw/docker/mqtt/config:/w \
  registry.corp.local/mqtt/eclipse-mosquitto:2.0.22 \
  mosquitto_passwd -b /w/passwd '<agentId>' '<새 비밀번호>'
```

출력이 없으면 성공이다. `-v` 로 마운트한 호스트의 `/sw/docker/mqtt/config/passwd` 가 바뀐다.

**② 브로커에 재읽기 지시**

```bash
docker kill -s HUP mqtt-broker
```

**재시작이 아니다.** 컨테이너는 그대로이고 접속 중인 Agent 도 끊기지 않는다. 확인:

```bash
docker compose ps        # STATUS 가 Up (재시작되면 uptime 이 초기화된다)
```

**③ 검증**

```bash
# 새 비밀번호로 접속 — 성공해야 한다
mosquitto_sub -h <BROKER> -p 1883 -u '<agentId>' -P '<새 비밀번호>' \
  -t 'cmd/<agentId>/req' -C 1 -W 3

# 옛 비밀번호로 접속 — 거부되어야 한다
mosquitto_sub -h <BROKER> -p 1883 -u '<agentId>' -P '<옛 비밀번호>' \
  -t 'cmd/<agentId>/req' -C 1 -W 3
```

기대 출력:
```
(새 비밀번호) Timed out              ← 접속 성공. 명령이 없어 타임아웃
(옛 비밀번호) Connection error: Connection Refused: not authorised.
```

**④ Agent 쪽 갱신**

```bash
ssh <호스트> "sed -i 's|^mqtt.password=.*|mqtt.password=<새 비밀번호>|' /opt/agent/agent.properties"
ssh <호스트> "systemctl restart agent"
```

#### 순서를 지켜야 하는 이유

**브로커를 먼저 갱신한다.** 그래야 해당 Agent 는 접속 중에는 살아 있고, 재접속하는 순간부터 새 값을 요구받는다. Agent 를 먼저 바꾸면 그 사이 재접속이 발생했을 때 즉시 실패한다.

1. 브로커 갱신 + SIGHUP (①②)
2. 검증 (③)
3. Agent 설정 교체 + 재기동 (④)

2~3 사이에 Agent 가 끊기면 복구되지 않으므로 **300대를 한 번에 돌리지 않고 배치로 나눈다.**

> 만료 정책은 두지 않는다. 기계 간 인증이고 사람이 외우지 않으므로 주기적 변경의 이득이 없다. **유출 의심·담당자 변경·감사 요구 시에만** 갱신한다.

### 8.2 계정 추가 (Agent 증설)

```bash
docker run --rm --user "$(id -u):$(id -g)" -v /sw/docker/mqtt/config:/w \
  registry.corp.local/mqtt/eclipse-mosquitto:2.0.22 \
  mosquitto_passwd -b /w/passwd '<새 agentId>' '<비밀번호>'
docker kill -s HUP mqtt-broker
```

확인:
```bash
grep -c '' /sw/docker/mqtt/config/passwd             # 계정 수가 1 늘어야 한다
cut -d: -f1 /sw/docker/mqtt/config/passwd | tail -1
```

**ACL 은 수정할 필요가 없다.** `pattern read cmd/%u/req` 가 계정명으로 자동 치환되므로, 계정만 추가하면 해당 Agent 의 토픽 권한이 바로 생긴다.

### 8.3 계정 삭제 (Agent 폐기)

```bash
docker run --rm --user "$(id -u):$(id -g)" -v /sw/docker/mqtt/config:/w \
  registry.corp.local/mqtt/eclipse-mosquitto:2.0.22 \
  mosquitto_passwd -D /w/passwd '<agentId>'
docker kill -s HUP mqtt-broker
```

> SIGHUP 은 **기존 연결을 끊지 않는다.** 삭제된 계정으로 접속 중인 Agent 는 **끊길 때까지 살아 있다.** 즉시 차단해야 한다면 해당 Agent 를 먼저 정지시킨다.

### 8.4 백업

```bash
cd /sw/docker/mqtt
docker compose stop
tar czf ~/mqtt-backup-$(date +%F).tgz -C /sw/docker/mqtt config data
docker compose start
docker compose ps        # Up (healthy) 확인
```

`data/mosquitto.db` 는 **세션과 오프라인 큐**다. 유실되면 복구 후 발행된 명령이 **세션이 없어 조용히 폐기된다.** 오류도 로그도 남지 않으므로 백업 대상에서 빼지 않는다.

> `docker compose stop` 중에는 300대가 재접속을 시도한다. 백업은 짧게 끝내고, 업무 시간 외에 수행한다.

#### 복원

```bash
cd /sw/docker/mqtt
docker compose down
tar xzf ~/mqtt-backup-<날짜>.tgz -C /sw/docker/mqtt
docker compose up -d
```

`tar` 가 권한 비트(600)를 보존하므로 별도 `chmod` 는 필요 없다. 다만 복원 후 `ls -l` 로 한 번 확인한다.

> 백업 파일은 **평문 비밀번호는 없지만 해시와 ACL 을 포함**한다. `~/` 대신 별도 보관소로 옮긴다면 접근 권한을 확인한다.

### 8.5 브로커 재시작 시 주의

패치·설정 변경으로 재시작하면 **300대가 동시에 재접속한다.**

```bash
cd /sw/docker/mqtt && docker compose restart
```

- Agent 에 **지수 백오프 + `random(0, 5s)` jitter** 가 들어 있는지 확인한다. Paho 의 `setAutomaticReconnect` 는 백오프는 하지만 **jitter 가 없어 위상이 겹친다.**
- 평문이라 TLS 핸드셰이크 부하가 없어 재접속 폭주 자체는 수백 ms 내 완료된다. 다만 재접속이 실패 반복으로 이어지면 로그가 폭증하므로 §2.3 로테이션을 확인한다.

재시작 후 접속 수 확인:
```bash
docker exec mqtt-broker mosquitto_sub -h localhost -p 1883 \
  -u ops -P '<ops 비밀번호>' -t '$SYS/broker/clients/connected' -C 1
```

### 8.6 일상 점검

```bash
cd /sw/docker/mqtt
docker compose ps                              # Up (healthy)
docker compose logs --since 24h | grep -ci error
df -h /var/lib/docker                          # 로그 누적 확인
ls -lh /sw/docker/mqtt/data/mosquitto.db             # 큐 크기 추이
```

`mosquitto.db` 가 계속 커지면 오프라인 Agent 가 쌓이고 있다는 뜻이다. 접속 수(§8.5)와 대조한다.

---


### 8.7 설정 파일 수정

`mosquitto.conf`·`acl` 은 **`docker` 사용자 소유**에 권한 `600` 이다. 본인 파일이므로 **`sudo` 없이 그대로 편집한다.**

```bash
cd /sw/docker/mqtt/config
vi mosquitto.conf
```

다른 방법:

```bash
# 한 줄 치환
sed -i 's/^max_queued_messages .*/max_queued_messages 200/' mosquitto.conf

# 전체 교체 (§4.1 형식)
tee mosquitto.conf >/dev/null <<'CONF'
... 내용 ...
CONF
```

#### 수정 후 권한 확인

```bash
ls -l /sw/docker/mqtt/config/
```

기대: `-rw------- 1 docker docker`

소유자는 바뀌지 않지만 **권한 비트는 도구에 따라 달라진다.** `tee` 로 새로 만들면 umask 에 따라 `644` 가 되고, 일부 편집기도 저장 시 권한을 재설정한다. 어긋나 있으면:

```bash
chmod 600 /sw/docker/mqtt/config/mosquitto.conf
```

> 빠뜨리면 다음 기동에서 `world readable permissions` 경고가 뜨고, mosquitto 상위 버전에서는 **브로커가 뜨지 않는다**(§2.2).

#### 반영 방법 — SIGHUP 과 재시작을 구분한다

```bash
docker kill -s HUP mqtt-broker        # 대부분의 설정. 접속 유지
```

브로커 로그에 `Reloading config.` 가 찍히면 반영된 것이다.

| 변경 항목 | 반영 방법 |
|---|---|
| `password_file`, `acl_file` 내용 | **SIGHUP** |
| `connection_messages`, `log_type` | **SIGHUP** |
| `max_queued_messages`, `max_inflight_messages` | **SIGHUP** |
| `persistent_client_expiration` | **SIGHUP** |
| **`listener` 추가·삭제** | **재시작** |
| **리스너의 TLS 인증서 경로** | **재시작** |
| **compose 의 `user:`, `logging:`, `ports:`** | **`docker compose up -d`** (컨테이너 재생성) |

리스너를 바꾸고 SIGHUP 을 보내면 조용히 무시되지 않고 다음 오류가 남는다:

```
Reloading config.
Error: It is not currently possible to add/remove listeners when reloading the config file.
```

**이 줄이 보이면 재시작해야 한다.** 재시작은 300대 동시 재접속을 유발하므로 §8.5 를 함께 본다.

```bash
cd /sw/docker/mqtt && docker compose restart
docker compose logs --tail 20        # Opening ipv4 listen socket on port ... 확인
```

`docker-compose.yml` 자체를 고쳤다면 `restart` 로는 반영되지 않는다. 컨테이너를 다시 만들어야 한다:

```bash
cd /sw/docker/mqtt && docker compose up -d      # 변경 감지 시 재생성
```

#### 수정 전 백업

```bash
cd /sw/docker/mqtt/config
cp -p mosquitto.conf mosquitto.conf.$(date +%F)
```

`-p` 가 소유자와 권한을 함께 보존한다.

> ⚠️ 백업본을 `config/` 안에 두면 브로커가 읽지는 않지만(`-c` 로 지정한 파일만 읽는다) 디렉터리가 지저분해진다. 장기 보관은 `~/` 등 밖으로 옮긴다.

---


## 9. 트러블슈팅

> **[호스트]** — 접속 수 조회만 **[컨테이너 내부]**

| 증상 | 원인 | 조치 |
|---|---|---|
| 브로커가 안 뜸 / `refuse to load` | `passwd`·`acl` 권한 | `chmod 600`. 소유자는 `docker` 여야 한다 (§2.2) |
| `Config loaded` 로그가 없음 | 마운트 경로 불일치 | §6.1 볼륨 경로 확인 |
| 컨테이너 `unhealthy` 반복 | 헬스체크에 `central` 사용 | `central` 은 읽기 권한이 없어 ACL 이 거부한다. `health` 계정을 쓸 것 (§4.2) |
| 특정 Agent만 인증 실패 | 계정명 ≠ agentId | §5.1. `${HOSTNAME}_${USER}_J` 와 정확히 일치해야 함 |
| 명령이 간헐적으로 유실 | 방화벽 idle timeout | §7.8. half-open. idle timeout 상향 |
| 접속은 되는데 명령이 안 옴 | ACL 위반 (SUBACK은 정상) | §7.3. 전달 시점 차단이라 구독은 성공해 보인다 |
| 재시작 후 명령이 조용히 사라짐 | `data/` 유실 → 세션 없음 | §8.4 복원 |
| `data/` 에 root 소유 파일 | `.env` 의 `MQTT_UID`/`MQTT_GID` 누락 → 컨테이너가 root 로 뜸 | §6.2. 재생성 후 root 파일 삭제는 인프라 담당 요청 |
| 두 Agent 가 무한 재접속 | clientId 충돌 (session takeover) | 한 호스트·한 계정으로 Agent 를 두 개 띄운 경우. agentId 가 겹친다 (§5.1) |
| 브로커 호스트 디스크 고갈 | Agent 재접속 반복 → 접속 로그 누적 | §2.3 로테이션. 근본 원인은 §7.8(idle timeout) 또는 clientId 충돌 |
| 설정 파일이 `Permission denied` | 소유자가 `docker` 가 아님 | `ls -l` 확인. root 소유면 인프라 담당에 소유권 이전 요청 (§8.7) |
| 설정을 고쳤는데 반영이 안 됨 | SIGHUP 미전송 또는 리스너 변경 | §8.7. 리스너 추가·삭제는 **재시작**이 필요하다 |
| SIGHUP 후 `not currently possible to add/remove listeners` | 리스너를 바꾸고 SIGHUP 을 보냄 | `docker compose restart` (§8.7) |
| 미상 프로토콜로 차단 | IPS/DPI 오탐 | §1.1. 평문이라 DPI 가 페이로드를 본다. 예외 등록 요청 |
| 오프라인 Agent 에 명령이 안 쌓임 | **구독 QoS 가 0** | 구독에 `-q 1`. QoS 0 은 큐잉 대상이 아니다 (§7.5) |
| 발행이 성공하는데 아무도 못 받음 | ACL 거부인데 MQTT 3.1.1 로 확인 | `-V 5` 로 PUBACK 사유 코드 확인. `RC:135` = 거부 (§7.4) |
| 재접속했는데 큐가 비어 있음 | clientId 불일치 또는 세션 만료 | `-i <agentId>` 고정, `-x` 를 메시지 만료보다 길게 (§7.5) |

**[호스트]** — 브로커 로그 확인:
```bash
cd /sw/docker/mqtt && docker compose logs --tail 100 -f
```

**[컨테이너 내부]** — 현재 접속 수 조회. 돌고 있는 컨테이너 안에서 실행하는 명령은 이것과 §8.5 의 같은 명령 **둘뿐**이다:
```bash
docker exec mqtt-broker mosquitto_sub -h localhost -p 1883 \
  -u ops -P '<ops 비밀번호>' -t '$SYS/broker/clients/connected' -C 1
```

> `-h localhost` 는 **컨테이너 안에서 본 자기 자신**이다. 호스트에서 같은 명령을 쓰려면 `docker exec` 를 빼고 `-h <브로커 IP>` 로 바꾼다.

---

## 부록 A. 나중에 TLS를 켤 때

무중단 전환이 가능하다.

1. 사내 CA 에서 서버 인증서 발급 — **SAN 에 FQDN 과 IP 를 모두 포함**시킨다. 폐쇄망에서는 DNS 없이 IP 로 접속하는 경우가 잦은데 SAN 에 IP 가 없으면 검증이 실패한다. 가장 흔한 실패 원인이다.
2. 방화벽에 8883 추가 신청 — **§1.1 에서 1883 을 신청할 때 함께 올려두면 리드타임이 절약된다.**
3. `mosquitto.conf`에 8883 리스너를 추가한다. `per_listener_settings`가 기본 `false`라 `password_file`·`acl_file`·`allow_anonymous`가 **양쪽 리스너에 그대로 적용된다.**
   ```conf
   listener 8883
   protocol mqtt
   certfile /mosquitto/certs/broker.crt      # 중간 CA 체인 포함
   keyfile  /mosquitto/certs/broker.key
   require_certificate false                 # mTLS 미사용
   ```
4. `corp-ca.crt`를 Agent truststore(`keytool -importcert`)와 중앙서버(`tls_set(ca_certs=...)`)에 배포한다.
5. Agent를 배치별로 8883으로 전환한다.
6. 전부 넘어간 것을 확인하고 1883 리스너를 삭제, 방화벽도 닫는다.

전환 후에는 §8.3 의 재접속 폭주에 TLS 핸드셰이크가 더해진다. 300대 × RSA-2048 은 약 1초간 코어를 점유하므로 **ECDSA 인증서 + 세션 재개(session resumption)** 를 쓰고 jitter 를 반드시 확인한다.

**인증서 만료일을 자산 목록에 등록한다.** 폐쇄망은 만료 알림이 오지 않아 그대로 서비스가 멈춘다.

## 부록 B. nginx를 두지 않는 이유

MQTT는 HTTP가 아니라 nginx `stream`(L4) 모듈이 필요하며, path 기반 엔드포인트를 만들 수 없다. 만들 수 있는 것은 `mqtt.corp.example.com:1883`, 즉 **도메인 + 포트**까지다.

**도메인 이름만이 목적이라면 DNS A 레코드로 충분하다.** 브로커가 단일 노드라 로드밸런싱 이득이 없고, mosquitto 2.0은 **PROXY protocol을 지원하지 않아** 프록시를 거치면 접속 로그의 출발지 IP를 300대 모두 잃는다.

사내 표준이 "모든 외부 유입은 nginx 경유"라면 `stream` 블록으로 구성하되 `proxy_timeout 3600s`를 반드시 지정한다(기본 10분).
