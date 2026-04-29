# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

실리콘 컵 물 증발 시뮬레이터 — 순수물 및 히알루론산(HA) 수용액의 증발을 물리 모델 기반으로 시뮬레이션하는 단일 페이지 웹 앱. 빌드 툴체인 없이 순수 HTML/CSS/JS로 작성됐으며 Vercel에 정적 사이트로 배포된다.

## 파일 구조

- `index.html` — 메인 배포 파일. 반응형(모바일/태블릿) CSS 포함
- `evaporation_calculator.html` — 이전 버전 (반응형 미포함, ~246줄 차이). 레거시 참고용
- `vercel.json` — GitHub 자동 배포 설정 (`{"github": {"enabled": true}}`)

**작업 기준 파일은 `index.html`**이다. `evaporation_calculator.html`은 수정하지 않는다.

## 로컬 개발

빌드/컴파일 단계 없음. 브라우저에서 직접 열면 된다.

```bash
# Windows
start index.html

# 또는 VS Code Live Server 확장을 사용하면 핫리로드 가능
```

## 외부 라이브러리 (CDN)

```
Chart.js 4.4.0    — 2D 증발 시간-부피 그래프
Three.js 0.134.0  — 3D 컵 시각화 (OrbitControls 포함)
XLSX 0.18.5       — 시뮬레이션 결과 엑셀 다운로드
```

버전을 올릴 때는 Three.js OrbitControls import 경로가 함께 바뀌는지 확인한다 (`examples/js/controls/OrbitControls.js`).

## 핵심 아키텍처

모든 로직은 `<script>` 블록 하나에 있다 (약 line 1640~2490).

### 물리 엔진 — 순수물 (`simulate`, line 1681)

Fick의 확산 법칙 기반 해석해:

- `D_water(T_K)` — 온도별 수분 확산계수 (m²/s)
- `c_sat(T_K)` — 포화 수증기 농도 (Magnus 공식)
- `cupVolume / cupRadius / findH0` — 원뿔대 컵 기하 계산 (이진탐색으로 수위 역산)
- 구동력: `deltaC = c_sat × (1 − RH)`
- 특성 시간 `tau_s`는 해석적으로 계산, 500포인트 샘플링

### 물리 엔진 — 히알루론산 (`simulateHA`, line 1752)

수치 적분 (Euler, `dt_s` ≤ 30 s, 최대 4000 스텝):

- `HA_PARAMS` — 분자량별 k₂ 계수 (50 kDa ~ 2 MDa)
- 수분활성도 모델: `a_w = exp(−k₂ × c_HA²)`
- 평형 농도: `c_eq = sqrt(−ln(RH) / k₂)`
- 질량 보존으로 농도 추적: `c_HA = c₀ × V₀ / V`

### UI 흐름

1. `DOMContentLoaded` → `link()` 함수로 number input ↔ range slider 동기화
2. 슬라이더/입력 변경 → `update()` 호출
3. `update()` → `simulate()` + (HA 활성 시) `simulateHA()` → `refreshWaterChart()` / `refreshHAChart()` → DOM 메트릭 갱신
4. 3D 컵은 `init3DCup()` / `drawCup()` → Three.js requestAnimationFrame 루프

### 차트 구성

- `initCharts()` (line 1971): Chart.js 인스턴스 2개 생성 (`waterChart`, `haChart`)
- `makeAnnotationPlugin()` — 커스텀 x축 마커 플러그인 (Chart.js 내장 annotation 미사용)
- `makeChartOptions()` — 두 차트 공통 옵션 팩토리

### 수식 패널

`updateFormula()` (line 2222) — 슬라이더 값이 바뀔 때마다 계산된 물리 파라미터를 `<details>` 수식 박스에 갱신한다.

## 배포

```bash
# vercel.json이 GitHub 자동 배포를 설정하므로 main 브랜치에 push하면 자동 배포됨
git push origin main
```

## 주의사항

- 모든 단위는 SI 내부 연산, UI 표시만 mm/μL/min
- `tau_w_min` — HA 시뮬레이션에서 순수물 tau를 참고값으로 넘겨 타임스텝 크기를 결정함
- `findH0` 이진탐색 80회 반복 — 정밀도 충분, 변경 금지
- `STORE_EVERY = 5` — HA 수치적분에서 5스텝마다 1포인트 저장 (메모리/성능 트레이드오프)
