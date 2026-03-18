# ⚽ 풋살 전술판

풋살 팀 훈련 시 전술 움직임을 시각화하고 영상으로 공유하는 도구입니다.
단일 HTML 파일로 동작하며, 외부 라이브러리나 서버 없이 브라우저에서 바로 실행됩니다.

---

## 기능 요약

- 선수/공을 코트에 배치하고 드래그로 위치 조정
- 프레임 기반으로 전술 흐름 작성 (키프레임 방식)
- 편집 중 다음 프레임 이동 방향을 점선 화살표로 미리 표시 (ghost trail)
- 재생 버튼으로 선수가 부드럽게 이동하는 애니메이션 확인
- MP4/WebM 영상으로 내보내기 → 단톡방 공유

---

## 실행 방법

`index.html` 파일을 브라우저에서 열면 바로 실행됩니다.
또는 GitHub Pages URL로 접속:

```
https://your-username.github.io/futsal-tactics/
```

> 영상 저장 기능은 **Chrome 또는 Edge** 권장 (Chrome 130+에서 MP4 출력)

---

## 개발 히스토리

### v1 — 기본 전술판
- Canvas 기반 풋살 코트 렌더링
- 선수 추가 / 이동 / 지우기
- 경로(이동/드리블/패스)를 직접 그리는 방식

**문제점**: 경로선이 코트 위에 모두 표시되어 가독성 저하

### v2 — 번호 체계 개선
- 선수를 지우고 다시 추가하면 빈 번호부터 재시작
- `getNextPlayerNum()`: 전체 선수 중 사용되지 않은 가장 작은 번호 배정

### v3 — 팀별 번호 독립
- 색상(팀)이 다르면 번호를 1번부터 독립적으로 시작
- `getNextNum(color)`: 같은 색상 선수들 사이에서만 빈 번호 탐색

### v4 — 핵심 UX 개편
경로 직접 그리기 방식을 폐기하고, **선수 원이 직접 이동하는 방식**으로 전환

- 프레임마다 선수 위치를 저장하고, 재생 시 프레임 사이를 easeInOut 보간으로 이동
- 편집 중 다음 프레임 이동 방향을 ghost trail(반투명 점선 화살표)로 미리 표시
- 재생: 800ms 이동 + 300ms 정지 사이클

### v5 (현재) — 영상 내보내기

`canvas.captureStream` + `MediaRecorder` API로 영상 녹화 방식 도입

- 캔버스 스트림을 30fps로 캡처하면서 애니메이션을 직접 재생 → 브라우저가 인코딩
- 출력 포맷: MP4 우선(`avc1` 코덱), WebM 폴백 — `MediaRecorder.isTypeSupported()`로 자동 선택
- 비트레이트: 4Mbps
- 녹화 중 캔버스 조작 비활성화

---

## 코드 구조

### 데이터 모델

```js
// 선수 목록 (전역)
players = [{ id: 'p1', color: '#E24B4A', num: 1 }, ...]

// 프레임 목록
frames = [
  { positions: { 'p1': {x, y}, 'p2': {x, y} }, ballPos: {x, y} | null },
  ...
]
```

선수 정보(색상, 번호)는 전역 `players`에, 위치 정보는 `frames[i].positions[id]`에 분리 저장.
선수를 삭제할 때 다른 프레임에서 해당 id만 제거하면 되는 구조.

### 주요 함수

| 함수 | 역할 |
|---|---|
| `lerpPos(fromF, toF, t)` | 두 프레임 사이 선수/공 위치 보간 (t: 0~1) |
| `drawGhostTrails()` | 편집 중 다음 프레임 방향 점선 화살표 표시 |
| `renderState(positions, ballPos)` | 주어진 위치 상태로 전체 화면 렌더링 |
| `togglePlay()` | rAF 루프로 easeInOut 애니메이션 재생/정지 |
| `exportVideo()` | captureStream + MediaRecorder로 영상 녹화 후 다운로드 |
| `getNextNum(color)` | 팀(색상)별 독립 번호 부여 |

### 영상 내보내기 흐름

```
exportVideo()
  ├─ canvas.captureStream(30fps) → MediaRecorder 시작
  ├─ frames[0] 정지 (PAUSE_MS=600)
  ├─ for fi in 0..frames.length-2:
  │    ├─ rAF 루프: lerpPos(frames[fi], frames[fi+1], ease) → renderState (MOVE_MS=800)
  │    └─ 도착 프레임 정지 (PAUSE_MS=600)
  └─ recorder.stop() → Blob → a.click() 다운로드
```

### Canvas 렌더링 구조

| 함수 | 역할 |
|---|---|
| `drawCourtOn(ctx)` | 코트 배경 (그라디언트 + 라인 마킹) |
| `drawPlayerOn(ctx, p, x, y)` | 선수 원 + 번호 |
| `drawBallOn(ctx, x, y)` | 공 |

---

## 기술 스택

- **순수 HTML/CSS/JS** — 프레임워크 없음, 외부 의존 없음, 단일 파일
- **Canvas 2D API** — 코트 및 선수 렌더링
- **canvas.captureStream + MediaRecorder** — 영상 녹화
- **requestAnimationFrame** — 재생 및 녹화 중 애니메이션 루프

---

## 백로그

우선순위 높음:
- [ ] 선수 이름/포지션 레이블 표시 옵션
- [ ] 전술 저장/불러오기 (JSON export/import)
- [ ] 프레임 재생 속도 조절 슬라이더

우선순위 중간:
- [ ] 코트 방향 선택 (가로/세로)
- [ ] 선수 번호 직접 편집
- [ ] 패스 경로 별도 표시 옵션
- [ ] 모바일 터치 UX 개선

우선순위 낮음:
- [ ] 전술 템플릿 (기본 포메이션 프리셋)
- [ ] 실행취소 (Undo)
- [ ] 프레임 순서 드래그로 변경

---

## 프로젝트 정보

- 제작자: 조은
- 주요 용도: 월패스, 침투 등 전술 설명용 영상 제작 후 단톡방 공유
