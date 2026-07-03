# 시멘틱 감사 — 中国 EC Dashboard
> 일자: 2026-06-16
> 범위: `c:/Users/user/claude code/ec_dashboard_china/` 자기 담당 영역만
> 기준: BOH 메인 레포 `.agents/semantic/{metrics,rules,sources}.yaml` (2026-06-16 신규)
> 점검자: China Team agent

## 0. 감사 요약 (TL;DR)

- 🔴 Critical: **2건** (ROAS 배수 표기 · 환율 디폴트 불일치)
- 🟡 Warning: **3건** (시멘틱 미정의 지표 · CTR 소수자리 일관성 · 시트 파생컬럼 미정의)
- 🟢 OK: **4건** (CID 30일 윈도우 명시 · 위안→원 환율 변수화 · 중국어 원본+한글 병기 · %p 변화 표기)

전체적으로 메인 시멘틱과의 충돌은 **제한적**. 중국 도메인은 SingleOne이 아닌 CID 어트리뷰션을 쓰므로 `rules.yaml > attribution.china_cid` (15일/뷰티 30일) 룰만 핵심. CID 윈도우는 이미 코드·UI에 `归因30天 · 红猫计划`으로 명시되어 양호. 가장 큰 갭은 **ROAS/ROI 값 표기 단위(배수 vs %)** 와 **환율 디폴트 값**.

---

## 1. 발견 항목 (시멘틱 기준 점검)

