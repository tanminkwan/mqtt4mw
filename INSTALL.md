# MQTT 브로커 운영 설치 가이드

대상: **Docker Compose 단일 노드 / 폐쇄망 / Agent 300대 / 평문 1883**
설계 근거는 `DESIGN.md`. 이 문서는 "그대로 따라 하면 되는" 절차만 담는다.

---

## 0. 전제와 범위

| 항목 | 값 |
|---|---|
| 브로커 | `eclipse-mosquitto:2.0` (2.0.22 검증) |
| 프로토콜 | MQTT 5.0 |
| 리스너 | **평문 1883 단일** (TLS 미사용 — §0.1) |
| 인증 | `password_file` + ACL |
| 실행 | Docker Compose, 단일 노드 (Swarm 미사용) |
| 인스턴스 | **1개.** Mosquitto는 클러스터링이 없다 (§10, §16.6) |
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
| Agent별 계정 분리 — 하나가 새도 그 Agent 하나만 위험 | §5 |
| `central` 자격증명을 가장 엄격히 관리 (300대 전체 명령 권한) | §5.4 |
| 헬스체크 계정을 무력화 — 노출돼도 할 수 있는 게 없음 | §4.2 |

나중에 TLS를 켜려면 `DESIGN.md` §15.4(사내 CA, SAN에 FQDN+IP)를 따른다. 8883 리스너를 추가하고 전환 기간에 두 포트를 함께 열면 무중단으로 넘어갈 수 있다.

### 0.2 소요 시간

| 단계 | 시간 |
|---|---|
| §1 방화벽 신청 | **수일~수주** (병목) |
| §2~§6 설치 | ~1시간 |
| §7 검증 | ~1.5시간 (유휴 테스트 1시간 포함) |

---

## 1. 선행 작업

### 1.1 방화벽 신청

`DESIGN.md` §15.2 기준. **포트를 8883이 아닌 1883으로** 신청한다. 출발지는 개별 IP 300개가 아니라 **대역(CIDR)**으로 낸다.

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

> **idle timeout이 이 설치의 최대 사고 요인이다.** 값이 짧으면 양쪽 다 끊긴 줄 모르는 half-open 상태가 되어, 서버는 정상 publish + PUBACK을 받지만 Agent에는 영영 도달하지 않는다 (§15.3-a).

체크리스트:
- [ ] 6개 항목 접수, Agent 대역 CIDR 확정
- [ ] idle timeout ≥ 60초 확인 (권장 3600)
- [ ] NAT 경유 시 세션 테이블 여유 확인 — 상시 300세션이 **회수되지 않고** 점유된다
- [ ] IPS/DPI 애플리케이션 검사 여부 확인 — MQTT 시그니처가 없는 장비는 미상 프로토콜로 차단할 수 있다 (§15.3-c). **TLS가 없으므로 DPI가 페이로드를 그대로 본다**

### 1.2 이미지 반입 경로

사내 레지스트리 등록이 가능한지, tar 반입인지 확인한다. 이미지는 **10.6 MB**라 tar 반입도 부담이 없다.

---

## 2. 호스트 준비

### 2.1 디렉터리

```bash
sudo mkdir -p /srv/mqtt/{config,data,log}
```

### 2.2 권한 — 건너뛰면 브로커가 안 뜬다

컨테이너 내부 mosquitto는 **uid/gid 1883**으로 동작한다.

```bash
sudo chown -R 1883:1883 /srv/mqtt
sudo chmod 700 /srv/mqtt/config
```

mosquitto 2.0은 `passwd`·`acl`이 world-readable이거나 소유자가 다르면 경고한다:

```
Warning: File /mosquitto/config/acl has world readable permissions.
         Future versions will refuse to load this file.
```

**"Future versions will refuse to load"는 예고다.** 폐쇄망에서 브로커가 안 뜨는 상황은 복구가 번거로우니 처음부터 맞춘다. 설정 디렉터리를 `:ro`로 마운트하면 컨테이너 entrypoint의 `chown`이 실패하므로 **반드시 호스트에서** 잡는다.

