# SNULION 14th 챌린지 채점기

세미나 챌린지 과제(과제 2)의 4 서비스 결함 풀이를 검증하는 CLI 채점기입니다.

## 다운로드

[**Releases**](https://github.com/SNULION-14th/lion-grader/releases/latest) 페이지에서
최신 `lion-grader` 바이너리를 받습니다. Linux x86_64 단일 실행 파일이라 추가 설치 없습니다.

```bash
# EC2에서 (Linux x86_64)
wget https://github.com/SNULION-14th/lion-grader/releases/latest/download/lion-grader
chmod +x lion-grader
```

## 사용법

```bash
./lion-grader --service <name> --target <url> [--chaos-token <token>]
```

- `--service`: `lionpay` · `lionchat` · `lionlink` · `lionscribe` 중 하나
- `--target`: 배포한 백엔드 base URL (예: `https://lionpay-api.example.com`)
- `--chaos-token`: LionScribe 채점에만 필요. EC2의 `.env`에 설정한 `CHAOS_TOKEN` 값
- `--no-color`: 출력에서 ANSI 색상 제거

### 예시

```bash
# LionPay 채점
./lion-grader --service lionpay --target https://your-lionpay-api.example.com

# LionScribe 채점 (chaos token 필요)
./lion-grader --service lionscribe \
  --target https://your-lionscribe-api.example.com \
  --chaos-token $(cat ~/chaos-token.txt)
```

## 무엇을 검증하나요?

| 서비스 | 검증 항목 |
|---|---|
| **LionPay** (C2) | 송금 동시성 — hot-sender 80건 동시 송금 + 잔액 부족 동시성 |
| **LionChat** (C5) | 동기 블로킹 — 무거운 LLM 호출 12건 + 가벼운 GET 50건 부하 |
| **LionLink** (Sec14) | SSRF — IMDS·loopback·DNS 우회·IP 인코딩 우회 8 vector |
| **LionScribe** (R1) | Durable queue — chaos restart 후 작업 보존 + 유효 PDF 생성 |

각 채점은 PASS / FAIL 외에도 **무엇을 시도했는지·왜 실패했는지·어느 방향으로 풀어야 하는지**를 친절하게 안내합니다.

## 제출

채점기를 자신의 배포 환경에 돌려 `✅ PASS` 결과가 나오는 **스크린샷**을 제출합니다.

## 종료 코드

- `0` — PASS
- `1` — FAIL (구체 진단은 출력 참조)
- `2` — 예외 (네트워크 오류 등)
