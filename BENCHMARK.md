# MQTT 브로커 실측 결과

Agent 300대 규모에서 Mosquitto 브로커의 실제 부하를 측정한 기록이다.
설계 근거로 쓰인 추정치를 수치로 대체하는 것이 목적이다.

측정일: **2026-09-16**

---

## 0. 테스트 환경

| 항목 | 값 |
|---|---|
| OS | Ubuntu 22.04.4 LTS (kernel 6.8.0-138) |
| CPU | **Intel Celeron J4025 @ 2.00GHz / 2 cores** |
| RAM | 7 GiB |
| Docker | 26.0.0 |
| Docker Compose | 2.3.3 |
| 브로커 | `eclipse-mosquitto:2.0` → mosquitto **2.0.22** (이미지 10.6 MB) |
| 클라이언트 | `mosquitto_sub`/`mosquitto_pub` 2.0.22, `paho-mqtt` 2.1.0 (Python) |
| 프로토콜 | MQTT 5.0, QoS 1, 평문 1883 |

> ⚠️ **CPU 가 저사양이라는 점을 감안해서 읽어야 한다.** Celeron J4025 는 2 코어 저전력 칩이다.
> 아래 CPU 수치는 운영 서버에서 **더 낮게** 나올 것이므로, 여기서 여유가 있다면 운영에서도 여유가 있다고 볼 수 있다.

### 측정의 한계

정직하게 적어둔다. 이 수치를 운영 용량 산정에 쓸 때 고려해야 한다.

- **클라이언트가 브로커와 같은 호스트에서 돌았다.** 루프백 통신이므로 실제 네트워크 지연·손실이 반영되지 않았다.
- **메모리는 `docker stats` 의 컨테이너 RSS 다.** mosquitto 내부 힙(`$SYS/broker/heap/current`)과는 다르다.
- 트래픽 수치는 컨테이너 네트워크 계층 기준이라 **TCP/IP 헤더를 포함**한다.
- 측정 중 한 차례, 클라이언트 컨테이너가 조기 종료된 상태에서 잰 값이 있었다. 해당 회차는 폐기하고 접속 수를 확인한 뒤 재측정했다. 아래 값은 모두 **측정 시점에 접속 수를 확인한 것**이다.

---

## 1. 결론 요약

| 질문 | 답 |
|---|---|
| 300개 상시 연결이 부하가 되는가 | **아니다.** 2.58 MiB / CPU 0.06%. 접속 0개와 CPU 가 같다 |
| 무엇이 실제로 자원을 쓰는가 | **오프라인 큐.** 연결 수가 아니라 큐에 쌓인 명령 건수에 비례한다 |
| 최악의 경우 메모리는 | **16.55 MiB** (300 세션 × 100건 만재). `memory_limit 512MB` 의 3% |
| WebSocket 보다 네이티브 TCP 가 가벼운가 | **그렇다.** 메모리 8.4배, CPU 14배, 접속 설정 64배 차이 |
| `max_queued_messages 100` 은 적절한가 | **보수적이다.** 512 MB 를 채우려면 세션당 약 3,400건이 필요하다 (34배 여유) |

---

## 2. 상시 연결 300개

Agent 300대가 접속만 유지하고 명령이 흐르지 않는 정상 상태.

클라이언트: `mosquitto_sub -V 5 -q 1 -c -x 86400 -k 30` × 300 (각자 고유 계정·clientId)

| 상태 | 메모리 | CPU |
|---|---|---|
| 접속 0개 (기준선) | 1.23 MiB | 0.06% |
| **접속 300개 유지** | **2.56 MiB** | **0.06%** |
| 300건 명령 발행 직후 | 2.56 MiB | 0.07% |

- **연결당 메모리 ≈ 4.5 KB** — (2.56 − 1.23) MiB ÷ 300
- **CPU 는 접속 0개와 구분되지 않는다.** 유휴 샘플 5회: 0.06 / 0.06 / 0.06 / 0.07 / 0.07 %
- 브로커 `$SYS/broker/clients/connected` = 300 으로 확인

### 유휴 트래픽

`keepAlive 30` 설정으로 300개 연결이 유지될 때, 60초간 증가량:

