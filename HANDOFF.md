# HANDOFF — 코드 검토·배포 인수인계 문서

작성일: 2026-06-11
대상: "오늘, 셋" 루틴·업무 트래커 (오프라인 우선 PWA)

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 형태 | 서버리스 정적 PWA (빌드 도구·프레임워크·외부 의존성 0) |
| 런타임 | 브라우저 단독. 네트워크 불필요 |
| 데이터 | localStorage 단일 키 `todaythree:v1` |
| 배포 | GitHub Pages (Actions 워크플로 포함) |
| 모바일 | PWA 설치 또는 PWABuilder/Bubblewrap으로 APK 래핑 |

## 2. 파일 구조

```
today-three/
├── index.html              앱 전체 (HTML + CSS + JS 단일 파일, ~31KB)
├── sw.js                   서비스 워커 (캐시 우선 + 백그라운드 갱신)
├── manifest.json           PWA 매니페스트
├── icons/icon-192.png      앱 아이콘
├── icons/icon-512.png
├── .github/workflows/deploy.yml   main push 시 Pages 자동 배포
├── README.md               사용자용 안내
└── HANDOFF.md              이 문서
```

의도적 단일 파일 구성: 외부 연결 금지 요건상 CDN/번들러를 배제했고,
규모(JS ~16KB)가 분리 비용을 정당화하지 않는다고 판단. 향후 확장 시 분리 권장 (아래 §7).

## 3. 데이터 스키마 (localStorage `todaythree:v1`)

```js
{
  routines: [{
    id: string,
    title: string,
    freq: "daily" | "weekdays" | "weekly" | "monthly" | "monthfirst" | "monthlast" | "yearly",
    days: number[],        // weekly일 때 요일 (0=일 … 6=토)
    dom: number,           // monthly/yearly일 때 일자 (1–31)
    month: number,         // yearly일 때 월 (1–12)
    counterpart: string,
    active: boolean        // false = 일시중지 (기본 true)
  }],
  routineLog: {            // 날짜별 루틴 완료 기록
    "YYYY-MM-DD": { [routineId]: true }
  },
  tasks: [{
    id: string,
    title: string,
    counterpart: string,
    status: "todo" | "doing" | "done" | "hold",
    targetDate: "YYYY-MM-DD" | "",   // 목표일
    actualDate: "YYYY-MM-DD" | "",   // 실제 완료일
    notes: string,
    createdAt: "YYYY-MM-DD"
  }],
  lastPopup: "YYYY-MM-DD",  // "오늘 챙길 것" 팝업 하루 1회 제어
  lastBackup: "YYYY-MM-DD"  // 마지막 JSON 내보내기일 — 30일 경과 시 팝업 리마인더
}
```

마이그레이션 정책: 스키마 변경 시 키를 `todaythree:v2`로 올리고
load()에서 v1 → v2 변환 코드를 추가할 것. (현재 v1 — yearly/month 추가는
기존 데이터에 영향 없는 additive 변경이라 키 유지)

별도 키 `todaythree:pin`: 화면 잠금 PIN의 해시(SHA-256, 비보안 컨텍스트에서는
djb2 폴백). 의도적으로 state 밖에 두어 JSON 백업/복원에 포함되지 않음 (기기별 잠금).
클라이언트 측 잠금이므로 "기기를 만지는 타인 차단" 용도이며 암호학적 보호가 아님.

## 4. 핵심 함수 맵 (index.html 내 <script>)

| 영역 | 함수 | 역할 |
|---|---|---|
| 데이터 | `load()` / `save()` | localStorage 입출력. save 실패 시 alert |
| 날짜 | `todayStr()`, `diffDays(a,b)` | 로컬 타임존 기준 YYYY-MM-DD, 일수 차 |
| 루틴 | `routineDueOn(r, date)` | 해당 날짜 표시 여부. **monthly/yearly는 말일 클램핑** (2/30 설정 + 평년 → 2/28 표시) |
| 잠금 | `hashPin`, `unlock`, `setPin`, `removePin`, `updatePinUI` | 화면 잠금. 해시는 `todaythree:pin` 키 (백업 미포함) |
| 렌더 | `render()` | 전체 5개 탭 일괄 리렌더 (상태 변경마다 호출) |
| 캘린더 | `renderCalendar()`, `dayItemsHtml(ds)` | 월간 그리드(42셀)·주간 리스트. `calMode/calAnchor/calSel` 상태, 날짜 클릭 시 상세 |
| 렌더 | `taskCard(t)` | 업무 카드. D-day / 지연 / 조기·지연 완료 배지 계산 |
| 렌더 | `renderStats(ts)` | 기한 준수율, 평균 지연일, 최근 7일 루틴 달성 |
| 팝업 | `showPopupIfNeeded(force)` | 지연 + D-1 이내 마감 + 미완료 루틴 집계, `lastPopup`으로 하루 1회 |
| 동작 | `toggleRoutine`, `quickDone`, `saveTask`, `saveRoutine` 등 | CRUD |
| 백업 | `exportData()` / `importData(ev)` | JSON 다운로드 / 복원(전체 교체, confirm 후) |
| 보안 | `esc(s)` | 모든 사용자 입력 출력 시 HTML 이스케이프 (XSS 방지) |
| 부팅 | 하단 즉시실행부 | load → render → 팝업 → 60초 간격 날짜 변경 감지 → SW 등록 |