### 2.3 로그 로테이션

브로커가 오래 죽어 있으면 Agent 300대가 재접속 실패 로그를 쏟아내 **디스크가 찬다** (§16.7). Compose 파일의 `logging:` 블록(§6.1)으로 처리하며, 데몬 기본값도 함께 잡아두면 안전하다.

`/etc/docker/daemon.json`:
```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "5" }
}
```
```bash
sudo systemctl restart docker      # 기존 컨테이너 재시작됨. 설치 전에 수행
```

---

## 3. 이미지 반입

### 3.1 사내 레지스트리

```bash
# 인터넷 가능 구간
docker pull eclipse-mosquitto:2.0
docker tag  eclipse-mosquitto:2.0 registry.corp.local/mqtt/eclipse-mosquitto:2.0.22
docker push registry.corp.local/mqtt/eclipse-mosquitto:2.0.22
```

### 3.2 tar 반입

```bash
# 인터넷 가능 구간
docker pull eclipse-mosquitto:2.0
docker save eclipse-mosquitto:2.0 -o mosquitto-2.0.22.tar     # ~10 MB
sha256sum mosquitto-2.0.22.tar > mosquitto-2.0.22.sha256

# 폐쇄망
sha256sum -c mosquitto-2.0.22.sha256
docker load -i mosquitto-2.0.22.tar
```

### 3.3 검증

```bash
docker run --rm --entrypoint mosquitto <이미지> -h | head -3
# → mosquitto is an MQTT v5.0/v3.1.1/v3.1 broker.
```

> **태그는 `2.0`이 아니라 `2.0.22`로 고정한다.** `2.0`은 이동 태그라 반입 시점에 따라 다른 바이너리가 들어올 수 있다.

---

## 4. 설정 파일

저장소의 `broker/config/` 내용과 동일하다. 운영 경로로 배치한다.

### 4.1 `/srv/mqtt/config/mosquitto.conf`

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

# 오프라인 Agent 세션 보존 기간. 명령 만료(1h)보다 충분히 길게
persistent_client_expiration 7d

# 세션당 큐 상한 — 명령은 1h 에 자동 소멸한다 (§12.5)
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

> **MQTT 5 관련 설정은 없다.** mosquitto 2.0은 3.1.1과 5.0을 같은 리스너에서 동시에 받고, `Message Expiry Interval` 처리는 기본 동작이다. 프로토콜 버전은 클라이언트가 결정한다 (Python `protocol=mqtt.MQTTv5`, Java `org.eclipse.paho.mqttv5.client`).

### 4.2 `/srv/mqtt/config/acl`

```
# 컨트롤러: 명령 발행 전용. 읽기 권한 없음
user central
topic write cmd/#

# 운영자 — 브로커 지표 조회만
user ops
topic read  $SYS/#

# 헬스체크 전용 — 토픽 하나. 노출돼도 할 수 있는 게 없다 (§5.1)
user health
topic read  $SYS/broker/uptime

# Agent 공통 패턴 — %u 는 접속 username 으로 치환
pattern read  cmd/%u/req
pattern read  cmd/broadcast/req
```

`central`에 읽기 권한이 없고 Agent에 쓰기 권한이 없다. **발행 전용 / 구독 전용 설계가 ACL 레벨에서 강제된다.**

기동 시 다음 경고가 뜨는데 **정상이다.** `pattern` 행에 치환 문자가 없어서 나며, 모든 인증 사용자에게 적용된다 (§5.2, 실측 확인):

```
Warning: ACL pattern 'cmd/broadcast/req' does not contain '%c' or '%u'.
```

---

## 5. 계정 생성

### 5.1 username = agentId

ACL이 `%u` 치환에 의존하므로 **계정명은 agentId와 정확히 일치해야 한다.** agentId는 Agent가 스스로 만들지 않고 다음 규칙으로 조립된다 (§2.1):

```
agentId = ${HOSTNAME}_${USER}_J        # 예: myhost01_wasadm_J
```