```
60초 전: 216 kB in / 128 kB out
60초 후: 337 kB in / 209 kB out
```

**≈ 2 kB/s (300개 합계)**. PINGREQ 자체는 2바이트이며, 측정값의 대부분은 TCP/IP 헤더다.

### 왜 이렇게 가벼운가

1. **mosquitto 는 단일 스레드 epoll 이다.** 연결마다 스레드를 만들지 않고 300개 fd 를 하나의 이벤트 루프에서 감시한다. 유휴 연결은 깨울 일이 없으므로 비용이 거의 0 이다.
2. **유휴 MQTT 연결은 소켓 하나 + 작은 구조체다.**
3. **이 설계에는 폴링이 없다.** Agent 는 구독 전용, 중앙서버는 발행 전용이라 명령이 없으면 트래픽이 발생하지 않는다. 상태 토픽도 쓰지 않는다.

### 명령 발행

300대 전체에 순차 발행(`mosquitto_pub` 프로세스 300회 기동 포함):

```
300건 발행 소요: 1.89 초
발행 직후 브로커: 2.562 MiB / CPU 0.07%
```

프로세스 기동·TCP 연결·CONNECT 핸드셰이크가 300회 포함된 값이므로 **브로커 몫은 이 중 일부**다. 단일 연결을 유지하는 실제 중앙서버 방식은 이보다 훨씬 빠르다.

---

## 3. 오프라인 큐 — 유일하게 실제로 늘어나는 항목

Agent 300대가 전부 꺼진 상태에서 세션을 유지한 채(`-c -x 86400`) 명령을 쌓았다.
`max_queued_messages 100` 이므로 세션당 100건이 상한이다.

페이로드는 실제 명령 형식의 JSON (~130 bytes):
```json
{"cmdId":"q-1-1","type":"RESTART_SERVICE","args":{"serviceName":"nginx","force":false},"issuedAt":"2026-09-16T05:00:00Z"}
```

| 상태 | 메모리 | 저장 메시지 |
|---|---|---|
| 300 세션 오프라인, 큐 비어 있음 | 2.13 MiB | 0 |
| 큐잉 30초 경과 | 5.79 MiB | — |
| 큐잉 60초 경과 | 9.65 MiB | — |
| **300 세션 × 100건 만재** | **16.55 MiB** | **30,023** |
| 재접속·전달 후 | 3.77 MiB | 51 |

- **큐 메시지 1건당 ≈ 500 bytes.** 페이로드 130 bytes 에 MQTT 메타데이터와 토픽 문자열이 더해진 값이다.
- **CPU 는 30,000건을 쌓는 동안에도 0.06%** 로 변화 없음
- **전달 후 메모리가 회수된다** (16.55 → 3.77 MiB)

### `max_queued_messages` 판단

| 값 | 300 세션 만재 시 예상 메모리 |
|---|---|
| **100 (현재 설정)** | **~16.5 MiB** |
| 1000 (mosquitto 기본값) | ~150 MiB |
| `memory_limit 512MB` 를 채우는 값 | 세션당 약 3,400건 |

현재 값은 **34배 여유**가 있다. 다만 올릴 이유는 없다 — 명령은 1시간에 자동 소멸하므로 한 세션에 100건 넘게 쌓이는 것 자체가 비정상이며, 그 경우 큐를 늘리기보다 원인을 봐야 한다.

---

## 4. 네이티브 TCP vs WebSocket

같은 브로커에 리스너만 달리 구성하고, **같은 클라이언트 라이브러리(paho-mqtt 2.1.0)** 로 전송 방식만 바꿔 비교했다.

```conf
listener 1883
protocol mqtt
# ── vs ──
listener 1883
protocol websockets
```

| 항목 | 네이티브 TCP | WebSocket | 배수 |
|---|---|---|---|
| 기준선 메모리 (접속 0) | 1.28 MiB | **18.54 MiB** | 14× |
| 기준선 CPU (접속 0) | 0.07% | 0.08% | 차이 없음 |
| 300 접속 메모리 | 2.58 MiB | 21.69 MiB | 8.4× |
| **연결당 메모리** | ~4.4 KB | ~10.7 KB | 2.4× |
| **CPU (300 접속 유휴)** | 0.05~0.06% | 0.76~0.92% | ~14× |
| **300개 접속 소요 시간** | **0.49 초** | **31.62 초** | 64× |
| keepAlive 트래픽 (60초) | ~240 kB | ~219 kB | **차이 없음** |

