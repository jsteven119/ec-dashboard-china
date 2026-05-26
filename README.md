# 중국 광고성과 대시보드

WAKEMAKE × COLORGRAM 샤오홍슈(샤오홍싱) 광고 + KOL 성과 통합 대시보드.

## 사용

브라우저에서 `index.html`을 열면 자동으로 Google Sheets에서 최신 데이터를 동기화합니다.

- 상단 **🔄 시트 동기화** 버튼으로 수동 갱신 가능
- 데이터 소스: Google Sheets (gviz CSV API · 공개 읽기 권한 필요)
- 시트 ID 변경은 `index.html` 내 `SHEET_ID` 상수 수정

## 동기화 시트 탭

| 탭 | 용도 |
|---|---|
| `일별raw통합` | 일별 추세 + 월별 광고비/ROI/매체 구성 자동 집계 |
| `KOL_WM` / `KOL_CG` | KOL 산점도/주력상품/팔로워구간/콘텐츠앵글 (게시일 기준 dedupe) |

## 배포

Cloudflare Pages에 GitHub 연동으로 배포. `main` 브랜치 push 시 자동 배포.
