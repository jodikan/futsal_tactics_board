# 풋살 전술판 — 개발 히스토리 & 인수인계 문서

> Cowork에서 이어서 작업할 때 이 파일을 첨부하고 "이 문서 기반으로 풋살 전술판 개선 작업 이어서 해줘"라고 하면 컨텍스트 바로 이어받을 수 있어요.

---

## 1. 프로젝트 개요

**목적**: 풋살 팀 훈련 시 전술 움직임을 팀원들에게 시각적으로 공유하기 위한 도구

**산출물**: `futsal_tactics_board.html` — 단일 HTML 파일, 인터넷 연결 없이 브라우저에서 바로 실행 가능

**주요 사용 흐름**:
1. 선수/공 배치
2. 프레임 추가하며 위치 변경 (= 전술 흐름)
3. 재생으로 애니메이션 확인
4. 영상으로 저장 → 카톡/단톡방 공유

**공유 방법**:
- **파일 직접 전달**: html 파일 그대로 카톡/이메일로 전송 → 상대방이 브라우저에서 바로 실행 가능 (서버 불필요)
- **URL 공유**: GitHub Pages 또는 Netlify Drop에 파일을 올리면 무료로 URL 발급 가능

---

## 2. 개발 히스토리 (이터레이션 요약)

### v1 — 기본 전술판
- Canvas 기반 풋살 코트 렌더링
- 선수 추가 / 이동 / 지우기
- 경로(이동/드리블/패스)를 직접 그리는 방식
- 프레임 추가 후 GIF 저장

**문제점**: 경로선이 코트 위에 다 표시되어 가독성 저하

### v2 — 번호 체계 개선
- 선수 지우고 다시 추가하면 번호가 빈 번호부터 재시작
- `getNextPlayerNum()`: 전체 선수 중 사용되지 않은 가장 작은 번호 배정

### v3 — 팀별 번호 독립
- 색상(팀)이 다르면 번호를 1번부터 독립적으로 시작
- `getNextNum(color)`: 같은 색상 선수들 사이에서만 빈 번호 탐색

### v4 — 핵심 UX 개편
**경로 그리기 → 선수 원이 직접 움직이는 방식으로 전환**

- 프레임마다 선수 위치를 찍고 재생 시 easeInOut 보간으로 이동
- 편집 중 다음 프레임 방향을 반투명 점선 화살표(ghost trail)로 미리 표시
- GIF 내보내기 시도 — 브라우저 샌드박스 정책 문제로 CDN Worker 차단 → Blob URL 방식으로 해결

**GIF 인코딩 버그 경과**:
- 최초 GIF: 프레임마다 NeuQuant 인스턴스 새로 생성 → 각기 다른 팔레트 → 2번 프레임부터 색상 깨짐
- 수정 시도: 전 프레임 합산으로 NeuQuant 1회 훈련 → 팔레트 공유 → 여전히 심각하게 깨짐 (horizontal stripe 아티팩트)
- **결론**: 직접 구현한 NeuQuant+LZW 파이프라인의 근본적 신뢰성 문제. 수정보다 대안 방식이 적합하다고 판단

### v5 (현재 최신) — 영상 내보내기로 전환

**GIF 인코더 전면 제거 → `canvas.captureStream` + `MediaRecorder` 방식으로 교체**

핵심 변경:
- `makeWorkerBlob()`, `exportGif()` 삭제 — NeuQuant, LZW 코드 전부 제거
- `exportVideo()` 추가: 캔버스를 30fps 스트림으로 캡처하면서 실제 애니메이션을 재생 → MediaRecorder가 그대로 녹화
- 출력 포맷: WebM(vp9/vp8) 우선, MP4 폴백 — `MediaRecorder.isTypeSupported()`로 자동 선택
- 비트레이트: 4Mbps

장점:
- 256색 제한 없음 → 원본 Canvas 품질 그대로
- 인코딩 로직이 브라우저에 위임되므로 코드가 극도로 단순해짐
- GIF 대비 파일 크기 작고 품질 우수