### 해석

**① 리스너를 켜는 것만으로 18 MiB 를 쓴다.** 접속이 0개인데도 14배 차이가 난다. libwebsockets 가 버퍼를 미리 확보하기 때문이다. 접속 0개일 때 CPU 는 양쪽 모두 0.07~0.08% 로 같으므로, 이것은 순수한 메모리 선점이며 유휴 폴링 비용은 아니다.

**② 접속 설정이 압도적으로 느리다.** HTTP Upgrade 핸드셰이크가 연결마다 왕복을 추가한다. **브로커 재시작 시 300대 재접속 시나리오에 직접 영향이 간다** — 네이티브는 1초 내에 끝나지만 WebSocket 은 30초 이상 걸린다.

> ⚠️ 단, 이 64배에는 **paho Python 의 WebSocket 구현 비용이 상당 부분 섞여 있다.** 브로커만의 비용이 아니므로 배수를 그대로 받아들이면 안 된다. 방향성만 유효하다.

**③ 트래픽 차이는 없다.** 예상과 다르지만 이유가 명확하다. keepAlive PINGREQ 는 MQTT 2바이트인데 WebSocket 프레이밍이 6~10바이트를 더한다. 그런데 **TCP/IP 헤더가 이미 40~56바이트**라 전체에서 차이가 묻힌다. 명령 페이로드(~130 bytes)에서도 프레임 헤더는 2~5% 수준이다.

### Agent 측 변경 범위

전환 시 **코드 변경은 없다.** Paho 는 URI 스킴으로 전송 방식을 결정하고, `org.eclipse.paho.mqttv5.client` 에 WebSocket 지원이 내장되어 있어 의존성 추가도 없다.

```java
new MqttClient("tcp://10.x.y.10:1883", agentId)        // 현재
new MqttClient("ws://10.x.y.10:9001/mqtt", agentId)    // WebSocket
```

`subscribe`, 콜백, `sessionExpiryInterval`, 메시지 만료 처리는 그대로다. MQTT 의미론은 전송 방식과 무관하다.

**단, 설정 형식이 URI 를 담을 수 있어야 한다.** `agent.properties` 를 `mqtt.host` + `mqtt.port` 로 나눠두면 스킴을 표현할 수 없어 Agent 가 코드에서 `"tcp://" + host + ":" + port` 로 조립하게 되고, 그러면 전송 방식 변경이 **jar 재빌드 + 300대 재배포**가 된다. `mqtt.broker.uri=tcp://host:1883` 한 줄로 두면 설정 교체로 끝난다(`INSTALL.md` §5.8).

### 판단

**"WebSocket 이 무거워서 못 쓴다"는 결론은 과하다.** 21.69 MiB 는 절대값으로 아무것도 아니다. 네이티브를 택하는 이유는 부하가 아니라 **이 환경에서 1883 이 직접 도달 가능하므로 HTTP Upgrade 계층을 끼울 이유가 없다**는 것이다.

거꾸로 **WebSocket 을 택해야 하는 상황도 분명하다** — 방화벽이 443 만 열어주거나, DPI 가 MQTT 를 미상 프로토콜로 차단하는 경우다. 그때는 위 비용을 감수하는 것이 맞고, 감수할 만한 수준이다.

---

## 5. 부수적으로 확인된 동작

성능 측정 과정에서 함께 확인된 것들. 설치·운영 시 함정이 되는 항목이다.

### 5.1 ACL 거부 발행이 MQTT 3.1.1 에서는 성공으로 보인다

ACL 로 쓰기 권한이 없는 계정이 발행을 시도했을 때:

| 프로토콜 | PUBACK | 종료코드 | 판정 가능? |
|---|---|---|---|
| MQTT 3.1.1 (기본) | `RC:0` | 0 | **불가 — 성공처럼 보인다** |
| MQTT 5 (`-V 5`) | **`RC:135`** (Not authorized) | 0 | 가능 |

