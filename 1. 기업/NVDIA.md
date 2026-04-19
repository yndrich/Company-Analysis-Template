---
company: NVDIA
ticker: NVDA
sector: Technology
industry: Semiconductor
market_cap: 4505.53B
CEO: 젠슨황
career:
  - 전기공학(학사,석사), 엔비디아 공동 창업자 및 CEO
inauguration: "1993"
competitors:
tags:
  - AI
  - 반도체
  - 자율주행
  - 로봇
cycle:
  - 독보적인 AI 인프라 제공
  - 사용자 ↑
  - 고품질 데이터 수집
  - AI 인프라 고도화
financials: "[[1. NVDA 분석.xlsx]]"
latest_earnings: 2026-02-25
upcoming_earnings: 2026-05-20
cash_asset:
price: 201.68
buy_min: 145
buy_max: 165
target_min:
target_max:
accounts:
  - name: ho
    shares: 252
    total_cost: 28460.75
    status: holding
  - name: father
    shares: 3
    total_cost: 530.31
    status: holding
  - name: sister
    shares: 1
    total_cost: 137.24
    status: holding
---

## 💡 투자이유

-  AI 필수 회사(반도체, AI인프라(학습, 추론))


## ⚠️ 리스크 요소

-  정부 수출 규제
-  신제품 출시 지연(공급부족)
-  AI 수요 감소
-  압도적인 경쟁자 출현 


## 🔄 사이클

![[NVDA 사이클.png]]

1. 독보적인 AI 인프라 제공
2. 사용자 ↑
3. 고품질 데이터 수집
4. AI 인프라 고도화
5. 사용 분야 확대 (로봇 등)

## 📊 실적 발표 

```dataview
TABLE WITHOUT ID
  link(file.link,fiscal_period) AS 분기,
  choice(lower(result) = "beat", "⬆️", choice(lower(result) = "miss", "⬇️", "➖")) AS 결과,
  revenue AS 매출,
  revenue_yoy AS 성장률,
  grossprofit AS 총이익률,
  eps_yoy AS EPS성장률,
  tags AS 메모
FROM "3. 어닝콜/NVDIA"
WHERE contains(file.name, "summary") 
SORT date(announce_date) DESC
```

