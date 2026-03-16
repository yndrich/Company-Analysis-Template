
## 📌 기업 개요 

```dataviewjs

const now = moment();
const weekLater = moment().add(7, "days");

// 헤더에 최신매출, 최신가이던스 추가
dv.table(
  ["회사", "티커", "CEO", "시총","최신매출", "가이던스", "다음실적발표"],
  dv.pages('"1. 기업"')
    .where(p => p.ticker)
    .sort(p => {
      let mc = p.market_cap || "";
      if (mc.includes("T")) return -Number(mc.replace("T", "")) * 1e12;
      if (mc.includes("B")) return -Number(mc.replace("B", "")) * 1e9;
      if (mc.includes("M")) return -Number(mc.replace("M", "")) * 1e6;
      return -Number(mc);
    })
    .map(p => {
      // 1. 기존 로직: CEO 및 날짜 처리
      const ceo = p.file.frontmatter?.CEO || p.ceo || "N/A";
      const earningsRaw = p.file.frontmatter?.upcoming_earnings || p.upcoming_earnings;
      const earningsDate = moment(earningsRaw, ["YYYY-MM-DD", moment.ISO_8601]);
      const diffDays = earningsDate.diff(now, "days");

      let formattedDate;
      if (!earningsDate.isValid()) {
        formattedDate = "❓데이터 없음";
      } else if (diffDays <= 14 && diffDays >= 0) {
        formattedDate = `🔴 **${earningsDate.format("YYYY-MM-DD")}**`;
      } else {
        formattedDate = earningsDate.format("YYYY-MM-DD");
      }

      // 2. 추가 로직: 어닝콜 폴더에서 최신 요약 노트 찾기
      // "3. 어닝콜" 폴더 내의 파일 중, 현재 행의 티커(p.ticker)로 시작하는 파일을 찾습니다.
      // 파일명(예: GOOG_2024Q2, GOOG_2024Q1)을 내림차순 정렬하면 최신 분기가 0번째에 옵니다.
      let latestNote = dv.pages('"3. 어닝콜"')
        .where(k => k.file.name.startsWith(p.ticker + "_") && k.file.name.endsWith("_summary"))
        .sort(k => k.file.name, 'desc')[0];

      // 최신 노트가 있으면 frontmatter 값을 가져오고, 없으면 "-" 표시
      let revenue = latestNote ? (latestNote.revenue || "-") : "-";
      let guidance = latestNote ? (latestNote.guidance || "-") : "-";
      
      // (선택사항) 요약 노트로 바로 이동할 수 있게 링크를 걸고 싶다면 아래 주석 해제
      // if (latestNote) { revenue = `[${revenue}](${latestNote.file.path})` }

      return [
        p.file.link,
        p.ticker,
        ceo,
        p.market_cap,
        revenue,   
        guidance,   
        formattedDate
      ];
    })
);
```

