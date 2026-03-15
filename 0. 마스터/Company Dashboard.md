
## 🏢기업개요
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


## 📊최근 실적(4분기)
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

}function formatMarketCap(n) {

  if (!Number.isFinite(n) || n <= 0) return "";

  if (n >= 1e12) return (n / 1e12).toFixed(2).replace(/\.00$/,"") + "T";

  if (n >= 1e9)  return (n / 1e9 ).toFixed(2).replace(/\.00$/,"") + "B";

  if (n >= 1e6)  return (n / 1e6 ).toFixed(2).replace(/\.00$/,"") + "M";

  return String(n);

}function norm(v) {

  if (Array.isArray(v)) return v.filter(x => x != null && String(x).trim() !== "").join(", ") || "";

  if (v == null || String(v).trim() === "") return "";

  return String(v);

}const toTicker = v => (v ?? "").toString().trim().toUpperCase();

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

  .where(p => p.announce_date && p.file.name.includes("summary"))

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


## 🎙️어닝콜
```dataviewjs

/***** 사용자 설정 *****/

const ROOT = '3. 어닝콜'; // 트랜스크립트가 있는 최상위 폴더

const COMPANY_ROOT = '1. 기업'; // 시총 메타데이터가 있는 최상위 폴더

const FILE_NAME_RE = /^(?<company>[^_]+)_(?<period>\d{4}Q[1-4])_transcript/i;

/***** 유틸 *****/

// 분기 정렬용 키

const periodKey = (s) => {

const m = /(\d{4})Q([1-4])/i.exec(String(s||'').trim());

return m ? (parseInt(m[1],10) * 10 + parseInt(m[2],10)) : -1; // 큰 값이 최신

};

// (참고) 어닝콜 트리에서 회사 폴더 추정 (시총 조회에는 사용 안 함)

const getCompanyDir = (path) => {

const parts = String(path).replace(/\\/g,'/').split('/');

const i = parts.indexOf(ROOT);

if (i >= 0 && parts.length > i + 1) return parts.slice(0, i + 2).join('/');

return parts.slice(0, -1).join('/');

};

// market_cap 문자열 파서: '3.2T', '350B', '25M', '3.4조', '750억', '25,000,000,000'

const parseMarketCap = (v) => {

if (v == null) return 0;

if (typeof v === 'number') return v;

const s = String(v).trim();

// 한글 단위: 조(1e12), 억(1e8)

const kMatch = s.match(/([\d.,]+)\s*(조|억)/);

if (kMatch) {

const num = parseFloat(kMatch[1].replace(/,/g,''));

const unit = kMatch[2];

if (!isNaN(num)) {

if (unit === '조') return num * 1e12;

if (unit === '억') return num * 1e8;

}

}

// 영어 단위: T/B/M/K

const enMatch = s.match(/([\d.,]+)\s*([TMBK])\b/i);

if (enMatch) {

const num = parseFloat(enMatch[1].replace(/,/g,''));

const unit = enMatch[2].toUpperCase();

if (!isNaN(num)) {

if (unit === 'T') return num * 1e12;

if (unit === 'B') return num * 1e9;

if (unit === 'M') return num * 1e6;

if (unit === 'K') return num * 1e3;

}

}

// 단위 없음: 숫자만

const plain = parseFloat(s.replace(/[^\d.]/g,''));

return isNaN(plain) ? 0 : plain;

};

// 사람이 읽기 쉬운 시총 표기

const humanizeCap = (n) => {

if (!n || !isFinite(n)) return '';

if (n >= 1e12) return (n/1e12).toFixed(2).replace(/\.?0+$/,'') + 'T';

if (n >= 1e9) return (n/1e9 ).toFixed(2).replace(/\.?0+$/,'') + 'B';

if (n >= 1e6) return (n/1e6 ).toFixed(2).replace(/\.?0+$/,'') + 'M';

return n.toLocaleString();

};

/***** 회사명 기반 market_cap 탐색 (폴더/단일파일 모두 지원 + 느슨매칭) *****/

// 이름 정규화: 소문자, 공백/기호 제거, &→and, 기업접미사 제거

const NORMALIZE = s =>

String(s||'')

.toLowerCase()

.replace(/&/g,'and')

.replace(/\band\b/g,'and')

.replace(/incorporated|inc|corp(oration)?|co(mpany)?|ltd|limited/gi,'')

.replace(/[\s._\-]/g,'')

.trim();

// 가벼운 레벤슈타인(편집거리) — 오타 허용(≤2)

const lev = (a,b,max=2)=>{

a=String(a); b=String(b);

const n=a.length,m=b.length;

if (Math.abs(n-m)>max) return max+1;

const dp=new Array(m+1).fill(0).map((_,j)=>j);

for(let i=1;i<=n;i++){

let prev=dp[0]; dp[0]=i; let best=dp[0];

for(let j=1;j<=m;j++){

const t=dp[j];

dp[j]=Math.min(dp[j]+1, dp[j-1]+1, prev+(a[i-1]===b[j-1]?0:1));

prev=t; if (dp[j]<best) best=dp[j];

}

if (best>max) return max+1;

}

return dp[m];

};

const META_FILES = ['_meta.md','index.md','meta.md','README.md'];

const getFM = f => app.metadataCache.getFileCache(f)?.frontmatter;

/**

* 회사명으로 COMPANY_ROOT 아래에서 market_cap을 찾아 반환

* 탐색 우선순위:

* 1) exact 파일: 1. 기업/<company>.md

* 2) COMPANY_ROOT의 .md / 폴더 내부 md 중

* - 파일명 정규화 완전일치

* - frontmatter name/aliases 일치

* - 파일명 편집거리(≤2) 허용

*/

const getCompanyMarketCapByName = (companyName) => {

try {

if (!companyName) return 0;

const target = NORMALIZE(companyName);

// 1) exact path 시도: 1. 기업/<company>.md

{

const exact = app.vault.getAbstractFileByPath(`${COMPANY_ROOT}/${companyName}.md`);

const v = exact && getFM(exact)?.market_cap;

if (v != null) return parseMarketCap(v);

}

// 2) COMPANY_ROOT 아래 항목 수집

const root = app.vault.getAbstractFileByPath(COMPANY_ROOT);

if (!root || !root.children) return 0;

const files = [];

for (const ch of root.children) {

if (ch?.children) {

// 폴더인 경우: 내부 md 수집 (동명 md 우선, 그 다음 meta 후보, 그 외)

const md = ch.children.filter(x => x.path?.toLowerCase().endsWith('.md'));

const sameName = md.find(f => NORMALIZE(f.name.replace(/\.md$/i,'')) === NORMALIZE(ch.name));

if (sameName) files.push(sameName);

const pri = [

...md.filter(f => META_FILES.some(n => f.path.endsWith('/'+n))),

...md.filter(f => !META_FILES.some(n => f.path.endsWith('/'+n))),

];

files.push(...pri);

} else if (ch.path?.toLowerCase().endsWith('.md')) {

// COMPANY_ROOT 바로 아래의 단일 md

files.push(ch);

}

}

// 2-1) 파일명 정규화 완전일치

let hit = files.find(f => NORMALIZE(f.name.replace(/\.md$/i,'')) === target);

// 2-2) frontmatter name/aliases 일치

if (!hit) {

hit = files.find(f => {

const fm = getFM(f);

if (!fm) return false;

const nameHit = fm.name && NORMALIZE(fm.name) === target;

const aliasHit = Array.isArray(fm.aliases) && fm.aliases.some(a => NORMALIZE(a) === target);

return nameHit || aliasHit;

});

}

// 2-3) 오타 허용(편집거리 ≤ 2)

if (!hit) {

hit = files.find(f => lev(NORMALIZE(f.name.replace(/\.md$/i,'')), target, 2) <= 2);

}

if (!hit) return 0;

// 최종 파일의 frontmatter에서 market_cap

const fm = getFM(hit);

if (fm && fm.market_cap != null) return parseMarketCap(fm.market_cap);

// 같은 디렉터리의 meta 후보 재시도

const dir = hit.parent;

if (dir?.children) {

const metas = dir.children.filter(x => META_FILES.some(n => x.path.endsWith('/'+n)));

for (const m of metas) {

const v = getFM(m)?.market_cap;

if (v != null) return parseMarketCap(v);

}

}

return 0;

} catch (e) {

console.error('market_cap read error for', companyName, e);

return 0;

}

};

/***** 대상 노트 수집 (하위폴더 포함) *****/

const pages = dv.pages(`"${ROOT}"`)

.where(p => /_transcript/i.test(p.file.name))

.where(p => FILE_NAME_RE.test(p.file.name));

/***** CEO 콜아웃 추출 *****/

const rows = [];

for (const p of pages) {

const file = app.vault.getAbstractFileByPath(p.file.path);

const txt = await app.vault.cachedRead(file);

const fnm = p.file.name.match(FILE_NAME_RE);

const companyFromName = fnm?.groups?.company ?? '';

const periodFromName = fnm?.groups?.period ?? '';

const companyDir = getCompanyDir(p.file.path);

// > [!ceo] Speaker | Period ... ^blockid

const reCallout =

/>\s*\[!ceo\]\s*(?<name>.+?)\s*\|\s*(?<period>.+?)\n(?<body>(?:>.*\n?)+).*?\^(?<bid>[a-zA-Z0-9\-]+)/g;

for (const m of txt.matchAll(reCallout)) {

const speaker = m.groups.name.trim();

const period = (p.fiscal_period ?? m.groups.period ?? periodFromName).trim();

const quote = (m.groups.body || '').replace(/^>\s?/gm,'').trim();

const bid = m.groups.bid.trim();

rows.push({

company: (p.company ?? companyFromName).trim(),

ticker: (p.ticker ?? '').trim(),

period,

periodKey: periodKey(period || periodFromName),

date: String(p.announce_date ?? ''), // 보조 정렬

speaker,

quote: `[[${p.file.path}#^${bid}|${quote.length > 220 ? quote.slice(0,220) + '…' : quote}]]`,

path: p.file.path,

companyDir

});

}

}/***** row 기준 기본 정렬(회사명 → 분기 ↓ → 날짜 ↓) *****/