## 5. 검토 체크리스트 (리뷰 시 확인 권장 지점)

- [ ] **XSS**: 사용자 입력이 innerHTML로 들어가는 모든 지점에 `esc()` 적용 여부 (title, counterpart, notes)
- [ ] **날짜/타임존**: `todayStr()`은 기기 로컬 타임존 기준. 자정 직후 동작, 해외 출장 시 타임존 변경 시나리오
- [ ] **자정 넘김**: 앱을 켜둔 채 날짜가 바뀌면 60초 폴링으로 재렌더 + 팝업 재표시 (`setInterval` 부분)
- [ ] **monthly 말일 클램핑**: `routineDueOn()`의 `Math.min(dom, 말일)` 로직
- [ ] **localStorage 한도**: 약 5MB. routineLog가 무한 누적되므로 장기 사용 시 정리 로직 필요 (§7 TODO)
- [ ] **SW 캐시 버전**: 코드 수정 배포 시 `sw.js`의 `CACHE = 'today-three-v1'` 버전 문자열을 반드시 올릴 것 (안 올리면 구버전이 계속 서빙될 수 있음 — stale-while-revalidate라 다음 방문 시 갱신되긴 함)
- [ ] **삭제 동작**: 루틴 삭제 시 routineLog의 과거 기록은 의도적으로 보존 (통계 정합성). 업무 삭제는 완전 삭제
- [ ] **import 동작**: 가져오기는 병합이 아닌 **전체 교체** — 사용자 confirm 있음
- [ ] **접근성**: 체크 버튼 aria-label, 키보드 포커스 — 기본 수준만 적용됨. 강화 여지 있음

## 6. 배포 절차

### GitHub Pages (자동)
```bash
git init && git add . && git commit -m "feat: 오늘셋 PWA v1"
git branch -M main
git remote add origin https://github.com/<ID>/today-three.git
git push -u origin main
```
저장소 Settings → Pages → Source를 **GitHub Actions**로 선택.
이후 main에 push할 때마다 `.github/workflows/deploy.yml`이 자동 배포.
(수동 방식 선호 시: Source를 Deploy from a branch / main / root로 선택해도 동작 — 모든 경로 상대경로)

### APK
1. PWABuilder: https://www.pwabuilder.com 에 배포 URL 입력 → Android 패키지 다운로드
2. 또는 Bubblewrap:
```bash
npm i -g @bubblewrap/cli
bubblewrap init --manifest https://<ID>.github.io/today-three/manifest.json
bubblewrap build
```
빌드에만 인터넷 1회 필요. 설치된 앱은 완전 오프라인 동작.

### 배포 후 검증
1. 사이트 1회 방문 → DevTools Application 탭에서 SW 등록·캐시 확인
2. 비행기 모드 → 새로고침 → 정상 로드 확인
3. 루틴 1개 + 업무 1개(목표일 어제로 설정) 추가 → 새로고침 → 팝업에 지연 표시 확인

## 7. 알려진 한계 및 개선 TODO

| 우선순위 | 항목 | 비고 |
|---|---|---|
| ~~상~~ 완료 | routineLog 보존 기간 정책 | load()에서 180일 초과분 자동 정리 (`LOG_KEEP_DAYS`) |
| ~~상~~ 완료 | 데이터 백업 리마인더 | 마지막 백업 30일 경과(또는 미백업+항목 5개 이상) 시 팝업 |
| 중 | 푸시 알림 | 현재는 앱 실행 시 팝업만. 로컬 알림은 APK(TWA)에서 제약 있어 Capacitor 전환 검토 필요 |
| 중 | 업무 정렬·검색, 카운터파트별 필터 | 업무 수 증가 대비 |
| 중 | localStorage → IndexedDB 전환 | 데이터 대용량화 시 |
| 하 | AI 에이전트(브레인덤프 구조화, if-then 생성) 온라인 옵션 복원 | 1차 버전 코드 재사용 가능. 사내망 정책 확인 필요 |
| 하 | 다크 모드, i18n | |

## 8. 보안·정보 참고

- **멀티유저 구조**: 서버·계정 없음. 접속자마다 자기 기기 localStorage에 독립 저장 →
  URL을 아는 외부인이 접속해도 빈 앱만 보이며 타인의 데이터는 절대 볼 수 없음.
  반대로 기기 간 동기화도 없음 (JSON 내보내기/가져오기로 수동 이전)
- **화면 잠금(PIN)**: 같은 기기를 쓰는 타인으로부터 보호. 기록 탭에서 설정.
  클라이언트 측 잠금이라 개발자도구로 우회 가능 — 억제책이지 암호화가 아님.
  데이터 자체 암호화가 필요해지면 WebCrypto AES-GCM + PIN 유도키 도입 검토
- 모든 데이터는 사용자 기기에만 저장되며 어떤 서버로도 전송되지 않음 (네트워크 호출 코드 없음, SW의 fetch는 자체 정적 자산 캐싱용)
- 단, localStorage는 평문 저장이므로 **대외비·개인정보가 포함된 업무명/메모 입력은 지양**하도록 사용 안내 권장
- 공용 PC 브라우저 사용 금지 안내 권장