메시지는 양쪽 모두 실제로 전달되지 않는다(구독자 수신 0건 확인). 3.1.1 에는 PUBACK 사유 코드가 없어 발행자가 알 수 없을 뿐이다. **권한 검증 시 `-V 5` 는 필수다.**

### 5.2 오프라인 큐잉은 구독 QoS 1 이 필요하다

발행을 QoS 1 로 해도 **구독이 QoS 0 이면 유효 QoS 는 0** 이라 큐에 쌓이지 않는다. 오류도 경고도 없이 누락되므로 가장 찾기 어렵다.

큐잉에 필요한 조건 — **하나라도 빠지면 동작하지 않는다**:

| 옵션 | 의미 |
|---|---|
| `-q 1` | 구독 QoS |
| `-c` | clean session 해제 |
| `-i <clientId>` | clientId 고정 |
| `-x <초>` | 세션 만료. MQTT 5 는 기본값이 0(즉시 소멸)이다 |

### 5.3 메시지 만료

`-D PUBLISH message-expiry-interval` 로 발행한 두 건을 오프라인 큐에 넣고 8초 후 재접속:

```
만료 5초   → exp-A  : 미전달 (폐기됨)
만료 300초 → keep-B : 전달
```

> `mosquitto_pub` 의 `-x` 는 **session-expiry-interval 이지 메시지 만료가 아니다.**

### 5.4 토픽 격리

`pattern read cmd/%u/req` 기준, `central` 로 `cmd/myhost01_wasadm_J/req` 에 실제 발행했을 때:

| 구독자 | 결과 |
|---|---|
| `myhost02_wasadm_J` (남의 토픽) | **`Timed out` — 수신 0건** |
| `myhost01_wasadm_J` (자기 토픽) | `{"cmdId":"iso-test"}` 수신 |

**구독 자체는 SUBACK 0(성공)으로 응답되고 거부는 전달 시점에 적용된다.** SUBACK 으로는 격리 여부를 판정할 수 없다.

### 5.5 SIGHUP 은 `mosquitto.conf` 도 다시 읽는다

`password_file` 전용이 아니다. `connection_messages` 를 `false` 로 바꾸고 SIGHUP 을 보내면 반영되며, 로그에 `Reloading config.` 가 남는다.

단 **리스너 추가·삭제는 SIGHUP 으로 안 된다.** 조용히 무시되지 않고 명시적으로 오류를 남긴다:

```
Reloading config.
Error: It is not currently possible to add/remove listeners when reloading the config file.
```

재시작하면 해당 포트의 listen socket 이 열린다.

비밀번호 갱신 시 SIGHUP 동작 확인:

| 확인 항목 | 결과 |
|---|---|
| 기존 접속 유지 | **유지됨** (재시작 아님) |
| 새 비밀번호로 접속 | 성공 |
| 옛 비밀번호로 접속 | `Connection Refused: not authorised.` |

### 5.6 Docker 로그는 기본적으로 로테이션되지 않는다

```
docker inspect <컨테이너> --format '{{.HostConfig.LogConfig.Config}}'
→ map[]        ← max-size, max-file 둘 다 없음 = 무제한
```

`connection_messages true` 기준 **접속 1회당 브로커 로그 약 300~550 bytes.** 정상 운영 중에는 거의 늘지 않지만, Agent 가 재접속을 반복하면 누적된다:

| 상황 | 하루 로그량 |
|---|---|
| 정상 (300대 접속 유지) | 최초 ~100 KB, 이후 증가 없음 |
| idle timeout 으로 45초마다 재접속 | ~150 MB/일 |
| clientId 충돌로 초당 재접속 | ~7 GB/일 |

> **브로커가 꺼져 있는 동안에는 브로커 로그가 늘지 않는다.** 그때 재접속 실패 로그를 쏟는 것은 Agent 300대이며, 그것은 각 Agent 호스트의 디스크다.

### 5.7 Compose 의 CPU 제한이 올바로 적용되지 않는다

Compose **2.3.3** 기준:

