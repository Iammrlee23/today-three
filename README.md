# 오늘, 셋 — 루틴·업무 트래커 (오프라인 우선 PWA)

루틴 자동 표시(매일·평일·매주·매월·**매년**), 시작 시 "오늘 챙길 것" 팝업,
오늘 화면의 지연/오늘 마감/임박/진행중 그룹 보기와 14일 내 다가오는 일정,
**월간·주간 캘린더**(날짜별 루틴 달성/업무 목표일 표시, 날짜 클릭 시 상세),
업무별 카운터파트·목표일·실제 완료일 추적, 화면 잠금(PIN)을 제공하는
서버 없는 정적 웹앱입니다. 모든 데이터는 기기(localStorage)에만 저장되며 인터넷 없이 동작합니다.

> **데이터 분리**: 계정·서버가 없으므로 접속하는 사람마다 자기 기기에만 데이터가 저장됩니다.
> 다른 사람이 URL로 접속해도 내 데이터는 보이지 않습니다 (각자 빈 앱에서 시작).
> 같은 기기를 공유한다면 기록 탭에서 **PIN 잠금**을 설정하세요.

## 파일 구성

```
index.html      앱 전체 (HTML+CSS+JS 단일 파일)
sw.js           서비스 워커 — 오프라인 캐싱
manifest.json   PWA 매니페스트 — 홈화면 설치/APK용
icons/          앱 아이콘 (192/512px)
README.md       이 문서
```

## 1. 로컬에서 바로 실행

`index.html`을 브라우저로 열면 즉시 동작합니다.
(서비스 워커는 http(s) 환경에서만 등록되므로, 오프라인 캐시까지 확인하려면 아래처럼 로컬 서버 사용)

```bash
python3 -m http.server 8000
# 브라우저에서 http://localhost:8000 접속
```

## 2. GitHub Pages 배포

```bash
# 1) 새 저장소 생성 후 (예: today-three)
git init
git add .
git commit -m "feat: 오늘셋 PWA 초기 버전"
git branch -M main
git remote add origin https://github.com/<아이디>/today-three.git
git push -u origin main
```

2) GitHub 저장소 → Settings → Pages → Source: `Deploy from a branch`,
   Branch: `main` / `(root)` 선택 → Save

3) 1~2분 후 `https://<아이디>.github.io/today-three/` 에서 접속 가능
   - 모든 경로가 상대경로(`./`)로 작성되어 있어 하위 경로 배포에서도 그대로 동작합니다.
   - 한 번 접속하면 서비스 워커가 전체를 캐시하므로 이후 비행기 모드에서도 열립니다.

## 3. 안드로이드 APK 만들기

방법 A — **PWABuilder (권장, 가장 간단)**
1. https://www.pwabuilder.com 접속 → 배포된 GitHub Pages URL 입력
2. Android 패키지 선택 → APK/AAB 다운로드
3. 생성된 APK는 PWA를 감싼 형태(TWA)로, 설치 후 오프라인에서도 동작

방법 B — **Bubblewrap CLI**
```bash
npm i -g @bubblewrap/cli
bubblewrap init --manifest https://<아이디>.github.io/today-three/manifest.json
bubblewrap build   # → app-release-signed.apk 생성
```

방법 C — **설치만 필요하다면 APK 없이도 가능**
안드로이드 Chrome에서 사이트 접속 → 메뉴 → "홈 화면에 추가" → 독립 앱처럼 실행 (오프라인 포함)

> 참고: APK 빌드 자체는 인터넷이 1회 필요하지만, 설치된 앱은 외부 연결 없이 완전 동작합니다.

## 4. 데이터 관리

- 저장 위치: 브라우저/앱의 localStorage (기기 외부로 전송되지 않음)
- 기기 변경·브라우저 데이터 삭제에 대비해 **기록 탭 → JSON 내보내기**로 주기적 백업 권장
- 다른 기기에서 JSON 가져오기로 복원 가능

## 5. 기능 ↔ 인지심리학 근거

| 기능 | 근거 |
|---|---|
| 루틴 사전 등록 → 해당일 자동 표시 | 전망 기억(prospective memory)의 외부화 — 회상 부담 제거 |
| 시작 시 "오늘 챙길 것" 팝업 | 실행 의도: 회상 시점을 행동 시점과 연결 (Gollwitzer, 1999) |
| 목표일 vs 실제 완료일, 평균 지연 지표 | 계획 오류 보정 데이터 축적 (Kahneman & Tversky, 1979) |
| 머릿속 일을 즉시 기록 | 미완료 과제의 침투적 사고 감소 (Zeigarnik, 1927; Masicampo & Baumeister, 2011) |
| 진행 바 · 7일 달성 뷰 | 목표 구배 효과 (Hull, 1932; Kivetz et al., 2006) |
