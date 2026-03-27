---
company: COIN
ticker: COIN
sector: Financial
industry: Financial Data & Stock Exchanges
market_cap: 43.84B
CEO: 암스트롱
career:
  - 컴퓨터과학 학사, 경제 학사, 컴퓨터공학 석사
inauguration: "2012"
competitors:
tags: []
cycle: []
financials: "[[1. COIN 분석.xlsx]]"
latest_earnings: 2026-02-12
upcoming_earnings: 2026-04-30
price: "160.66"
shares: "50"
holding: holding
---

## 💡 투자이유

-  미래 금융 종합 결제 플랫폼 가능성 ↑

## ⚠️ 리스크 요소

- 암호화폐 거래 감소
- 정치, 로비로 인한 금융체계에서의 비중 축소


## 🔄 사이클

![[COIN 사이클.png]]

1.


## 📊 실적 발표 

```dataview
TABLE WITHOUT ID
  link(file.link,ticker) AS 티커,
  dateformat(announce_date,"yyyy-MM-dd") AS 발표,
  result AS 결과,
  revenue AS 매출,
  guidance AS 가이던스,
  tags AS 메모
FROM "3. 어닝콜/COIN"
WHERE contains(file.name, "summary") 
SORT date(announce_date) DESC    
```