제약:
- Chrome/Edge 권장 (Safari는 MediaRecorder 지원 불안정)
- 녹화 중 실제 애니메이션이 캔버스에서 재생됨 (사용자 조작 불가)

---

## 3. 현재 코드 구조

### 데이터 모델

```js
// 선수 목록 (전역)
players = [{ id: 'p1', color: '#E24B4A', num: 1 }, ...]

// 프레임 목록
frames = [
  { positions: { 'p1': {x, y}, 'p2': {x, y} }, ballPos: {x, y} | null },
  { positions: { ... }, ballPos: ... },
  ...
]
```

**핵심 설계 원칙**: 선수 정보(색상, 번호)는 전역 `players`에, 위치 정보는 각 `frames[i].positions[id]`에 분리 저장.

### 주요 함수

| 함수 | 역할 |
|---|---|
| `lerpPos(fromF, toF, t)` | 두 프레임 사이 선수/공 위치 보간 (t: 0~1) |
| `drawGhostTrails()` | 편집 중 다음 프레임 방향 점선 표시 |
| `renderState(positions, ballPos)` | 주어진 위치로 전체 화면 렌더링 |
| `togglePlay()` | rAF 루프로 easeInOut 애니메이션 재생 |
| `exportVideo()` | captureStream+MediaRecorder로 영상 녹화 후 다운로드 |
| `sleep(ms)` | Promise 기반 비동기 대기 |
| `getNextNum(color)` | 팀(색상)별 독립 번호 부여 |

### 영상 내보내기 흐름

```
exportVideo()
  ├─ canvas.captureStream(30fps) → MediaRecorder 시작
  ├─ frames[0] 정지 (PAUSE_MS)
  ├─ for fi in 0..frames.length-2:
  │    ├─ rAF 루프: lerpPos(frames[fi], frames[fi+1], ease) → renderState
  │    └─ 도착 후 정지 (PAUSE_MS)
  └─ recorder.stop() → Blob → a.click() 다운로드
```

### Canvas 렌더링 구조
- `drawCourtOn(ctx)`: 코트 배경 (그라디언트 + 라인 마킹)
- `drawPlayerOn(ctx, p, x, y)`: 선수 원 + 번호
- `drawBallOn(ctx, x, y)`: 공

---

## 4. 현재 파일 목록

| 파일 | 설명 |
|---|---|
| `futsal_tactics_board.html` | 메인 앱 (단일 파일, 외부 의존 없음) |
| `futsal_tactics_handover.md` | 이 문서 |

---

## 5. 개선 아이디어 (백로그)

우선순위 높음:
- [ ] 선수 이름/포지션 레이블 표시 옵션
- [ ] 전술 저장/불러오기 (JSON export/import)
- [ ] 프레임 재생 속도 조절 슬라이더

우선순위 중간:
- [ ] 코트 방향 선택 (가로/세로)
- [ ] 선수 번호 직접 편집
- [ ] 패스 경로 별도 표시 옵션 (선수 움직임과 분리)
- [ ] 모바일 터치 UX 개선

우선순위 낮음:
- [ ] 전술 템플릿 (기본 포메이션 preset)
- [ ] 실행취소(Undo) 기능
- [ ] 프레임 순서 드래그로 변경

---

## 6. 기술 스택

- **순수 HTML/CSS/JS** — 프레임워크 없음, 단일 파일
- **Canvas 2D API** — 코트 및 선수 렌더링
- **canvas.captureStream + MediaRecorder** — 영상 녹화 (v5~)
- **requestAnimationFrame** — 재생 및 녹화 중 애니메이션 루프

---

## 7. 작업 환경 참고

- 제작자: 조은 (풋살팀 픽소, 팀 캡틴)
- 팀 구성: 8인, 화요일 삼성역 정기 풋살
- 주요 용도: 월패스, 침투 등 전술 설명용 영상 제작 후 단톡방 공유