> ⚠️ `DESIGN.md` §5.1의 일괄 생성 예시는 `agent-001` 형식인데, §2.1 개정 전의 잔재다. **운영에서는 실제 agentId를 쓴다.** `agent-001`로 만들면 Agent가 `myhost01_wasadm_J`로 접속해 인증에 실패한다.

**Agent별 계정 분리는 타협 불가다.** 계정을 공유하면 `pattern read cmd/%u/req`가 모두 같은 토픽으로 풀려 아무 Agent나 남의 명령을 구독할 수 있다 (§5.1). 평문 구간에서는 이 분리가 더 중요하다 — 자격증명 하나가 새도 피해가 그 Agent 하나로 묶인다.

### 5.2 인벤토리

```bash
# /dev/shm/inventory.csv  — hostname,user
myhost01,wasadm
myhost02,wasadm
...
```

### 5.3 일괄 생성

```bash
IMG=registry.corp.local/mqtt/eclipse-mosquitto:2.0.22
WORK=/dev/shm/mqtt-prov                  # tmpfs. 디스크에 남기지 않는다
mkdir -p $WORK && cd $WORK
: > passwd && chmod 600 passwd           # ★ 먼저 잡아야 경고가 303번 뜨지 않는다

pwgen() { openssl rand -base64 24; }     # ~144 bit
add()  { docker run --rm --user "$(id -u):$(id -g)" -v "$WORK:/w" $IMG \
           mosquitto_passwd -b /w/passwd "$1" "$2"; }

# 운영 계정 3종
for u in central health ops; do
  PW=$(pwgen); add "$u" "$PW"; echo "$u,$PW" >> creds.csv
done

# Agent 300대
while IFS=, read -r host user; do
  [ -z "$host" ] && continue
  AID="${host}_${user}_J"
  PW=$(pwgen); add "$AID" "$PW"; echo "$AID,$PW" >> creds.csv
done < /dev/shm/inventory.csv

wc -l passwd                             # 303 이어야 한다
cut -d: -f1 passwd | head                # 계정명이 agentId 형식인지 육안 확인
sudo install -o 1883 -g 1883 -m 600 passwd /srv/mqtt/config/passwd
```

> `--user`와 `chmod 600`을 빼면 `mosquitto_passwd` 호출마다 권한 경고 3줄이 출력된다.
> 303회면 900줄이 쏟아져 **실제 실패를 놓친다.** 위 형태로 실행하면 출력이 없는 것이 정상이다.

생성 결과는 `username:$7$...` 해시 목록이다. **브로커는 평문을 가진 적이 없다.**

### 5.4 배포와 파기

평문 비밀번호와 해시는 **서로 다른 경로로** 나간다 (§5.1).

| 대상 | 주입 방식 |
|---|---|
| Agent 300대 | 설정파일 **권한 600, 서비스 계정 소유** |
| 중앙서버 (`central`) | `.env` 권한 600 또는 파이프라인 시크릿 |
| `health` | 배포 노드 `.env` (§6.2) |

- **환경변수로 Agent에 주입하지 말 것.** `/proc/<pid>/environ`, `ps e`, 코어덤프, 자식 프로세스로 전파된다.
- **이미지·jar 하드코딩 금지.** 300대에 같은 값이 박히고 회수가 불가능하다.
- Java 쪽에서 `MqttConnectionOptions`를 통째로 로그에 찍으면 자격증명이 샌다. `reasonString`만 찍는다 (§16.7).

```bash
shred -u $WORK/creds.csv && rm -rf $WORK    # 배포 완료 직후
```

`central`은 계정 1개지만 **가장 강력하다** — `topic write cmd/#`는 300대 전체에 임의 명령을 보낼 수 있다. Agent 자격증명 하나가 새면 그 Agent가 위험하지만, `central`이 새면 전체가 위험하다.

---

## 6. 배포

### 6.1 `/srv/mqtt/docker-compose.yml`

