---
company: NVDIA
ticker: NVDA
sector: Technology
industry: Semiconductor
market_cap: 4505.46B
CEO: 젠슨황
career: 전기공학(학사, 석사), 엔비디아 공동 창업자 및 CEO
inarguration: "1993"
competitors:
tags:
  - AI
  - 반도체
  - 자율주행
cycle:
financials: "[[1. NVDA 분석.xlsx]]"
latest_earnings: 2025-11-19
upcoming_earnings: "20260225"
---
## 💡 투자이유

- AI 필수 회사(반도체, AI인프라(학습,추론))

## ⚠️ 리스크 요소

- 정부 수출 규제
- 신제품 출시 지연(공급 부족)
- AI 수요 감소
- 압도적인 경쟁자 출연

## 📊 실적발표
```dataview
TABLE WITHOUT ID 
	link(file.link, fiscal_period) AS 분기,
	choice(lower(result) = "beat", "↑", choice(lower(result) = "miss", "↓", "-")) AS 결과,
	revenue AS 매출,
	revenue_yoy AS 성장률,
	grossprofit AS 총이익률, 
	eps_yoy AS EPS성장률,
	tags AS 메모 
FROM "3. 어닝콜/NVDA"
WHERE contains(file.name, "summary")
SORT date(announce_date) DESC 
```
## 💬 언급 
```dataviewjs
/***** 고정 대상 경로: 현재 노트명 기반 *****/

const ROOT = '3. 어닝콜';

const COMPANY_FROM_NOTE = (dv.current().file.name || '').trim(); // 예: "NVDIA"

const TARGET = `${ROOT}/${COMPANY_FROM_NOTE}`; // 예: "3. 어닝콜/NVDIA"

const FILE_NAME_RE = /^(?<company>[^_]+)_(?<period>\d{4}Q[1-4])_transcript/i;

/***** 유틸 *****/

const periodKey = (s) => {

const m = /(\d{4})Q([1-4])/i.exec(String(s||'').trim());

return m ? (parseInt(m[1],10) * 10 + parseInt(m[2],10)) : -1;

};

// [!ceo] 콜아웃 라인 파서(콜아웃 내부/바깥의 ^id 모두 허용)

function extractCeoBlocks(text) {

const lines = text.split(/\r?\n/);

const blocks = [];

for (let i = 0; i < lines.length; i++) {

if (/^>\s*\[!ceo\]/i.test(lines[i])) {

const buf = [lines[i]];

let j = i + 1;

while (j < lines.length && /^>\s?/.test(lines[j])) {

buf.push(lines[j]); j++;

}

let bid = null;

for (const L of buf) {

const mIn = L.match(/\^([A-Za-z0-9\-_]+)/); if (mIn) { bid = mIn[1]; break; }

}

if (!bid && j < lines.length) {

const mOut = lines[j].match(/^\^([A-Za-z0-9\-_]+)/);

if (mOut) { bid = mOut[1]; j++; }

}

const body = buf.slice(1).join("\n").replace(/^>\s?/gm, "").trim();

blocks.push({ block: buf.join("\n"), body, bid });

i = j - 1;

}

}

return blocks;

}/***** 대상 노트 수집 (하위폴더 포함) *****/

const pages = dv.pages(`"${TARGET}"`)

.where(p => p.file.folder && p.file.folder.startsWith(TARGET))

.where(p => /_transcript/i.test(p.file.name))

.where(p => FILE_NAME_RE.test(p.file.name));

/***** CEO 콜아웃 추출 *****/

const rows = [];

for (const p of pages) {

const file = app.vault.getAbstractFileByPath(p.file.path);

if (!file) continue;

const txt = await app.vault.cachedRead(file);

const fnm = p.file.name.match(FILE_NAME_RE);

const companyFromName = fnm?.groups?.company ?? '';

const periodFromName = fnm?.groups?.period ?? '';

const blocks = extractCeoBlocks(txt);

for (const b of blocks) {

if (!b.bid) continue;

const speaker = (b.block.match(/^>\s*\[!ceo\]\s*(.+?)\s*\|/i)?.[1] ?? '').trim();

const period = (p.fiscal_period ?? b.block.match(/\|\s*(.+?)\s*$/m)?.[1] ?? periodFromName).trim();

const quote = b.body;

rows.push({

company: (p.company ?? companyFromName).trim(),

ticker : (p.ticker ?? '').trim(),

period,

periodKey: periodKey(period || periodFromName),

date: String(p.announce_date ?? ''),

speaker,

quote: `[[${p.file.path}#^${b.bid}|${quote.length > 220 ? quote.slice(0,220) + '…' : quote}]]`

});

}

}/***** 정렬: 분기 내림차순 → (보조) 날짜 내림차순 *****/

rows.sort((a, b) => b.periodKey - a.periodKey || b.date.localeCompare(a.date));

/***** 출력 *****/

if (rows.length === 0) {

dv.paragraph(`> [!warning] \`"${TARGET}"\`에서 CEO 인용을 찾지 못했습니다.`);

const names = pages.map(p => p.file.name).array().join(", ");

dv.paragraph(`스캔된 transcript 파일 수: ${pages.length}`);

if (names) dv.paragraph(`파일 목록: ${names}`);

} else {

dv.table(["분기","화자","요약"], rows.map(r => [r.period, r.speaker, r.quote]));

}
```