## 💬 어닝콜 
```dataviewjs
/***** 현재 기업 노트 기반 경로 *****/
/***** 현재 기업 노트 기반 경로 *****/
const ROOT = "3. 어닝콜";
const COMPANY = (dv.current().file.name || "").trim();
const TARGET = `${ROOT}/${COMPANY}`;

const FILE_NAME_RE = /^(?<company>[^_]+)_(?<period>\d{4}Q[1-4])_transcript/i;

/***** util *****/
const periodKey = s=>{
  const m=/(\d{4})Q([1-4])/.exec(String(s||""));
  return m?parseInt(m[1])*10+parseInt(m[2]):-1;
};

/***** callout parser *****/
function extractBlocks(text){

  const lines=text.split(/\r?\n/);
  const blocks=[];

  for(let i=0;i<lines.length;i++){

    const head=lines[i].match(/^>\s*\[!([^\]]+)\]\s*(.+?)\s*\|/i);
    if(!head) continue;

    const type=head[1].toLowerCase();
    const speaker=head[2].trim();

    const buf=[lines[i]];
    let j=i+1;

    while(j<lines.length && /^>\s?/.test(lines[j])){
      buf.push(lines[j]);
      j++;
    }

    let bid=null;

    for(const L of buf){
      const m=L.match(/\^([A-Za-z0-9\-_]+)/);
      if(m){bid=m[1];break;}
    }

    if(!bid && j<lines.length){
      const m=lines[j].match(/^\^([A-Za-z0-9\-_]+)/);
      if(m){bid=m[1];j++;}
    }

    const body=buf
      .slice(1)
      .join("\n")
      .replace(/^>\s?/gm,"")
      .trim();

    blocks.push({
      type,
      speaker,
      body,
      bid
    });

    i=j-1;
  }

  return blocks;
}

/***** transcript 파일 수집 *****/
const pagesRaw=dv.pages(`"${TARGET}"`)
.where(p=>/_transcript/i.test(p.file.name))
.where(p=>FILE_NAME_RE.test(p.file.name));

/* 파일 중복 제거 */
const pages=[...new Map(
  pagesRaw.map(p=>[p.file.path,p])
).values()];

/***** rows 생성 *****/
const rows=[];
const seenRows=new Set();

for(const p of pages){

  const file=app.vault.getAbstractFileByPath(p.file.path);
  if(!file) continue;

  const txt=await app.vault.cachedRead(file);

  const fnm=p.file.name.match(FILE_NAME_RE);
  const periodFromName=fnm?.groups?.period??"";

  const blocks=extractBlocks(txt);

  for(const b of blocks){

    if(!b.bid) continue;

    const key=`${p.file.path}_${b.bid}`;

    if(seenRows.has(key)) continue;
    seenRows.add(key);

    const cleanQuote=b.body
      .replace(/#([A-Za-z0-9_-]+)/g,"")
      .replace(/\s{2,}/g," ")
      .trim();

    rows.push({
      type:b.type,
      speaker:b.speaker,
      period:periodFromName,
      periodKey:periodKey(periodFromName),
      quote:cleanQuote,
      path:p.file.path,
      block:b.bid
    });
  }
}

/***** 필터 상태 *****/
let activeType="";
let activePeriod="all";

/***** 유형 목록 *****/
const types=[...new Set(rows.map(r=>r.type))].sort();

/***** 기간 필터 *****/
function applyPeriodFilter(data){

  let filtered=[...data];

  if(activePeriod==="recentQ"){

    const maxKey=Math.max(...filtered.map(x=>x.periodKey));
    filtered=filtered.filter(x=>x.periodKey===maxKey);
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

    const target=`${r.path}#^${r.block}`;

    const a=document.createElement("a");
    a.href=target;
    a.className="internal-link";
    a.setAttribute("data-href",target);
    a.textContent=r.quote.slice(0,220);

    a.addEventListener("click",async(e)=>{
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

/***** render *****/
function render(){

  dv.container.innerHTML="";

  /* Type filter */
  const typeBar=document.createElement("div");
  typeBar.style.marginBottom="10px";

  const typeLabel=document.createElement("b");
  typeLabel.innerText="Type ";
  typeBar.appendChild(typeLabel);

  const typeSelect=document.createElement("select");

  const opt=document.createElement("option");
  opt.value="";
  opt.text="ALL";
  typeSelect.appendChild(opt);

  types.forEach(t=>{
    const o=document.createElement("option");
    o.value=t;
    o.text=t.toUpperCase();
    typeSelect.appendChild(o);
  });

  typeSelect.value=activeType;

  typeSelect.onchange=e=>{
    activeType=e.target.value;
    render();
  };

  typeBar.appendChild(typeSelect);
  dv.container.appendChild(typeBar);

  /* Period filter */
  const periodBar=document.createElement("div");
  periodBar.style.marginBottom="12px";

  const label=document.createElement("b");
  label.innerText="Period ";
  periodBar.appendChild(label);

  const makeBtn=(txt,val)=>{

    const btn=document.createElement("button");
    btn.textContent=txt;
    btn.style.marginRight="6px";

    btn.onclick=()=>{
      activePeriod=val;
      render();
    };

    periodBar.appendChild(btn);
  };

  makeBtn("ALL","all");
  makeBtn("최근 분기","recentQ");
  makeBtn("최근 2년","2Y");

  dv.container.appendChild(periodBar);

  /* 데이터 필터링 */
  let filtered=activeType
    ? rows.filter(r=>r.type===activeType)
    : rows;

  filtered=applyPeriodFilter(filtered);

  filtered.sort((a,b)=>b.periodKey-a.periodKey);

  dv.container.appendChild(createTable(filtered));
}

/***** 실행 *****/
render();
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