| # | 항목 | 시멘틱 정의 | 현행 中国 대시보드 | 심각도 | 권고 |
|---|---|---|---|---|---|
| 1 | ROAS/ROI 표기 | `Math.round(roas*100)+'%'` 자연수% ([[roas-natural-number]]) | `F2(roi)` → `1.88` 배수 표기 (라인 1167·1175·1392·1394·2375·2391·2392) | 🔴 | `F2(roi)` → `Math.round(roi*100)+'%'` 일괄 치환. 단 중국 도메인 관례(쥐광 ROI 1.88)도 사내 보고에 정착 — 대시보드 UI는 `188%` 자연수%로 통일, 내부 메모만 배수 허용 |
| 2 | CNY→KRW 환율 디폴트 | `cny_to_krw: 195.0` (rules.yaml) | `globalFx=190` (라인 637·1029·1932) | 🔴 | 디폴트 값 `190 → 195`로 정정. UI는 사용자 편집 가능하므로 변동성 수용 OK. 보고서 산출 시 적용환율 명시 룰 ([[cny-to-krw-conversion]])은 별도 — 보고서 작성 에이전트에서 강제 |
| 3 | 매출 정의 혼선 (광고 GMV vs 전점 GMV) | metrics.yaml: `revenue_ad_attributed` (광고 어트리뷰션 한정) vs `revenue_order` (전체 주문) 명확 분리 | `광고 GMV (추정)` 카드 — `쥐광 추정 광고매출 · 실제 전체매출 아님` 명시 양호 (라인 1166). 단 시멘틱과 직접 매핑되는 ID 부재 | 🟡 | China용 시멘틱 보완 필요: `revenue_juguang_estimated` (쥐광 추정) · `revenue_cid_confirmed` (CID 확정) 2개 신규 정의. sources.yaml에 中国 시트 탭 추가 |
| 4 | CTR 소수자리 | `toFixed(1)+'%'` ([[ctr-decimal-one]]) | 혼재: `viewRate.toFixed(1)%` (1518줄·OK) vs `(sumInt/sumImp*100).toFixed(2)%` (1519줄) vs `ER.toFixed(2)+'%'` (1520줄·과다) | 🟡 | ER·인터랙션률은 1자리로 통일. ER만 도메인 관례로 2자리 허용 시 주석으로 명시 |
| 5 | 시트 파생 컬럼 정리 | RAW 시트는 원본만, 코드에서 계산 ([[raw-sheet-no-formula]]) | `글로벌EC_RAW 데이터_china.xlsx` 내부 점검 안 됨 (런타임 동기화는 Google Sheets) — `CTR(%)`, `ROAS(%)` 컬럼이 시트에 있을 가능성 | 🟡 | sources.yaml에 중국 시트(`일별raw통합`, `KOL_WM`, `KOL_CG`) 추가. 시트의 파생 컬럼은 삭제 후 코드 계산 일원화 검토 (단 현재 코드도 계산 폴백 있음 — 라인 1962~1964) |
| 6 | CID 어트리뷰션 윈도우 명시 | `rules.yaml > attribution.china_cid.beauty_window_days: 30` | ✅ `归因30天 · 红猫计划` UI 배지 (라인 2263), `30日归因` 카드 sub (라인 1175), `归因30天` 매체 SUB_MAP (라인 2273) | 🟢 | 양호. 단 sub 텍스트 표준화 권장 — 모두 `CID 어트리뷰션 30일 (红猫计划 · 뷰티)`로 통일 |
| 7 | 위안→원 환율 변수화 | `cny_to_krw: 195.0` 코드 처리 | ✅ `globalFx` 변수 + UI 입력 + 보고서 적용 (라인 637·1029·1931) | 🟢 | 디폴트 값(#2)만 정정하면 OK. 보고서 산출 시 적용환율 + 적용일 자동 footer 권장 |
| 8 | 중국어 원본 + 한국어 병기 | [[china-agency-chinese-first]] · [[korean-translation-required]] | ✅ `平均 CPUV (主KPI · 메인)`, `广告费 (광고비)`, `归因30天 (뷰티 30일 윈도우 적용)` 일관 적용 | 🟢 | 양호 |
| 9 | %p 변화 표기 | 비율 지표 변화는 절대차이 ([[percentage-point-rule]]) | ✅ `roasDelta.toFixed(0)+'%p'` (라인 2231·2279) | 🟢 | 양호 |
| 10 | 추적 브랜드 정의 | `tracked_brand` (BOH·WM·CG·BG) — qoo10.js 미러 | 中国는 **WAKEMAKE·COLORGRAM 2개만** (README) — BOH·BG 미진출 | 🟢 | 의도된 차이. metrics.yaml에 `tracked_brand_china` 별도 분기 또는 `markets:` 필드 추가 권장 |
| 11 | SingleOne 어트리뷰션 룰 적용 | 中国는 **제외 시장** (`rules.yaml > attribution.singleone.excluded_market: 중국`) | ✅ 코드에 SingleOne 참조 없음 — CID/쥐광 자체 어트리뷰션 | 🟢 | 양호 |

---

## 2. 자동 수정 가능 / 수정 안 함 분류

### 🔧 자동 수정 후보 (자기 폴더 내, 사용자 컨펌 후 진행)
- **수정 #2 (환율 디폴트 195)**: 1줄(라인 637 `value="190"`) + 1줄(라인 1029 `globalFx=190`) + 1줄(라인 1932 `globalFx=+e.target.value||190`) — 안전한 변경. 단 메인 시멘틱 환율도 변동값이므로 사용자 컨펌 권장.
- **수정 #1 (ROAS 배수 → 자연수%)**: 영향 라인 ~10곳, UI 전반 일관성. 다만 **사용자 메모리에 자연수% 룰 명시되어 있음** ([[roas-natural-number]]) — 별도 컨펌 없이 진행해도 룰 부합.

### 🛑 본 감사 단계에선 자동 수정 안 함 (사용자 결정 필요)
- 수정 #3 (China용 시멘틱 신규 정의) — BOH 메인 레포 `.agents/semantic/*.yaml`에 추가해야 하므로 자기 영역 밖. Orchestrator 통해 보고만.
- 수정 #5 (시트 파생 컬럼 정리) — Google Sheets 원본 변경은 운영 영향 큼. 사용자 컨펌 필수.

본 감사에서는 **읽기·분석만 수행하고 코드 직접 수정은 하지 않음**. 사용자가 "수정해라" 지시하면 위 #1, #2 우선 적용 가능.

---

## 3. 추가 발견 — 메인 시멘틱이 中国 시장을 명시적으로 커버하지 못하는 영역

| 누락 항목 | 메인 시멘틱 현황 | 권고 (Orchestrator 보고) |
|---|---|---|
| `revenue_cid_confirmed` / `revenue_juguang_estimated` | 시멘틱 매출 3종(`ad_attributed`/`order`/`item`)만 — 中国 핵심 구분(추정 vs 확정) 없음 | metrics.yaml에 추가 |
| `cpuv` (Cost per UV) | 시멘틱 미정의 — 中国 메인 KPI인데 정의 없음 | metrics.yaml에 추가 (`spend / external_uv`) |
| `er` (Engagement Rate) | KOL 분석 핵심인데 미정의 | metrics.yaml에 추가 |
| China 시트 매핑 | sources.yaml은 일본 시트만 (`1cAaFW...`) | sources.yaml에 中国 섹션 신설 (시트ID·gid·탭명) |

---

## 4. 결론 & 액션

**자기 폴더 영향 — 즉시 조치 가능:**
1. 환율 디폴트 `190 → 195` (rules.yaml 정렬) — 1분 작업
2. ROAS/ROI `F2(x)` → `Math.round(x*100)+'%'` 통일 — 10분 작업, 일관성 향상

**메인 레포 영향 — Orchestrator 통해 보고:**
3. 메인 시멘틱에 China 분기 추가 (`revenue_cid_confirmed`, `cpuv`, `er`)
4. sources.yaml에 中国 시트 매핑 신설

**현재 양호 — 유지:**
- CID 30일 윈도우 (红猫计划) UI/코드 모두 명시 ✅
- 위안→원 환율 변수화 ✅
- 중국어 원본 + 한글 병기 ✅
- %p 변화 표기 ✅

---

## 5. 메타데이터
- 점검 파일: `index.html` (638KB · 단일 파일 SPA)
- 점검 외 파일: `글로벌EC_RAW 데이터_china.xlsx` (오프라인 백업 · 런타임 미사용)
- 시멘틱 참조: `.agents/semantic/{metrics,rules,sources}.yaml` (BOH 메인 레포, 2026-06-16)
- 관련 메모리: `cid-attribution-window-china`, `cny-to-krw-conversion`, `china-agency-chinese-first`, `roas-natural-number`, `ctr-decimal-one`, `percentage-point-rule`, `raw-sheet-no-formula`, `korean-translation-required`
