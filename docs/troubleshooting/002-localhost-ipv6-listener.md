# 002: localhost는 404, 127.0.0.1은 200 — 같은 포트, 다른 응답

- **날짜**: 2026-08-14
- **Phase**: Phase 1
- **소요 시간**: (직접 기입)

## 증상 (Symptom)

```
localhost:8080/healthz  → 404
127.0.0.1:8080/healthz  → 200
```

404 응답은 연결 실패가 아니라 Go 서버의 정상 응답이었다
(`404 page not found`, `X-Content-Type-Options: nosniff`).

## 원인 (Root Cause)

WSL에서 `/healthz`가 포함되지 않은 이전 `./server` 프로세스가 PID `4827`로
`*:8080`을 사용하고 있었다. WSL의 localhost 전달 프로세스인 `wslrelay.exe`가
이를 Windows의 `[::1]:8080`에 노출했다.

동시에 Docker Desktop은 `api-test` 컨테이너를 `0.0.0.0:8080`과 `[::]:8080`에
공개했다. `localhost`가 IPv6 `::1`로 해석되면 더 구체적인 `[::1]:8080`
리스너가 선택돼 WSL의 이전 서버가 404를 반환했다.

## 해결 (Resolution)

1. WSL의 실제 리스너 PID 4827 확인 (`netstat -ano | findstr ":8080"`)
2. 실행 경로와 명령 확인
3. PID 4827 정상 종료
4. wslrelay의 `[::1]:8080` 리스너 제거 확인

## 검증 (Verification)

IPv4·IPv6 healthz 재검증 — 두 주소 체계 모두 200 확인.

## 배운 것 / 재발 방지

- Docker와 WSL 개발 서버를 동시에 실행할 때 호스트 포트를 분리한다
- 컨테이너 실행 전 `netstat`과 `docker ps`로 기존 리스너를 확인한다
- `localhost` 오류 시 `curl -4`와 `curl -6` 결과를 분리한다
- 이미지를 새로 빌드한 뒤 기존 컨테이너도 재생성한다
- `200` 하나만 확인하지 않고 실제 운영에서 사용할 주소 체계까지 검증한다
