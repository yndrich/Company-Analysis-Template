---
company: PLTR
ticker: PLTR
sector: Technology
industry: Software
market_cap: 323.91B
CEO: 알렉스카프
career:
  - 팔란티어CEO
inauguration: "2003"
competitors:
tags: []
cycle:
  - 온체인 데이터 분석 플랫폼 제공
  - 사용자 ↑
  - 데이터 학습 ↑, 정확성 ↑
  - 정밀하고 커스터마이징 된 서비스 제공
financials: "[[1. PLTR 분석.xlsx]]"
latest_earnings: 2026-02-02
upcoming_earnings: 2026-05-02
shares: "143.06"
price: "100"
status: holding
---

## 💡 투자이유

- 독점적인 데이터 분석 솔루션 제공

## ⚠️ 리스크 요소

-  독점체계 붕괴
-  지속적인 성장 꺾임


## 🔄 사이클

![[PLTR 사이클.png]]

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
FROM "3. 어닝콜/PLTR"
WHERE contains(file.name, "summary") 
SORT date(announce_date) DESC
```

## 💬 언급 
```dataviewjs
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
const pages=dv.pages(`"${TARGET}"`)
.where(p=>/_transcript/i.test(p.file.name))
.where(p=>FILE_NAME_RE.test(p.file.name));

const rows=[];

for(const p of pages){

  const file=app.vault.getAbstractFileByPath(p.file.path);
  if(!file) continue;

  const txt=await app.vault.cachedRead(file);

  const fnm=p.file.name.match(FILE_NAME_RE);
  const periodFromName=fnm?.groups?.period??"";

  const blocks=extractBlocks(txt);

  for(const b of blocks){

    if(!b.bid) continue;

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

    const target = `${r.path}#^${r.block}`;

    const a=document.createElement("a");
    a.href=target;
    a.className="internal-link";
    a.setAttribute("data-href", target);
    a.textContent=r.quote.slice(0,220);

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