## 📊 최근 실적(4분기)
```dataviewjs
  

// ────────────────────────────────

// 유틸

// ────────────────────────────────

function parseMarketCap(raw) {

  if (!raw) return 0;

  const s = String(raw).trim().toUpperCase().replace(/[, ]/g,"");

  if (s.endsWith("T")) return parseFloat(s) * 1e12;

  if (s.endsWith("B")) return parseFloat(s) * 1e9;

  if (s.endsWith("M")) return parseFloat(s) * 1e6;

  return Number(s) || 0;

}

  
  

function formatMarketCap(n) {

  if (!Number.isFinite(n) || n <= 0) return "";

  if (n >= 1e12) return (n / 1e12).toFixed(2).replace(/\.00$/,"") + "T";

  if (n >= 1e9)  return (n / 1e9 ).toFixed(2).replace(/\.00$/,"") + "B";

  if (n >= 1e6)  return (n / 1e6 ).toFixed(2).replace(/\.00$/,"") + "M";

  return String(n);

}

  

function norm(v) {

  if (Array.isArray(v)) return v.filter(x => x != null && String(x).trim() !== "").join(", ") || "";

  if (v == null || String(v).trim() === "") return "";

  return String(v);

}

  
  

const toTicker = v => (v ?? "").toString().trim().toUpperCase();

// ────────────────────────────────

// ① "1. 기업" 폴더에서 (티커 → 시총/회사명) 맵 만들기

// ────────────────────────────────

const companyNotes = dv.pages('"1. 기업"').array();

const mcapMap = new Map();

for (const p of companyNotes) {

  // 회사 노트에서 티커 후보들(각자 쓰는 키가 다를 수 있음)

  const ticker =

    toTicker(p.ticker ?? p.symbol ?? p.company?.ticker ?? p.stock?.ticker ?? "");

  // market_cap 후보들

  const rawCap =

    p.market_cap ?? p.company?.market_cap ?? p.marketCap ?? p.company?.marketCap ?? null;

  const mcap_num = parseMarketCap(rawCap);

  // 회사명(표시용)

  const name = p.company?.name ?? p.company ?? p.name ?? p.file.name;

  if (ticker) mcapMap.set(ticker, { mcap_num, name });

}// ────────────────────────────────

/* ② 어닝콜 노트 읽기 & 티커별 묶기 */

// ────────────────────────────────

const pages = dv.pages('"3. 어닝콜"')

  .where(p => p.announce_date && p.file.name.endsWith("summary"))

  .array();

const groups = {};

for (const p of pages) {

  const key = toTicker(p.ticker || "UNKNOWN");

  (groups[key] ??= []).push(p);

}// ────────────────────────────────

// ③ 그룹 정리(최신 4개) + 회사 노트 시총 결합

// ────────────────────────────────

let groupArray = Object.entries(groups).map(([ticker, rows]) => {

  rows = rows

    .sort((a, b) => dv.date(b.announce_date) - dv.date(a.announce_date))

    .slice(0, 4);

  const companyInfo = mcapMap.get(ticker) || { mcap_num: 0, name: "" };

  const mcap_num = companyInfo.mcap_num;

  const companyName =

    companyInfo.name ||

    (rows[0]?.company?.name ?? rows[0]?.company ?? "");

  return { ticker, rows, mcap_num, companyName };

});

// ────────────────────────────────

// ④ 시총 내림차순 정렬(0은 뒤로 밀기)

// ────────────────────────────────

groupArray.sort((a, b) => {

  const am = Number.isFinite(a.mcap_num) ? a.mcap_num : -1;

  const bm = Number.isFinite(b.mcap_num) ? b.mcap_num : -1;

  return bm - am;

});

// ────────────────────────────────

// ⑤ 출력 (이모티콘 및 '-' 적용)

// ────────────────────────────────

for (const g of groupArray) {

  const { ticker, rows, mcap_num, companyName } = g;

  if (!rows.length) continue;

  const mcapLabel = formatMarketCap(mcap_num);

  const titleRight = mcapLabel ? ` | 시총: ${mcapLabel}` : "";

  const titleLeft  = companyName ? `${companyName}` : ticker;

  dv.header(3, `🏢 ${titleLeft}`);

  dv.table(

    ["분기", "결과", "매출","성장률","총이익률","EPS성장률", "메모"],

    rows.map(r => {

      const fiscalLink = r.file.link.withDisplay(r.fiscal_period || "");

      // ⭐ 수정된 부분: r.result 값에 따라 이모티콘 및 '-' 추가

      let resultIcon = "";

      const rawResult = (r.result || "").toString().toLowerCase().trim();

      if (rawResult === "beat") {

        resultIcon = "⬆️ ";

      } else if (rawResult === "miss") {

        resultIcon = "⬇️ ";

      } else if (rawResult === "") {

          // r.result 값이 아예 비어있을 경우 '-'로 표시

        resultIcon = "-";

      }

      // 결과값이 비어있고 이모티콘이 '-'일 경우, '-'만 표시하도록 처리

      const result   = (resultIcon === "-" && rawResult === "") ? resultIcon : resultIcon + norm(r.result);

      // ⭐ 수정 끝

      const revenue  = norm(r.revenue);

      const revenue_yoy  = norm(r.revenue_yoy);

      const grossprofit  = norm(r.grossprofit);

      const eps_yoy  = norm(r.eps_yoy);

      const memo = Array.isArray(r.tags)

        ? r.tags.filter(x => x != null && String(x).trim() !== "").map(x => "· " + x).join("<br>")

        : (r.tags ? "· " + String(r.tags) : "");

      return [fiscalLink, result, revenue,revenue_yoy,grossprofit,eps_yoy, memo];

    })

  );

}
```

