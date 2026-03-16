---
company: META
ticker: META
sector: communication
industry: Internet
market_cap: 1673.61B
CEO: 마크 저커버그
career:
  - 하버드 컴퓨터공학
  - 하버드 법학박사
inauguration: "2004"
competitors:
tags: []
cycle:
  - 일상 생활에 도움이 되는 커뮤니티 제공
  - 사용자 ↑
  - 광고 매출 ↑
  - AI 기반 편의성 확대
financials: "[[1. META 분석.xlsx]]"
latest_earnings: 2026-01-28
upcoming_earnings: 2026-04-29
---

## 💡 투자이유

- 35억 명의 사용자로 AI 확대 시 쉽게 배포 가능
- 휴대폰 이후의 차세대 하드웨어 AI 안경 업계 리더
- 매출 대비 싼 가격(25.11.29 기준)

## ⚠️ 리스크 요소

- 유럽 규제로 인한 매출 하락 
- AI 기반 투자 실패로 인한 막대한 비용 손실


## 🔄 사이클

![[META 사이클.png]]

1. 일상 생활에 도움이 되는 커뮤니티 제공
2. 사용자 ↑
3. 광고 매출 ↑
4. AI 기반 편의성 확대


## 📊 실적 발표(재무)

```dataview
TABLE WITHOUT ID
  link(file.link,fiscal_period) AS 분기,
  choice(lower(result) = "beat", "⬆️", choice(lower(result) = "miss", "⬇️", "➖")) AS 결과,
  revenue AS 매출,
  revenue_yoy AS 성장률,
  grossprofit AS 총이익률,
  eps_yoy AS EPS성장률,
  tags AS 메모
FROM "3. 어닝콜/META"
WHERE contains(file.name, "summary") 
SORT date(announce_date) DESC
```

