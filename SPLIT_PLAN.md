# 중국 EC 대시보드 코드 분할 계획 (SPLIT_PLAN)

작성: 2026-07-02 · china-team 리뷰 액션 ⑤

## 현황

- 단일 파일 `index.html` = 약 720KB, 4,700+ 줄
- Cloudflare Pages 자동 배포 (main push → 정적 파일 그대로)
- 배치 3 이후 파일 크기 이전 대비 +50%. 향후 배치 계속 진행 시 점점 부담

## 왜 분할이 필요한가

1. **초기 로드 최적화** — 라이브 탭 안 볼 사용자도 라이브 코드 전량 파싱
2. **유지보수 격리** — 라이브/Tmall 각 도메인 독립 편집 시 diff 노이즈 감소
3. **캐시 효율** — Cloudflare 정적 자산 캐시 세분화. index.html만 변경 시 live.js 재다운로드 없음
4. **멀티 인스턴스 충돌 완화** — 파일별 편집 오너십 명확 (현재는 모든 편집이 index.html 충돌)

## 핵심 장애물

**전역 상태 script scope 격리**:
- `const DATA = {...}` (line 1069) — 초기 임베드 데이터, 이후 sync가 덮어씀
- `const KOL = {...}` — KOL 정규화 결과
- `let brand, tab, charts, globalFx, globalDateFrom, globalDateTo` 등 — script scope
- `Chart.js`, `Papa.parse` — 전역 라이브러리, 이미 window scope OK

`<script>` 태그를 여러 개로 나누면 각각 별도 script scope → 다른 파일에서 접근 불가.

## 해결 방안 (2단계)

### Phase 1: 전역 상태를 window에 노출 (회귀 위험 낮음)

```js
// index.html main script 초반부
window.STATE = {
  DATA, KOL,
  get brand(){return brand;}, set brand(v){brand=v;},
  get tab(){return tab;}, set tab(v){tab=v;},
  get globalFx(){return globalFx;}, set globalFx(v){globalFx=v;},
  get globalDateFrom(){return globalDateFrom;}, set globalDateFrom(v){globalDateFrom=v;},
  get globalDateTo(){return globalDateTo;}, set globalDateTo(v){globalDateTo=v;},
  charts
};
```

또는 더 단순히 `let brand`를 `var brand`로 (변수 scope 확장). 단 var는 hoisting 이슈 있음.

### Phase 2: 분할 대상 함수를 별도 파일로 이동

**우선순위 1: `js/live.js`** (배치 3 신설 · 완전 격리)
- `_pickLiveField`, `_liveBrand`, `processLiveData`
- `_liveRows`, `renderLive`, `liveCoverageBadge`, `liveKpis`
- `chLiveTrend`, `renderLiveBest`, `renderLiveTable`
- 예상: 250~300줄 · 약 15KB

**우선순위 2: `js/tmall.js`** (배치 2 · Tmall 전용)
- `_tmallCategoriesForBrand`, `_tmallPlatformGmv` (KPI용 helper 는 유지)
- `renderTmall`, `tmallCoverageBadge`, `tmallKpis`
- `renderTmallInflow`, `renderTmallSalesTop3`, `renderTmallBest`, `renderTmallWorst`
- `renderTmallROITable`, `renderTmallChannelDefRef`
- `_saleReason`, `_notSellReason`, `_classifyTmallChannel`
- `processTmallData`, `_lineNameToCategory` 등 정규화 helpers
- 예상: 500~600줄 · 약 25KB

**우선순위 3: `js/kol.js`** (배치 0 · KOL/KOC 앵글 인사이트)
- `renderKol`, `renderKolBest`, `renderCorrAnalysis`
- `chScatter`, `chCType`, `chTier`, `chAngle`, `chQuadrant`
- `renderAngleProductHeatmap`, `renderAngleActions`, `renderBrandFindings`
- 예상: 900~1000줄 · 약 40KB

### 로드 순서

```html
<!-- Chart.js CDN + PapaParse CDN -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<!-- 상태 + 코어 helpers (index.html 인라인 유지) -->
<script>const STATE = {...}; function normalizeByday(){...}; ...</script>
<!-- 도메인별 -->
<script src="js/tmall.js" defer></script>
<script src="js/live.js" defer></script>
<script src="js/kol.js" defer></script>
<!-- 부트스트랩 (index.html 인라인) -->
<script>syncSheets(); document.getElementById('tabs').addEventListener('click', ...);</script>
```

## 이번 세션 완료 (2026-07-02)

- **분할 계획서** 이 문서 작성 완료
- **잘라내기 표식** 삽입: index.html 내 도메인별 코드 앞뒤에 마커
  - `// ============ [SPLIT: js/live.js START] ============`
  - `// ============ [SPLIT: js/live.js END] ============`
  - `// ============ [SPLIT: js/tmall.js START] ============`
  - `// ============ [SPLIT: js/tmall.js END] ============`
- 실제 파일 분리는 **후속 세션**에서 (Phase 1 → Phase 2)

## 위험 관리

- **회귀 위험 高**: 전역 상태 참조 미묘한 순서 의존 있음 (예: syncSheets → render → DATA 참조)
- **테스트 순서**: (1) STATE 노출만 하고 배포 → 회귀 없는지 확인 (2) 첫 파일 분리 → 확인 (3) 나머지 분리
- **롤백**: 각 phase 별도 커밋 → 문제 시 revert 용이
- **폴백 없음**: 분리 후 파일 로드 실패 시 대시보드 완전 다운 → 반드시 CDN 실패/네트워크 오류 케이스 확인 필요