rows.sort((a, b) =>

a.company.localeCompare(b.company, 'ko') ||

b.periodKey - a.periodKey ||

b.date.localeCompare(a.date)

);

/***** 그룹핑 + 시총 주입 *****/

const groups = new Map(); // Map(company -> {items, ticker, marketCap})

for (const r of rows) {

const key = r.company || "(회사 미지정)";

if (!groups.has(key)) {

groups.set(key, {

items: [],

ticker: (r.ticker || '').trim(),

marketCap: 0

});

}

groups.get(key).items.push(r);

}// 회사명 기준으로 COMPANY_ROOT에서 market_cap 읽기

for (const [company, g] of groups.entries()) {

g.marketCap = getCompanyMarketCapByName(company) || 0;

// 디버깅 원하면 아래 주석 해제:

// console.log('mcap', company, g.marketCap);

}/***** 출력: 시가총액 내림차순으로 회사 그룹 정렬 후 렌더링 *****/

const sortedCompanies = Array.from(groups.entries())

.sort((a,b) => (b[1].marketCap - a[1].marketCap) || a[0].localeCompare(b[0], 'ko'));

if (sortedCompanies.length === 0) {

dv.paragraph("> [!info] 검색된 CEO 인용이 없습니다.");

} else {

for (const [company, g] of sortedCompanies) {

const firstTicker = g.ticker ? ` (${g.ticker})` : "";

const capStr = g.marketCap ? ` — mcap: ${humanizeCap(g.marketCap)}` : "";

dv.header(2, "🏢 " + company + firstTicker);

// 회사 내부 정렬(안전하게 한 번 더)

g.items.sort((a,b) => b.periodKey - a.periodKey || b.date.localeCompare(a.date));

dv.table(

["분기","화자","요약"],

g.items.map(r => [r.period, r.speaker, r.quote])

);

}

}
```