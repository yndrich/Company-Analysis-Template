---
company: SGOV
ticker: 
sector:
industry:
market_cap:
CEO:
career:
inauguration:
competitors:
tags: []
cycle: []
financials:
latest_earnings:
upcoming_earnings:
cash_asset: Y
price: 100.61
buy_min: 100
buy_max: 101
target_min:
target_max:
accounts:
  - name: ho
    shares: 176
    total_cost: 17688
    status: holding
  - name: father
    shares: 170
    total_cost: 17104.55
    status: holding
  - name: sister
    shares: 191 
    total_cost: 19188.5101
    status: holding
---

## 💡 투자이유

-  

## ⚠️ 리스크 요소

-  


## 🔄 사이클

![[SGOV 사이클.png]]

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
FROM "3. 어닝콜/NVDIA"
WHERE contains(file.name, "summary") 
SORT date(announce_date) DESC    
```


## ⚖️ 관련 법안 및 행정명령

```dataview
TABLE WITHOUT ID
  link(file.link, summary) AS 요약,
  choice(contains(file.folder, "미국_법안"), "법안", "행정명령") AS 구분,
  dateformat(default(date, file.mtime), "yyyy-MM-dd") AS 날짜,
  join(
    filter(
      default(tags, file.tags),
      (t) => contains(
        map(default(this.tags, this.file.tags), (x) => lower(string(x))),
        lower(string(t))
      )
    ),
    ", "
  ) AS 공통태그
FROM "6. 거시경제/정책/미국_법안" OR "6. 거시경제/정책/미국_행정명령"
WHERE any(
  default(tags, file.tags),
  (t) => contains(
    map(default(this.tags, this.file.tags), (x) => lower(string(x))),
    lower(string(t))
  )
)
SORT type, date DESC
```

## 📰 관련 스크랩

```dataview
TABLE WITHOUT ID
  link(file.link, summary) AS 요약,
  choice(contains(file.folder, "리포트"), "리포트", choice(contains(file.folder,"영상"),"영상","리포트")) AS 구분,
  dateformat(default(date, file.mtime), "yyyy-MM-dd") AS 날짜,
  join(
    filter(
      default(tags, file.tags),
      (t) => contains(
        map(default(this.tags, this.file.tags), (x) => lower(string(x))),
        lower(string(t))
      )
    ),
    ", "
  ) AS 공통태그
FROM "6. 거시경제/스크랩"
WHERE any(
  default(tags, file.tags),
  (t) => contains(
    map(default(this.tags, this.file.tags), (x) => lower(string(x))),
    lower(string(t))
  )
)
SORT type, date DESC
```

## 👥 로비 내역

```dataview
table without ID
link(file.link,title) AS "분류",
year+"_"+quarter+"Q" AS "시기", 
dateformat(date,"yyyy-MM-dd") AS "날짜",
lobbyist AS "로비스트",
amounts AS "금액",
		tags AS "요약"
from "4. 로비/로비내역/NVDIA"
where contains(file.name, "로비내역")
sort date desc
```

