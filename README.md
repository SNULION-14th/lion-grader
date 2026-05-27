# SNULION 14th 배포 챌린지 채점기 (lion-grader)

배포 세미나 챌린지의 두번째 과제인, 총 네 개의 서비스에 대한 결함 풀이를 자동 검증하는 CLI 채점기입니다.
*자신이 AWS에 배포한 백엔드* 위에서 동작합니다.

---

## 전체 흐름

```
 1. base 레포 4개 clone (lionpay·lionchat·lionlink·lionscribe)
 2. 결함 분석 + 코드·인프라 수정
 3. 자신의 AWS 계정에 배포 (EC2, S3, RDS, Route 53 등)
 4. 채점기 다운로드 → 자기 백엔드 URL 대상으로 실행
 5. ✅ PASS 결과 확인
```

채점은 **각 학생의 배포 환경 대상**으로 진행됩니다. 한 채점기를 모두가 같이
사용하지만, 채점 대상(`--target`)은 각자 다릅니다.

---

## 1. 다운로드

[**Releases**](https://github.com/SNULION-14th/lion-grader/releases/latest) 페이지에서
최신 `lion-grader` 바이너리를 받습니다.

```bash
wget https://github.com/SNULION-14th/lion-grader/releases/latest/download/lion-grader
chmod +x lion-grader
./lion-grader --help
```

### 지원 플랫폼

- **Linux x86_64 단일 실행 파일** (Python 설치 불필요)
- Mac/Windows 사용자: 자기 **EC2 인스턴스에 SSH로 접속해서 실행**을 권장합니다 (EC2는 Linux). 또는 Linux VM·WSL2에서 실행.

---

## 2. 사용법

```bash
./lion-grader --service <name> --target <url> [--chaos-token <token>] [--no-color]
```

| 옵션 | 설명 |
|---|---|
| `--service` | `lionpay` · `lionchat` · `lionlink` · `lionscribe` 중 하나 |
| `--target` | 자신이 배포한 백엔드 base URL (예: `https://lionpay-api.example.com`). 끝의 `/`는 자동 제거. **`/api` 등 prefix는 빼고 도메인까지만 적습니다.** |
| `--chaos-token` | **LionScribe에만 필요.** EC2 `.env`에 설정한 `CHAOS_TOKEN` 값 |
| `--no-color` | ANSI 색상 비활성 (CI·로그 캡처용) |

### 4 서비스 채점 명령어

```bash
# LionPay
./lion-grader --service lionpay --target https://YOUR-LIONPAY-API.example.com

# LionChat
./lion-grader --service lionchat --target https://YOUR-LIONCHAT-API.example.com

# LionLink
./lion-grader --service lionlink --target https://YOUR-LIONLINK-API.example.com

# LionScribe (chaos token 필수)
./lion-grader --service lionscribe \
  --target https://YOUR-LIONSCRIBE-API.example.com \
  --chaos-token YOUR_CHAOS_TOKEN
```

---

## 3. 무엇을 검증하나요?

| 서비스 | 검증 핵심 | 채점 시간 |
|---|---|---|
| **LionPay** | 송금 동시성 : hot-sender 80건 동시 송금(총합 보존) + 잔액 부족 동시 송금 1건만 통과(행 잠금) | ~30초 |
| **LionChat** | 동기 블로킹 : 무거운 LLM 호출 12건과 가벼운 GET 50건 동시 부하 시, 가벼운 요청 p95 < 1000ms | ~1분 |
| **LionLink** | SSRF :: IMDS·loopback·private·DNS 우회·옥타/십진/IPv6-mapped 인코딩 우회 8 vector 차단 | ~10초 |
| **LionScribe** | Durable queue : 5건 작업 제출 후 chaos restart, 90초 뒤 5/5 completed + 실제 PDF 생성 | ~2분 |

각 채점은 PASS/FAIL 외에도 **무엇을 시도했고, 어떤 결과가 나왔는지, 왜 실패했는지, 어느 방향으로 풀어야 하는지**를 친절하게 안내합니다.

---

## 4. PASS 결과 예시

```
  ╔════════════════════════════════════════════════════════════════╗
  ║   ✅ PASS  LionLink (Sec14 — SSRF)                             ║
  ║                                                                ║
  ║   차단: 8 / 8 (V1~V8 전부)                                     ║
  ║                                                                ║
  ║   내부 endpoint·IMDS·DNS 우회·IP 인코딩 우회 모두 차단됨.      ║
  ║   (ipaddress 모듈 기반 정밀 validation이 적용된 것으로 보임)   ║
  ╚════════════════════════════════════════════════════════════════╝
```

이 박스가 보이면 통과입니다.

## FAIL 결과 예시

```
  ╔════════════════════════════════════════════╗
  ║   ❌ FAIL  LionPay (C2 — 송금 동시성)      ║
  ║                                            ║
  ║   시나리오 A: FAIL (총합 보존 4120, ...)   ║
  ║   ...                                      ║
  ╚════════════════════════════════════════════╝
```

FAIL 시 박스 위에 출력된 **진단 / 힌트**를 읽어 어느 부분이 깨졌는지 파악할 수 있습니다.

---

## 5. 채점 시 주의사항

- **채점기는 학생 백엔드의 DB에 데이터를 만듭니다** — 채점용 사용자 수십 개,
  송금/preview/job 레코드 등. 채점이 끝난 뒤 정리를 원하면 Django shell이나
  관리 페이지에서 삭제하세요. *채점 통과 여부에는 영향이 없습니다.*

- **LionScribe 채점은 백엔드를 강제 종료 시킵니다** — `systemd Restart=always`
  같은 자동 재시작 설정이 없으면 백엔드가 다시 안 살아납니다. 채점기가 자동으로
  health check를 시도하니, 20초 안에 복구되지 않으면 FAIL이 떨어집니다.
  - **EC2 인스턴스 자체나 SSH 세션은 끊기지 않습니다** — chaos는 gunicorn worker
    프로세스 하나만 종료시킵니다. systemd가 즉시 다시 spawn하고, 채점기 프로세스는
    별도라 영향 없이 계속 동작합니다. 안심하고 EC2에서 실행하세요.

- **백엔드 ALLOWED_HOSTS / CORS 설정** — Django의 `ALLOWED_HOSTS`에 채점 도메인이
  포함돼야 합니다. 자기 도메인을 정확히 적었는지 확인하세요.

- **채점기 자체는 부수효과 없는 안전한 binary**입니다. 자기 EC2든 로컬 Linux든
  실행에 추가 패키지가 필요 없습니다.

---

## 6. 예상되는 오류들의 원인

| 증상 | 원인 / 해결 |
|---|---|
| `connection refused` | 백엔드가 안 떠 있거나 URL이 틀림. `curl -i <target>/api/users/` 로 먼저 확인 |
| `400 Bad Request` 응답 분포 다수 | `ALLOWED_HOSTS`에 도메인 누락. settings.py 확인 |
| `[FAIL] 가벼운 요청 모두 실패` | gunicorn worker가 전부 점유. LionChat 비동기 풀이 필요 |
| LionScribe — `백엔드가 20초 안에 복구되지 않음` | `systemd Restart=always` 미설정. systemd unit 파일 확인 |
| LionScribe — `--chaos-token` 누락 | EC2 `.env`의 `CHAOS_TOKEN` 값을 `--chaos-token`으로 넘기세요 |
| `[FAIL] 채점 도중 예외 — SSLError` | HTTPS 인증서 만료/자체 서명. Let's Encrypt 등 유효 인증서 사용 |

---

## 8. 종료 코드

| 코드 | 의미 |
|---|---|
| `0` | PASS |
| `1` | FAIL (진단 메시지가 출력에 포함) |
| `2` | 채점기 예외 (네트워크 오류, 잘못된 URL 등) |
