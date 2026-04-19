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
cash_asset:
price: 206.7
buy_min: 120
buy_max: 150
target_min:
target_max:
accounts:
  - name: ho
    shares: 44
    total_cost: 14415.51
    status: holding
  - name: father
    shares: 79
    total_cost: 17138.47
    status: holding
  - name: sister
    shares: 10
    total_cost: 3159.9
    status: holding
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


```dataviewjs
const pages = dv.pages('"4. 거시경제/스크랩"');

dv.header(2, "🌍 거시경제 스크랩");

let list = [];

// 현재 파일명 (소문자로 변환)
const currentFile = dv.current().file.name.toLowerCase();

for (let page of pages) {
  let content = await dv.io.load(page.file.path);
  let lines = content.split("\n");

  for (let line of lines) {

    // 🔥 대소문자 무시 비교
    if (!line.toLowerCase().includes(currentFile)) continue;

    let match = line.match(/\^([\w-]+)/);
    let blockId = match ? match[1] : null;

    if (blockId) {

      let cleanQuote = line.replace(/\s*\^[\w-]+/, "").trim();
      let source = page.file.name.replace(/^\d{4}-\d{2}-\d{2}_/, "");

      let link = `[[${page.file.name}#^${blockId}|(${source})]]`;

      list.push(`${cleanQuote} ${link}`);
    }
  }
}

dv.list(list);
```