##  💬 어닝콜 언급
```dataviewjs

```
```dataviewjs
/***** 설정 *****/
const ROOT = "3. 어닝콜";
const COMPANY_ROOT = "1. 기업";
const FILE_NAME_RE = /^(?<company>[^_]+)_(?<period>\d{4}Q[1-4])_transcript/i;

/***** util *****/
const periodKey = s=>{
  const m=/(\d{4})Q([1-4])/.exec(s||"");
  return m?parseInt(m[1])*10+parseInt(m[2]):-1;
};

/***** 기업 데이터 *****/
const companies=dv.pages(`"${COMPANY_ROOT}"`);

function parseMarketCap(v){

  if(!v) return 0;

  const s=String(v).trim();
  const m=s.match(/([\d.,]+)\s*([TMB])/i);

  if(!m) return Number(s.replace(/,/g,""))||0;

  const num=parseFloat(m[1].replace(/,/g,""));
  const unit=m[2].toUpperCase();

  if(unit==="T") return num*1e12;
  if(unit==="B") return num*1e9;
  if(unit==="M") return num*1e6;

  return 0;
}

function getCompanyMarketCap(company){

  try{
    const match=companies
      .where(p=>String(p.ticker||"").toUpperCase()===company.toUpperCase())
      .first();

    if(!match) return 0;

    return parseMarketCap(match.market_cap);

  }catch{
    return 0;
  }
}

/***** transcript load *****/
const pages=dv.pages(`"${ROOT}"`)
.where(p=>/_transcript/i.test(p.file.name))
.where(p=>FILE_NAME_RE.test(p.file.name));

const rows=[];

for(const p of pages){

  const file=app.vault.getAbstractFileByPath(p.file.path);
  const txt=await app.vault.cachedRead(file);

  const fnm=p.file.name.match(FILE_NAME_RE);
  const company=fnm?.groups?.company??"";
  const periodFromName=fnm?.groups?.period??"";

  const re=
  />\s*\[!(?<type>[^\]]+)\]\s*(?<name>.+?)\s*\|\s*(?<period>.+?)\n(?<body>(?:>.*\n?)+).*?\^(?<bid>[A-Za-z0-9\-]+)/gi;

  for(const m of txt.matchAll(re)){

    const type=m.groups.type.trim().toLowerCase();
    const speaker=m.groups.name.trim();
    const period=(m.groups.period??periodFromName).trim();
    const rawQuote=(m.groups.body||"").replace(/^>\s?/gm,"").trim();
    const bid=m.groups.bid;

    const topics=[...rawQuote.matchAll(/#([A-Za-z0-9_-]+)/g)].map(x=>x[1]);

    const displayQuote=rawQuote
      .replace(/\s*#([A-Za-z0-9_-]+)/g,"")
      .replace(/\s{2,}/g," ")
      .trim();

    rows.push({
      company,
      type,
      speaker,
      period,
      periodKey:periodKey(period),
      quote:displayQuote,
      topics,
      link:`${p.file.path}#^${bid}`
    });
  }
}

/***** 필터 목록 생성 *****/
const topicSet=new Set();
const companySet=new Set();

rows.forEach(r=>{
  (r.topics||[]).forEach(t=>topicSet.add(t));
  companySet.add(r.company);
});

const topics=[...topicSet].sort();
const companiesList=[...companySet].sort();

/***** 필터 상태 *****/
let activeTopic="";
let activeCompany="";
let activePeriod="all";

/***** company grouping *****/
const groups=new Map();

for(const r of rows){

  if(!groups.has(r.company)){
    groups.set(r.company,{
      items:[],
      marketCap:getCompanyMarketCap(r.company)
    });
  }

  groups.get(r.company).items.push(r);
}

/***** 시총 정렬 *****/
const sorted=
Array.from(groups.entries())
.sort((a,b)=>b[1].marketCap-a[1].marketCap);

/***** table renderer *****/
function createTable(data){

  const table=document.createElement("table");
  table.style.width="100%";

  const head=document.createElement("tr");

  ["분기","유형","화자","요약"].forEach(h=>{
    const th=document.createElement("th");
    th.innerText=h;
    head.appendChild(th);
  });

  table.appendChild(head);

  for(const r of data){

    const tr=document.createElement("tr");

    const td1=document.createElement("td");
    td1.innerText=r.period;

    const td2=document.createElement("td");
    td2.innerText=r.type;

    const td3=document.createElement("td");
    td3.innerText=r.speaker;

    const td4=document.createElement("td");

    const target=r.link;

    const a=document.createElement("a");
    a.href=target;
    a.className="internal-link";
    a.innerText=r.quote.slice(0,200);

    a.addEventListener("click", async (e)=>{
      e.preventDefault();
      await app.workspace.openLinkText(
        target,
        dv.current().file.path,
        false
      );
    });

    td4.appendChild(a);

    tr.appendChild(td1);
    tr.appendChild(td2);
    tr.appendChild(td3);
    tr.appendChild(td4);

    table.appendChild(tr);
  }

  return table;
}