| 설정 | `NanoCpus` | `Memory` |
|---|---|---|
| `deploy.resources.limits: { cpus: "2", memory: 2G }` | **2** (2 CPU 가 아니라 2 나노CPU) | 2147483648 ✅ |
| `cpus: 2` / `mem_limit: 2g` | **0** (미적용) | 2147483648 ✅ |

메모리는 양쪽 정상이다. **CPU 제한은 두지 않는 편이 안전하다** — 이 워크로드는 CPU 를 쓰지 않으므로 제한의 이득이 없고, 잘못 적용되면 브로커만 느려진다.

### 5.8 Compose 의 `.env` 조회 위치

| 실행 방식 | 읽는 `.env` |
|---|---|
| 프로젝트 디렉터리에서 `docker compose up` | 프로젝트 디렉터리의 `.env` |
| 다른 위치에서 `docker compose -f <경로>/compose.yml up` | **여전히 compose 파일이 있는 디렉터리의 `.env`** |
| 현재 디렉터리에만 `.env` 존재 | **무시됨** |

또한 `.env` 값은 **YAML 치환에만 쓰이고 컨테이너 환경변수로 들어가지 않는다**(`env_file:` 과 다르다). 컨테이너 안에서 `env` 로 조회해도 나오지 않는다.

값이 없으면 오류가 아니라 **빈 문자열로 조용히 진행**된다:
```
level=warning msg="The \"MQTT_UID\" variable is not set. Defaulting to a blank string."
```

### 5.9 이미지 구성

`eclipse-mosquitto:2.0` 이 10.6 MB 인 내역:

| 레이어 | 크기 |
|---|---|
| Alpine 3.23.5 minirootfs | 8.41 MB |
| mosquitto 2.0.22 빌드 결과물 + 런타임 | 2.14 MB |
| entrypoint 스크립트 | 318 B |

브로커 바이너리 자체는 **301 KB** 다. 동적 링크는 musl libc / OpenSSL / libwebsockets 뿐이며, JVM·Erlang VM 같은 런타임을 끼고 다니지 않는다. EMQX(~200 MB)·HiveMQ(~250 MB) 와의 차이는 기능이 아니라 런타임 유무에서 온다.

---

## 6. 재현 방법

### 상시 연결 부하 (§2)

```bash
# 계정 300개 생성
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD/config:/w" \
  --entrypoint sh eclipse-mosquitto:2.0 -c '
  for i in $(seq 1 300); do mosquitto_passwd -b /w/passwd "host$(printf %03d $i)_wasadm_J" "pw$i"; done
  mosquitto_passwd -b /w/passwd central cpw
  mosquitto_passwd -b /w/passwd ops opw'

# 300개 접속 (한 컨테이너에서 백그라운드 프로세스로)
docker run -d --name clients --network host --entrypoint sh eclipse-mosquitto:2.0 -c '
  for i in $(seq 1 300); do
    ID="host$(printf %03d $i)_wasadm_J"
    mosquitto_sub -h 127.0.0.1 -p 1883 -V 5 -u "$ID" -P "pw$i" -i "$ID" \
      -t "cmd/$ID/req" -q 1 -c -x 86400 -k 30 >/dev/null 2>&1 &
  done; wait'

# 측정 — 접속 수를 반드시 함께 확인한다
docker run --rm --network host eclipse-mosquitto:2.0 mosquitto_sub \
  -h 127.0.0.1 -p 1883 -u ops -P opw -t '$SYS/broker/clients/connected' -C 1
docker stats <브로커> --no-stream --format 'MEM={{.MemUsage}} CPU={{.CPUPerc}} NET={{.NetIO}}'
```

### 전송 방식 비교 (§4)

`paho-mqtt` 로 `transport` 만 바꾼다. 클라이언트 프로세스 모델을 동일하게 유지해야 비교가 성립한다.

```python
c = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2, client_id=cid,
                protocol=mqtt.MQTTv5, transport=transport)   # "tcp" | "websockets"
if transport == "websockets":
    c.ws_set_options(path="/mqtt")
c.connect(host, port, keepalive=30)
```

> 측정 시 **클라이언트 컨테이너가 살아 있는지 반드시 확인한다.** 클라이언트가 종료된 뒤 잰 값은 접속 0개 상태의 값이다. 이 함정으로 한 회차를 폐기했다.
