# 001: 런타임 이미지 크기 측정 (alpine baseline → 빌드 플래그 적용)

- **날짜**: 2026-08-14
- **Phase**: Phase 1
- **유형**: 성능 측정

## 가설 / 목표

alpine 기반 이미지의 크기 baseline을 기록하고, 빌드 플래그
(`-ldflags "-s -w"`, `-trimpath`) 적용이 이미지 크기를 줄이는지 확인한다.

## 방법

```
docker build -t api-server:test ./
docker images api-server:test    # DISK USAGE / CONTENT SIZE 확인
```

## 결과

| 항목 | Before (플래그 없음) | After (-s -w, -trimpath) |
|---|---|---|
| DISK USAGE (로컬 압축 해제) | 27.5MB | 23.4MB (-15%) |
| CONTENT SIZE (전송/압축) | 8.78MB | 6.78MB (-23%) |

## 해석

<!-- 직접 작성: -s -w가 무엇을 제거해서 줄어든 것인가?
     감수하는 것(트레이드오프)은 없는가? (힌트: 디버그 심볼) -->

## 후속 조치

- ADR-0001 재검토(Phase 8) 시 distroless 전환하면 alpine(~7MB)분이 추가로
  줄어들 수 있음 — 그때 이 표에 행 추가
