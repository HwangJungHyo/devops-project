# Troubleshooting 기록

실제로 겪은 문제와 해결 과정을 기록한다. "따라 만든 사람"과
"운영해본 사람"을 가르는 것이 문제 해결 경험이며, 이 기록은
겪은 직후가 아니면 복원되지 않는다.

## 규칙

- **막혀서 30분 이상 쓴 문제는 해결 직후 기록한다**
- 파일명: `NNN-짧은-증상.md` (예: `001-docker-build-version-mismatch.md`)
- 형식: 증상 → 원인 → 해결 → 검증 (template.md)
- 에러 메시지는 전문을 그대로 붙인다 — 검색되는 기록이 좋은 기록이다

## 목록

| # | 증상 | Phase | 근본 원인 |
|---|---|---|---|
| [001](./001-go-version-mismatch.md) | docker build 실패 (go 버전) | 1 | go.mod 요구 버전 > 빌더 이미지 버전, GOTOOLCHAIN=local |
| [002](./002-localhost-ipv6-listener.md) | localhost 404 / 127.0.0.1 200 | 1 | 이전 WSL 프로세스가 [::1]:8080 점유, localhost의 IPv6 해석 |
