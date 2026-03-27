

```dataviewjs
// 1. 전체 데이터 가져오기
let pages = dv.pages('"1. 기업"')
    .where(p => p.status === "holding");

// 2. 숫자 변환 함수
function toNumber(val) {
    if (val === null || val === undefined) return 0;
    return Number(val);
}

// 3. CASH 노트 분리 (🔥 수정된 부분)
let cashPage = dv.pages('"1. 기업"')
    .where(p => 
        (p.ticker && p.ticker.toUpperCase() === "CASH") || 
        p.file.name.toUpperCase() === "CASH"
    )
    .first();

// 🔍 디버깅 (필요하면)
if (!cashPage) {
    dv.paragraph("⚠️ CASH 노트를 찾지 못했습니다.");
}

// 4. 환율 및 현금 처리
let exchangeRate = cashPage ? toNumber(cashPage.exchange_rate) : 1;

let cashKRW = cashPage ? toNumber(cashPage.krw) : 0;
let cashUSD = cashPage ? toNumber(cashPage.dollars) : 0;

let cashTotal = cashKRW + (cashUSD * exchangeRate);

// 5. 일반 종목 처리
let data = pages
    .where(p => p.file.name.toUpperCase() !== "CASH" && p.shares && p.price)
    .map(p => {
        let shares = toNumber(p.shares);
        let price = toNumber(p.price);
        let valueKRW = shares * price * exchangeRate;

        return {
            ticker: p.ticker ?? p.file.name,
            shares: shares,
            price: price,
            value: valueKRW
        };
    })
    .array();

// 6. CASH 추가
if (cashPage) {
    data.push({
        ticker: "💵 CASH",
        shares: "",
        price: "",
        value: cashTotal
    });
}

// 7. 총 자산
let totalValue = data.reduce((sum, p) => sum + p.value, 0);

// 8. 비중 계산
data = data.map(p => ({
    ...p,
    weight: totalValue > 0 ? (p.value / totalValue * 100) : 0
}));

// 9. 정렬
data.sort((a, b) => b.value - a.value);

dv.header(3,"📊 자산 포트폴리오(원화, 달러포함)");
// 10. 출력
dv.table(
    ["Ticker", "주식수", "가격(USD)", "평가금액(KRW)", "비중(%)"],
    data.map(p => [
        p.ticker,
        p.shares,
        typeof p.price === "number" ? `$${p.price.toFixed(2)}` : p.price,
        `₩${p.value.toLocaleString()}`,
        `${p.weight.toFixed(2)}%`
    ])
);
```


```dataviewjs
// 1. 전체 데이터
let pages = dv.pages('"1. 기업"')
    .where(p => p.status === "holding");

// 2. 숫자 변환
function toNumber(val) {
    if (val === null || val === undefined) return 0;
    return Number(val);
}

// 3. CASH 찾기
let cashPage = dv.pages('"1. 기업"')
    .where(p => 
        (p.ticker && p.ticker.toUpperCase() === "CASH") || 
        p.file.name.toUpperCase() === "CASH"
    )
    .first();

// 4. USD 현금만 사용
let cashUSD = cashPage ? toNumber(cashPage.dollars) : 0;

// 5. 주식 (USD 기준 그대로)
let data = pages
    .where(p => p.file.name.toUpperCase() !== "CASH" && p.shares && p.price)
    .map(p => {
        let shares = toNumber(p.shares);
        let price = toNumber(p.price);

        return {
            ticker: p.ticker ?? p.file.name,
            shares: shares,
            price: price,
            value: shares * price // 🔥 USD 그대로
        };
    })
    .array();



// 7. 총합 (USD 기준)
let totalValue = data.reduce((sum, p) => sum + p.value, 0);

// 8. 비중
data = data.map(p => ({
    ...p,
    weight: totalValue > 0 ? (p.value / totalValue * 100) : 0
}));

// 9. 정렬
data.sort((a, b) => b.value - a.value);

// 10. 출력
dv.header(3, "💵 포트폴리오 (USD 기준)");

dv.table(
    ["Ticker", "주식수", "가격(USD)", "평가금액(USD)", "비중(%)"],
    data.map(p => [
        p.ticker,
        p.shares || "",
        typeof p.price === "number" ? `$${p.price.toFixed(2)}` : "",
        `$${p.value.toLocaleString()}`,
        `${p.weight.toFixed(2)}%`
    ])
);
```