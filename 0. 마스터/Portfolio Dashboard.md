

```dataviewjs
// 🔥 0. 데이터
const pages = dv.pages('"1. 기업"');

function safeAccounts(p) {
  if (!p || !Array.isArray(p.accounts)) return [];
  return p.accounts;
}

function norm(v) {
  return String(v ?? "").trim();
}

// 🔥 계좌 목록
const accountSet = new Set();
pages.forEach(p => {
  safeAccounts(p).forEach(a => {
    const name = norm(a.name);
    if (name) accountSet.add(name);
  });
});
const accountList = Array.from(accountSet);

// 🔥 상태
let selected = accountList[0] ?? "ho";

// 🔥 UI
const controls = document.createElement("div");
const output = document.createElement("div");

controls.style.marginBottom = "12px";
dv.container.appendChild(controls);
dv.container.appendChild(output);

// 🔥 콤보박스
const select = document.createElement("select");

accountList.forEach(name => {
  const opt = document.createElement("option");
  opt.value = name;
  opt.text = name;
  select.appendChild(opt);
});

select.value = selected;
select.onchange = e => {
  selected = norm(e.target.value);
  render();
};

controls.appendChild(select);


// 🔥 렌더
function render() {

  output.innerHTML = "";

  // 🔥 CASH
  const cashPage = pages.where(p => p.file.name == "CASH").first();
  const rate = Number(cashPage?.exchange_rate ?? 1300);
  const targetCashPct = Number(cashPage?.target_cash_pct ?? 10);

  const cashAcc = safeAccounts(cashPage)
    .find(a => norm(a.name) === selected) || {};

  const cashUSD = Number(cashAcc.dollars ?? 0);
  const cashKRW_raw = Number(cashAcc.krw ?? 0);
  const cashKRW = cashKRW_raw + Math.floor(cashUSD * rate);

  // 🔥 종목
  const stocks = pages
    .where(p => p.file.name != "CASH")
    .filter(p => {
      const acc = safeAccounts(p).find(a => norm(a.name) === selected);
      return acc &&
        norm(acc.status) === "holding" &&
        Number(acc.shares ?? 0) > 0;
    })
    .map(p => {

      const acc = safeAccounts(p).find(a => norm(a.name) === selected);

      const shares = Number(acc?.shares ?? 0);
      const price = Number(p.price ?? 0);

      // 🔥 핵심 변경: total_cost = USD 기준
      const totalCostUSD = Number(acc?.total_cost ?? 0);

      // 🔥 환율 적용 (계산 시점)
      const totalCostKRW = totalCostUSD * rate;

      const buyMin = p.buy_min ? Number(p.buy_min) : null;
      const buyMax = p.buy_max ? Number(p.buy_max) : null;
      const targetMin = p.target_min ? Number(p.target_min) : null;
      const targetMax = p.target_max ? Number(p.target_max) : null;

      const isCashAsset =
        String(p.cash_asset ?? "")
          .replace(/[“”‘’]/g, "")
          .trim()
          .toUpperCase() === "Y";

      const valueUSD = price * shares;
      const valueKRW = Math.floor(valueUSD * rate);

      // 🔥 수익률 (환율 자동 상쇄)
      const returnPct = totalCostKRW > 0
        ? (valueKRW / totalCostKRW - 1) * 100
        : 0;

      let strategy = "🟡 관찰";

      if (buyMin !== null && price < buyMin) {
        strategy = "🚀 적극매수";
      }
      else if (buyMax !== null && price <= buyMax) {
        strategy = "🟩 분할매수";
      }
      else if (targetMin !== null && price <= targetMin) {
        strategy = "🟡 보유";
      }
      else if (targetMax !== null && price <= targetMax) {
        strategy = "🟠 일부매도";
      }
      else {
        if (returnPct > 20) strategy = "🟠 이익실현 고려";
        else strategy = "🟡 관찰";
      }

      return {
        name: p.file.link,
        ticker: p.ticker ?? p.file.name,
        price,
        shares,
        valueKRW,
        totalCostUSD,   // ✅ USD 원가
        totalCostKRW,   // ✅ 환산 원가
        returnPct,
        strategy,
        isCashAsset,
        buyMin,
        buyMax
      };
    })
    .array();

  // 🔥 자산 계산
  const cashLikeKRW = stocks
    .filter(p => p.isCashAsset)
    .reduce((s, p) => s + p.valueKRW, 0);

  const stockOnlyKRW = stocks
    .filter(p => !p.isCashAsset)
    .reduce((s, p) => s + p.valueKRW, 0);

  const totalKRW = cashKRW + cashLikeKRW + stockOnlyKRW;
  const totalCashKRW = cashKRW + cashLikeKRW;

  const currentCashPct = totalKRW > 0 ? (totalCashKRW / totalKRW) * 100 : 0;
  const cashGap = currentCashPct - targetCashPct;

  const cashWeight = currentCashPct;

  // 🔥 전체 투자금 (USD 기준 → 환율 적용)
  const totalCostAllUSD = stocks.reduce((sum, p) => sum + p.totalCostUSD, 0);
  const totalCostAllKRW = totalCostAllUSD * rate;

  // 🔥 총 수익률
  const totalReturnPct = totalCostAllKRW > 0
    ? (totalKRW / totalCostAllKRW - 1) * 100
    : 0;

  // 🔥 비중
  stocks.forEach(p => {
    p.weight = totalKRW > 0 ? (p.valueKRW / totalKRW) * 100 : 0;
  });

  // 🔥 Top Action
  let topAction = null;

  if (currentCashPct < targetCashPct) {
    const sellCandidates = stocks
      .filter(p => !p.isCashAsset &&
        (p.strategy.includes("과열") || p.strategy.includes("이익")))
      .sort((a, b) => b.weight - a.weight);

    if (sellCandidates.length > 0) {
      topAction = sellCandidates[0];
    }

  } else {
    const buyCandidates = stocks
      .filter(p => !p.isCashAsset && p.strategy.includes("매수"))
      .sort((a, b) => a.weight - b.weight);

    if (buyCandidates.length > 0) {
      topAction = buyCandidates[0];
    }
  }

  let topActionText = "없음";

  if (topAction) {
    const diffKRW = Math.abs(cashGap / 100 * totalKRW);
    const sharesToTrade = Math.round(diffKRW / (topAction.price * rate));

    const displayName = topAction.ticker;

    topActionText =
      currentCashPct < targetCashPct
        ? `🔴 ${displayName} -${sharesToTrade}주 매도`
        : `🟢 ${displayName} +${sharesToTrade}주 매수`;
  }

  // 🔥 현금 비중 색상
  const cashGapHTML =
    cashGap > 0
      ? `<span style="color:red">+${cashGap.toFixed(1)}%</span>`
      : `<span style="color:blue">${cashGap.toFixed(1)}%</span>`;

  // 🔥 수익률 색상
  const totalReturnHTML =
    totalReturnPct >= 0
      ? `<span style="color:red">+${totalReturnPct.toFixed(1)}%</span>`
      : `<span style="color:blue">${totalReturnPct.toFixed(1)}%</span>`;

  // 🔥 요약
  const summary = document.createElement("div");

  summary.innerHTML = `
  <p>
  💰 총 자산: ${totalKRW.toLocaleString()}원<br>
  📈 총 수익률: ${totalReturnHTML}<br><br>

  💵 현금: ${totalCashKRW.toLocaleString()}원 (${cashWeight.toFixed(1)}%)<br>
  🎯 목표 현금비중: ${targetCashPct}% (${cashGapHTML})<br><br>

  👉 Top Action: ${topActionText}
  </p>
  `;

  output.appendChild(summary);

  // 🔥 테이블
  const tableHost = document.createElement("div");
  output.appendChild(tableHost);

  const old = dv.container;
  dv.container = tableHost;

  const rows = stocks
    .filter(p => !p.isCashAsset)
    .sort((a, b) => b.weight - a.weight)
    .map(p => {

      const returnPctHTML = p.returnPct >= 0
        ? `<span style="color:red">+${p.returnPct.toFixed(1)}%</span>`
        : `<span style="color:blue">${p.returnPct.toFixed(1)}%</span>`;

      return [
        p.name,
        `$${p.price}`,
        (p.buyMin !== null && p.buyMax !== null)
          ? `${p.buyMin} ~ ${p.buyMax}`
          : "-",
        p.strategy,
        p.weight.toFixed(1) + "%",
        returnPctHTML
      ];
    });

  rows.push([
    "💵 현금",
    "-",
    "-",
    "🟡 현금",
    cashWeight.toFixed(1) + "%",
    "-"
  ]);

  dv.table(
    ["자산", "현재가", "매수 구간", "전략", "비중", "수익률"],
    rows
  );

  dv.container = old;
}

// 🔥 실행
render();
```



