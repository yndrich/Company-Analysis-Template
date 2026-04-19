---
company: HIMS
ticker: HIMS
sector: Defensive
industry: Product
market_cap: 5.24B
CEO: 앤드류 듀둠
career:
  - 경영/경제 학사
  - Atomic 창업자
  - 12개 기업 육성
inauguration: "2017"
competitors:
tags:
  - 원격의료
  - 약값
  - 혼합약물
  - AI
cycle:
  - 원격의료 및 약처방
  - 구독자 ↑
  - 데이터, 통찰
  - 최적화된 개인맞춤 의료플랫폼
financials: "[[1. HIMS 분석.xlsx]]"
latest_earnings: 2026-02-23
upcoming_earnings: 2026-05-04
cash_asset:
price: 28.66
buy_min: 14
buy_max: 18
target_min:
target_max:
accounts:
  - name: ho
    shares: 709
    total_cost: 26705.47
    status: holding
  - name: father
    shares: 389
    total_cost: 13442
    status: holding
  - name: sister
    shares: 50
    total_cost: 2025.46
    status: holding
---

## 💡 투자이유

- 소비자 중심 차세대 헬스케어 플랫폼으로 미국 의료 시스템 혁신
- 제조사·유통사·소매점·PBM·보험사 구조 개혁 

## ⚠️ 리스크 요소

- 법적 규제
- 대형 제약사들의 횡포
- 소비자들의 신뢰·인식 

## 🔄 사이클

![[HIMS 사이클.png]]

1. 원격의료 및 약처방
2. 구독자 ↑
3. 데이터, 통찰
4. 최적화된 개인맞춤 의료플랫폼

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
FROM "3. 어닝콜/HIMS"
WHERE contains(file.name, "summary") 
SORT date(announce_date) DESC
```

## 💬 언급
```dataviewjs
/***** 고정 대상 경로: 현재 노트명 기반 *****/
const ROOT   = '3. 어닝콜';
const COMPANY_FROM_NOTE = (dv.current().file.name || '').trim();   // 예: "NVDIA"
const TARGET = `${ROOT}/${COMPANY_FROM_NOTE}`;                      // 예: "3. 어닝콜/NVDIA"

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
}

/***** 대상 노트 수집 (하위폴더 포함) *****/
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
  const periodFromName  = fnm?.groups?.period ?? '';

  const blocks = extractCeoBlocks(txt);
  for (const b of blocks) {
    if (!b.bid) continue;
    const speaker = (b.block.match(/^>\s*\[!ceo\]\s*(.+?)\s*\|/i)?.[1] ?? '').trim();
    const period  = (p.fiscal_period ?? b.block.match(/\|\s*(.+?)\s*$/m)?.[1] ?? periodFromName).trim();
    const quote   = b.body;

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
}

/***** 정렬: 분기 내림차순 → (보조) 날짜 내림차순 *****/
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