```yaml
services:
  broker:
    image: registry.corp.local/mqtt/eclipse-mosquitto:2.0.22
    container_name: mqtt-broker
    restart: unless-stopped
    ports:
      - "1883:1883"
    volumes:
      - /srv/mqtt/config:/mosquitto/config:ro
      - /srv/mqtt/data:/mosquitto/data
      - /srv/mqtt/log:/mosquitto/log
    healthcheck:
      # central 은 읽기 권한이 없다(§5.2). health 전용 계정을 쓴다
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

Agent 300대는 Mosquitto 단일 노드 용량(~10k 커넥션)의 **3% 수준**이다. 1 vCPU / 1 GB로 충분하며 위 값은 여유분이다 (§13).

### 6.2 `.env` (권한 600)

```bash
sudo tee /srv/mqtt/.env >/dev/null <<'ENV'
MQTT_HEALTH_PW=<health 비밀번호>
ENV
sudo chmod 600 /srv/mqtt/.env
```

> 이 값은 compose 파싱 시점에 컨테이너 설정에 박혀 **`docker inspect`로 평문 노출된다.** 숨기려 애쓰는 대신 **노출돼도 아무것도 못 하는 계정**을 쓴다 — `health`는 `$SYS/broker/uptime` 읽기 권한 하나뿐이다 (§5.1).

### 6.3 기동

```bash
cd /srv/mqtt && docker compose up -d
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

`Warning: ... world readable permissions`가 보이면 §2.2로 돌아간다.

**설정 파일은 compose에 파일명으로 나타나지 않는다.** 디렉터리를 통째로 마운트하고(`/srv/mqtt/config:/mosquitto/config:ro`), 이미지 기본 CMD가 `mosquitto -c /mosquitto/config/mosquitto.conf`이기 때문에 경로가 맞물려 적용된다. `Config loaded from ...` 로그로 확인한다.

---

## 7. 설치 검증

`DESIGN.md` §9의 운영판. **순서대로** 진행한다.

### 7.1 경로

```bash
# Agent 호스트에서 — 애플리케이션 없이 경로만 먼저
nc -zv <broker-ip> 1883
```

### 7.2 인증

```bash
# 정상 계정
mosquitto_sub -h <broker-ip> -p 1883 -u ops -P '<pw>' \
  -t '$SYS/broker/version' -C 1

# 익명 거부 확인
mosquitto_sub -h <broker-ip> -p 1883 -t '$SYS/#' -C 1
# → Connection Refused: not authorised.
```

### 7.3 토픽 격리

```bash
# agentA 계정으로 agentB 토픽 구독 시도
mosquitto_sub -h <broker-ip> -p 1883 \
  -u 'hostA_wasadm_J' -P '<pw>' -t 'cmd/hostB_wasadm_J/req' -v
```

> ⚠️ **SUBACK으로 판정하지 말 것.** mosquitto는 ACL 위반 구독도 SUBACK 0으로 응답하고 **전달 시점에 차단한다** (§9-3). 반드시 `central`로 해당 토픽에 실제 발행한 뒤 **수신되지 않음**을 확인한다.

### 7.4 발행 전용 / 구독 전용 강제

```bash
# central 은 읽기 권한이 없어야 한다 → 0건
mosquitto_sub -h <broker-ip> -p 1883 -u central -P '<pw>' -t 'evt/#' -W 5

# Agent 계정은 발행 권한이 없어야 한다
mosquitto_pub -h <broker-ip> -p 1883 -u 'hostA_wasadm_J' -P '<pw>' \
  -t 'cmd/hostA_wasadm_J/req' -m 'x'
```

### 7.5 메시지 만료

```bash
mosquitto_pub -h <broker-ip> -p 1883 -u central -P '<pw>' \
  -t 'cmd/hostA_wasadm_J/req' -q 1 \
  -D PUBLISH message-expiry-interval 10 -m '{"cmdId":"exp-A"}'
```

> `mosquitto_pub`의 `-x`는 **session-expiry-interval이지 메시지 만료가 아니다.**
> `-D PUBLISH message-expiry-interval <초>`가 맞다 (§9-2, 실측으로 확인된 함정).

Agent 세션을 미리 만들어 둔 상태에서 발행하고, 11초 후 접속시켜 `exp-A`가 **오지 않는지** 확인한다. 만료 이내(예: 3600초)로 보낸 메시지는 전달되어야 한다.