/***** 전역 필터 적용 *****/
function applyGlobalFilters(data){

  let filtered=[...data];

  if(activeTopic)
    filtered=filtered.filter(x=>x.topics.includes(activeTopic));

  if(activePeriod==="4Q"){

    const maxKey=Math.max(...filtered.map(x=>x.periodKey).filter(x=>x>=0),-1);

    if(maxKey>=0)
      filtered=filtered.filter(x=>x.periodKey>=maxKey-3);
  }

  if(activePeriod==="2Y"){

    const years=filtered
      .map(x=>parseInt(x.period))
      .filter(x=>!isNaN(x));

    const currentYear=Math.max(...years);

    filtered=filtered.filter(x=>{
      const y=parseInt(x.period);
      return !isNaN(y)&&y>=currentYear-1;
    });
  }

  return filtered;
}

/***** company render *****/
function renderCompany(company,data){

  if(activeCompany && company!==activeCompany) return;

  const globallyFiltered=applyGlobalFilters(data)
  .sort((a,b)=>b.periodKey-a.periodKey);

  if(globallyFiltered.length===0) return;

  const wrapper=document.createElement("details");
  wrapper.style.marginBottom="16px";
  wrapper.open=true;

  const summary=document.createElement("summary");
  summary.innerText=`🏢 ${company}`;
  summary.style.cursor="pointer";
  summary.style.fontWeight="600";

  wrapper.appendChild(summary);

  const content=document.createElement("div");
  content.style.marginTop="8px";

  const types=[...new Set(globallyFiltered.map(x=>x.type))].sort();

  const tabBox=document.createElement("div");
  tabBox.style.marginBottom="8px";

  const tableBox=document.createElement("div");

  function render(type){

    tableBox.innerHTML="";

    let filtered=type
      ?globallyFiltered.filter(x=>x.type===type)
      :globallyFiltered;

    filtered.sort((a,b)=>b.periodKey-a.periodKey);

    tableBox.appendChild(createTable(filtered));
  }

  const makeBtn=(label,type)=>{

    const btn=document.createElement("button");
    btn.textContent=label;
    btn.style.marginRight="6px";

    btn.onclick=()=>render(type);

    tabBox.appendChild(btn);
  };

  makeBtn("ALL","");

  types.forEach(t=>makeBtn(t.toUpperCase(),t));

  content.appendChild(tabBox);
  content.appendChild(tableBox);

  wrapper.appendChild(content);

  dv.container.appendChild(wrapper);

  render("");
}

/***** dropdown filter *****/
function createSelectFilter(title,list,currentValue,setFunc){

  const bar=document.createElement("div");
  bar.style.marginBottom="10px";

  const label=document.createElement("b");
  label.innerText=title+" ";
  bar.appendChild(label);

  const select=document.createElement("select");

  const opt=document.createElement("option");
  opt.value="";
  opt.text="ALL";
  select.appendChild(opt);

  list.forEach(v=>{
    const o=document.createElement("option");
    o.value=v;
    o.text=v;
    select.appendChild(o);
  });

  select.value=currentValue||"";

  select.onchange=e=>{
    setFunc(e.target.value);
    renderAll();
  }

  bar.appendChild(select);
  dv.container.appendChild(bar);
}

/***** period buttons *****/
function createPeriodBar(){

  const bar=document.createElement("div");
  bar.style.marginBottom="14px";

  const label=document.createElement("b");
  label.innerText="Period ";
  bar.appendChild(label);

  const makeBtn=(text,val)=>{

    const btn=document.createElement("button");
    btn.textContent=text;
    btn.style.marginRight="6px";

    btn.onclick=()=>{
      activePeriod=val;
      renderAll();
    };

    bar.appendChild(btn);
  };

  makeBtn("ALL","all");
  makeBtn("최근 4Q","4Q");
  makeBtn("최근 2Y","2Y");

  dv.container.appendChild(bar);
}

/***** 전체 렌더 *****/
function renderAll(){

  dv.container.innerHTML="";

  createSelectFilter("Topic",topics,activeTopic,v=>activeTopic=v);
  createSelectFilter("Company",companiesList,activeCompany,v=>activeCompany=v);
  createPeriodBar();

  for(const [company,g] of sorted){
    renderCompany(company,g.items);
  }
}

/***** 초기 실행 *****/
renderAll();
```

## 🏷 CEO 이력

```dataview
TABLE WITHOUT ID
  ticker AS 티커,
  CEO AS CEO, 
  career AS 이력,
  inauguration AS 취임연도
FROM "1. 기업"
WHERE tags
SORT market_cap DESC
```

## 🔄 사이클 & 태그 & 경쟁사 

```dataview 
TABLE WITHOUT ID
  ticker AS 티커,
  cycle AS "사이클",
  tags AS "태그",
  competitor AS "경쟁사"
FROM "1. 기업"
WHERE cycle
SORT market_cap DESC
```