### 7.6 오프라인 큐잉

Agent를 내린 상태에서 명령을 발행하고, 재접속 시 전달되는지 확인한다. 이어서 브로커를 재시작한 뒤에도 큐가 살아 있는지 본다 (`persistence true`).

```bash
docker compose restart
```

### 7.7 장시간 유휴 — 폐쇄망 전용 필수 항목

```bash
# Agent 1대를 붙여놓고 1시간 이상 방치한 뒤 명령 발행
```

**건너뛰면 운영 중에 발견하게 된다.** 방화벽·NAT가 유휴 세션을 RST 없이 조용히 버리면, 브로커는 정상 publish + PUBACK을 받지만 Agent에는 도달하지 않는다 (§15.3-a).

### 7.8 체크리스트

- [ ] `nc -zv` 소통
- [ ] 익명 접속 거부
- [ ] 토픽 격리 — **발행 실측으로** 확인
- [ ] `central` 구독 0건 / Agent 발행 거부
- [ ] 메시지 만료 — 경과분 폐기, 이내분 전달
- [ ] 브로커 재시작 후 오프라인 큐 복원
- [ ] **1시간 유휴 후 명령 도달**
- [ ] `docker compose ps` → healthy

---

## 8. 운영 절차

### 8.1 비밀번호 갱신 — 무중단

mosquitto는 SIGHUP으로 `password_file`을 재읽기하며 **기존 연결을 끊지 않는다.** 인증은 CONNECT 시점에만 일어나기 때문이다.

```bash
# 설정 디렉터리가 :ro 이므로 호스트에서 수정한다
sudo docker run --rm --user 1883:1883 -v /srv/mqtt/config:/w \
  registry.corp.local/mqtt/eclipse-mosquitto:2.0.22 \
  mosquitto_passwd -b /w/passwd '<agentId>' '<new-pw>'

docker kill -s HUP mqtt-broker          # 재시작 아님. 300대 연결 유지
```

**순서가 중요하다.** 브로커를 먼저 갱신하면 해당 Agent는 접속 중에는 살아 있지만 **재접속하는 순간 실패**한다.

1. 브로커 갱신 + SIGHUP
2. Agent 설정파일 갱신
3. Agent 재기동

### 8.2 백업

```bash
docker compose stop
sudo tar czf mqtt-backup-$(date +%F).tgz -C /srv/mqtt config data
docker compose start
```

`data/mosquitto.db`는 세션과 오프라인 큐다. 유실되면 복구 후 발행분이 **세션이 없어 조용히 폐기된다** (§16.1-B).

### 8.3 브로커 재시작 시 주의

패치·설정 변경으로 재시작하면 **300대가 동시에 재접속한다.** Agent에 지수 백오프 + `random(0, 5s)` jitter가 들어 있는지 확인한다. Paho의 `setAutomaticReconnect`는 백오프는 하지만 **jitter가 없어 위상이 겹친다** (§13).

평문이라 TLS 핸드셰이크 부하는 없어 **재접속 폭주 자체는 수백 ms 내 완료된다** (§13). 다만 §16.7의 로그 폭증은 그대로 적용되므로 §2.3 로테이션을 확인한다.

### 8.4 계정 추가 (Agent 증설)

```bash
sudo docker run --rm --user 1883:1883 -v /srv/mqtt/config:/w \
  registry.corp.local/mqtt/eclipse-mosquitto:2.0.22 \
  mosquitto_passwd -b /w/passwd '<새 agentId>' '<pw>'
docker kill -s HUP mqtt-broker
```

ACL은 `pattern` 기반이라 **수정할 필요가 없다.** 계정만 추가하면 `cmd/%u/req`가 자동으로 적용된다.

---

## 9. 트러블슈팅

| 증상 | 원인 | 조치 |
|---|---|---|
| 브로커가 안 뜸 / `refuse to load` | `passwd`·`acl` 권한·소유자 | §2.2. **호스트에서** `chown 1883:1883`, `chmod 600` |
| `Config loaded` 로그가 없음 | 마운트 경로 불일치 | §6.1 볼륨 경로 확인 |
| 컨테이너 `unhealthy` 반복 | 헬스체크에 `central` 사용 | `central`은 읽기 권한이 없어 ACL이 거부한다. `health` 계정 (§5.1) |
| 특정 Agent만 인증 실패 | 계정명 ≠ agentId | §5.1. `${HOSTNAME}_${USER}_J` 와 정확히 일치해야 함 |
| 명령이 간헐적으로 유실 | 방화벽 idle timeout | §7.7. half-open. idle timeout 상향 |
| 접속은 되는데 명령이 안 옴 | ACL 위반 (SUBACK은 정상) | §7.3. 전달 시점 차단이라 구독은 성공해 보인다 |
| 재시작 후 명령이 조용히 사라짐 | `data/` 유실 → 세션 없음 | §8.2 복원 |
| 두 Agent가 무한 재접속 | clientId 충돌 (session takeover) | 한 계정으로 Agent 2개를 띄운 경우. 운영 규칙 위반 (§2.1) |
| 디스크 고갈 | 브로커 장기 다운 → 로그 폭증 | §2.3 로테이션 (§16.7) |
| 미상 프로토콜로 차단 | IPS/DPI 오탐 | §1.1. 평문이라 DPI가 페이로드를 본다. 예외 등록 요청 |

```bash
docker compose logs --tail 100 -f
docker exec mqtt-broker mosquitto_sub -h localhost -u ops -P '<pw>' \
  -t '$SYS/broker/clients/connected' -C 1      # 현재 접속 수
```

---

## 부록 A. 나중에 TLS를 켤 때

무중단 전환이 가능하다.

1. 사내 CA에서 서버 인증서 발급 — **SAN에 FQDN과 IP 모두 포함** (§15.4). IP 누락이 가장 흔한 실패다.
2. 방화벽에 8883 추가 신청 — **1883 신청 시 함께 올려두면 리드타임이 절약된다** (§15.2).
3. `mosquitto.conf`에 8883 리스너를 추가한다. `per_listener_settings`가 기본 `false`라 `password_file`·`acl_file`·`allow_anonymous`가 **양쪽 리스너에 그대로 적용된다.**
   ```conf
   listener 8883
   protocol mqtt
   certfile /mosquitto/certs/broker.crt      # 중간 CA 체인 포함
   keyfile  /mosquitto/certs/broker.key
   require_certificate false                 # mTLS 미사용 (§5.4)
   ```
4. `corp-ca.crt`를 Agent truststore(`keytool -importcert`)와 중앙서버(`tls_set(ca_certs=...)`)에 배포한다.
5. Agent를 배치별로 8883으로 전환한다.
6. 전부 넘어간 것을 확인하고 1883 리스너를 삭제, 방화벽도 닫는다.

전환 후에는 §8.3의 재접속 폭주에 TLS 핸드셰이크가 더해진다. ECDSA 인증서 + 세션 재개를 쓰고 jitter를 반드시 확인한다 (§13).

**인증서 만료일을 자산 목록에 등록한다.** 폐쇄망은 만료 알림이 오지 않아 그대로 서비스가 멈춘다.

## 부록 B. nginx를 두지 않는 이유

MQTT는 HTTP가 아니라 nginx `stream`(L4) 모듈이 필요하며, path 기반 엔드포인트를 만들 수 없다. 만들 수 있는 것은 `mqtt.corp.example.com:1883`, 즉 **도메인 + 포트**까지다.

**도메인 이름만이 목적이라면 DNS A 레코드로 충분하다.** 브로커가 단일 노드라 로드밸런싱 이득이 없고, mosquitto 2.0은 **PROXY protocol을 지원하지 않아** 프록시를 거치면 접속 로그의 출발지 IP를 300대 모두 잃는다.

사내 표준이 "모든 외부 유입은 nginx 경유"라면 `stream` 블록으로 구성하되 `proxy_timeout 3600s`를 반드시 지정한다(기본 10분).
