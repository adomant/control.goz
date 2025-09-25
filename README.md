# control.goz
Контроль ГОЗ без РОС
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>ГОЗ — Контроль лимита с типами платежей</title>
  <style>
    :root { --bg:#f5f7fb; --card:#fff; --ink:#1f2937; --muted:#6b7280; --brand:#2563eb; --ok:#16a34a; --warn:#f59e0b; --bad:#dc2626; --exceed:#fde2e7; }
    *{box-sizing:border-box}
    body{margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Arial,sans-serif;background:var(--bg);color:var(--ink)}
    .wrap{max-width:1200px;margin:24px auto;padding:12px}
    .bar{display:flex;gap:8px;align-items:center;flex-wrap:wrap;margin-bottom:12px}
    .brand{font-weight:700;font-size:18px;margin-right:auto}
    .btn{border:0;background:var(--brand);color:#fff;border-radius:10px;padding:8px 12px;font-size:13px;cursor:pointer}
    .btn.alt{background:#111827}
    .btn.warn{background:var(--warn)}
    .card{background:var(--card);border-radius:14px;box-shadow:0 2px 10px rgba(0,0,0,.06)}
    .tabs{display:flex;overflow:auto;border-bottom:1px solid #e5e7eb}
    .tab{padding:10px 12px;background:transparent;border:0;cursor:pointer;font-size:13px;color:var(--muted)}
    .tab.active{color:var(--brand);font-weight:700;border-bottom:2px solid var(--brand)}
    .content{padding:12px}
    .table-wrap{overflow:auto;max-height:60vh}
    table{width:100%;border-collapse:collapse;font-size:12px}
    th,td{border:1px solid #e5e7eb;padding:6px;text-align:left}
    th{background:#f3f4f6;position:sticky;top:0}
    input,select{width:100%;border:1px solid #e5e7eb;border-radius:6px;padding:4px;font-size:12px}
    input[type="checkbox"]{width:auto}
    .stats{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:10px;margin-top:8px}
    .kpi{background:#fafafa;border:1px solid #eee;border-radius:12px;padding:10px}
    .kpi b{display:block;font-size:18px}
    .badge{padding:2px 8px;border-radius:999px;background:#eef2ff;font-weight:600;font-size:11px}
    .pill-ok{background:#ecfdf5;color:#065f46}
    .pill-warn{background:#fffbeb;color:#92400e}
    .pill-bad{background:#fef2f2;color:#991b1b}
    .row-exceed{background:var(--exceed)!important}
  </style>
</head>
<body>
<div class="wrap">
  <div class="bar">
    <div class="brand">ГОЗ • Контроль лимита (регулярные / разовые)</div>
    <button class="btn" id="addRowBtn">+ Добавить договор</button>
    <button class="btn alt" id="saveBtn">💾 Сохранить</button>
    <button class="btn alt" id="loadBtn">📥 Загрузить</button>
    <button class="btn warn" id="clearBtn">♻️ Очистить</button>
  </div>

  <div class="card">
    <div class="tabs" id="tabs">
      <button class="tab active" data-tab="registry">Реестр договоров</button>
    </div>
    <div class="content">
      <div id="registry" class="tabpage">
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>№</th>
                <th>Дата</th>
                <th>Предмет</th>
                <th>Контрагент</th>
                <th>Тип платежа</th>
                <th>Сумма обязательства</th>
                <th>Аванс 1 (сумма)</th>
                <th>Аванс 1 (дата)</th>
                <th>Аванс 1 (исп.)</th>
                <th>Оконч. расчёт (сумма)</th>
                <th>Оконч. расчёт (дата)</th>
                <th>Оконч. расчёт (исп.)</th>
                <th>Не исполнено</th>
              </tr>
            </thead>
            <tbody id="tbody-contracts"></tbody>
          </table>
        </div>
        <div class="stats">
          <div class="kpi"><span class="note">Всего договоров</span><b id="kpi-total">0</b></div>
          <div class="kpi"><span class="note">В работе</span><b id="kpi-active">0</b></div>
          <div class="kpi"><span class="note">Неисполненных обязательств</span><b id="kpi-notexec">0 ₽</b></div>
        </div>
      </div>
    </div>
  </div>

  <div class="card" style="margin-top:14px">
    <div class="tabs" id="monthTabs"></div>
    <div class="content" id="monthsHolder">
      <div class="note">Горизонт отображения: текущий месяц + 3 вперёд. Ретроспектива: все прошедшие месяцы только по <b>исполненным</b> платежам. Лимит: <b>5 200 000 ₽</b>/мес.</div>
    </div>
  </div>
</div>

<script>
// === ПАРАМЕТРЫ ===
const MONTH_LIMIT = 5_200_000;
const today = new Date();
const curKey = (d=>`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}`)(today);
const horizonEnd = new Date(today.getFullYear(), today.getMonth()+3, 1); // конец горизонта (начало месяца +3)

// === ДАННЫЕ ===
let contracts = [];
let counter = 1;

// === HELPERS ===
const el = s=>document.querySelector(s);
const fmt = n=> new Intl.NumberFormat('ru-RU',{style:'currency',currency:'RUB',maximumFractionDigits:0}).format(Number(n||0));
const mKey = d=>{ const x=new Date(d); if(Number.isNaN(+x)) return null; return `${x.getFullYear()}-${String(x.getMonth()+1).padStart(2,'0')}`; };
const mName = key=>{ if(!key) return ''; const [y,m]=key.split('-').map(Number); const names=['Январь','Февраль','Март','Апрель','Май','Июнь','Июль','Август','Сентябрь','Октябрь','Ноябрь','Декабрь']; return `${names[m-1]} ${y}`; };
function addMonths(date, n){ const d=new Date(date); d.setMonth(d.getMonth()+n); return d; }
function monthKeysBetween(startDate, endDate){ const a=[]; let d=new Date(startDate.getFullYear(), startDate.getMonth(), 1); const end=new Date(endDate.getFullYear(), endDate.getMonth(), 1); while(d<=end){ a.push(mKey(d)); d.setMonth(d.getMonth()+1);} return a; }

// === UI: РЕЕСТР ===
function addRow(prefill){
  const id = `c_${Date.now()}_${counter++}`;
  const c = Object.assign({ id, number:'', date:'', subject:'', contractor:'', ptype:'single', total:0,
    adv1:0, adv1Date:'', adv1Done:false,
    fin:0, finDate:'', finDone:false,
    notExec:0, done:false }, prefill||{});
  contracts.push(c); renderRegistry();
}

function renderRegistry(){
  const tb = el('#tbody-contracts'); tb.innerHTML = '';
  for (const c of contracts){
    const tr = document.createElement('tr'); tr.id = `row_${c.id}`;
    tr.innerHTML = `
      <td><input value="${c.number}" oninput="updateC('${c.id}','number',this.value)"></td>
      <td><input type="date" value="${c.date}" oninput="updateC('${c.id}','date',this.value)"></td>
      <td><input value="${c.subject}" oninput="updateC('${c.id}','subject',this.value)"></td>
      <td><input value="${c.contractor}" oninput="updateC('${c.id}','contractor',this.value)"></td>
      <td>
        <select onchange="updateC('${c.id}','ptype',this.value)">
          <option value="single" ${c.ptype==='single'?'selected':''}>Разовый</option>
          <option value="regular" ${c.ptype==='regular'?'selected':''}>Регулярный</option>
        </select>
      </td>
      <td><input type="number" value="${c.total}" oninput="updateC('${c.id}','total',parseFloat(this.value)||0)"></td>
      <td><input type="number" value="${c.adv1}" oninput="updateC('${c.id}','adv1',parseFloat(this.value)||0)"></td>
      <td><input type="date" value="${c.adv1Date}" oninput="updateC('${c.id}','adv1Date',this.value)"></td>
      <td style="text-align:center"><input type="checkbox" ${c.adv1Done?'checked':''} onchange="updateC('${c.id}','adv1Done',this.checked)"></td>
      <td><input type="number" value="${c.fin}" oninput="updateC('${c.id}','fin',parseFloat(this.value)||0)"></td>
      <td><input type="date" value="${c.finDate}" oninput="updateC('${c.id}','finDate',this.value)"></td>
      <td style="text-align:center"><input type="checkbox" ${c.finDone?'checked':''} onchange="updateC('${c.id}','finDone',this.checked)"></td>
      <td id="ne_${c.id}">${fmt(c.notExec||0)}</td>`;
    tb.appendChild(tr);
  }
  recalcAll();
}

function updateC(id, field, val){
  const c = contracts.find(x=>x.id===id); if(!c) return; c[field]=val;
  // not executed
  const paid = (c.adv1Done?Number(c.adv1)||0:0) + (c.finDone?Number(c.fin)||0:0);
  c.notExec = Math.max(0,(Number(c.total)||0) - paid);
  c.done = c.notExec<=0 && (c.total||0)>0;
  const cell = el(`#ne_${id}`); if(cell) cell.textContent = fmt(c.notExec);

  // проверка лимита для новых/изменённых — без учёта текущего договора, потом с ним
  checkLimitImpact(c);

  recalcAll();
}

// === ЛОГИКА ПО МЕСЯЦАМ ===
function visibleMonths(){
  // Текущий + 3 вперёд
  const now = new Date();
  const forwardKeys = monthKeysBetween(now, addMonths(now,3));
  // Ретроспектива по исполненным
  const retroKeys = new Set();
  for(const c of contracts){
    if(c.adv1Done && c.adv1Date){ const k=mKey(c.adv1Date); if(k && k<curKey) retroKeys.add(k); }
    if(c.finDone && c.finDate){ const k=mKey(c.finDate); if(k && k<curKey) retroKeys.add(k); }
  }
  return Array.from(new Set([...retroKeys,...forwardKeys])).sort();
}

function rebuildMonths(){
  const keys = visibleMonths();
  const tabs = el('#monthTabs'); tabs.innerHTML='';
  const holder = el('#monthsHolder'); holder.querySelectorAll('.tabpage').forEach(p=>p.remove());
  keys.forEach(k=>{
    const b=document.createElement('button'); b.className='tab'; b.dataset.tab=k; b.textContent=mName(k); b.onclick=()=>showTab(k); tabs.appendChild(b);
    const page=document.createElement('div'); page.id=`m_${k}`; page.className='tabpage'; page.style.display='none';
    page.innerHTML = `
      <div class="table-wrap">
        <table>
          <thead><tr>
            <th>Поставщик</th><th>Сумма</th><th>Плановая дата</th><th>Тип</th><th>Договор</th><th>Исполнено</th>
          </tr></thead>
          <tbody id="mb_${k}"></tbody>
        </table>
      </div>
      <div class="stats">
        <div class="kpi"><span class="note">План на месяц</span><b id="kp_plan_${k}">0 ₽</b></div>
        <div class="kpi"><span class="note">Факт</span><b id="kp_fact_${k}">0 ₽</b></div>
        <div class="kpi"><span class="note">Свободный лимит</span><b id="kp_free_${k}">0 ₽</b></div>
        <div class="kpi"><span class="note">Статус лимита</span><b><span id="kp_status_${k}" class="badge">OK</span></b></div>
      </div>`;
    holder.appendChild(page);
  });
  if(keys.length) showTab(keys[0]);
  keys.forEach(fillMonth);
}

function plannedEntriesForContract(c){
  const entries = [];
  const items = [
    { amt:c.adv1, date:c.adv1Date, done:c.adv1Done, label:'Аванс 1' },
    { amt:c.fin,  date:c.finDate,  done:c.finDone,  label:'Оконч. расчёт' }
  ];
  for(const it of items){
    if(!(Number(it.amt)>0) || !it.date) continue;
    const start = new Date(it.date);
    if(c.ptype==='single'){
      const key = mKey(start);
      entries.push({ key, contractor:c.contractor, amt:Number(it.amt)||0, date:it.date, type:it.label, number:c.number, done:it.done });
    } else {
      // регулярный: ежемесячная трансляция от start до конца горизонта
      const now = new Date();
      const keys = monthKeysBetween(start, addMonths(now,3));
      for(const k of keys){
        const kDateParts = k.split('-').map(Number);
        const d = new Date(kDateParts[0], kDateParts[1]-1, start.getDate());
        const inPast = k < curKey;
        if(inPast && !it.done) continue; // ретро только по исполненным
        entries.push({ key:k, contractor:c.contractor, amt:Number(it.amt)||0, date:d.toISOString().slice(0,10), type:it.label, number:c.number, done:it.done && (k===mKey(it.date)) });
      }
    }
  }
  return entries;
}

function fillMonth(k){
  const body = el(`#mb_${k}`); if(!body) return; body.innerHTML='';
  let plan=0, fact=0;
  // собрать все записи
  let rows=[];
  for(const c of contracts){ rows = rows.concat(plannedEntriesForContract(c).filter(e=>e.key===k)); }
  for(const r of rows){
    plan += r.amt; if(r.done) fact += r.amt;
    const tr=document.createElement('tr');
    tr.innerHTML = `<td>${r.contractor||''}</td><td style="text-align:right;font-weight:700">${fmt(r.amt)}</td><td>${new Date(r.date).toLocaleDateString('ru-RU')}</td><td>${r.type}${/* show Reg/Single hint */''}</td><td>${r.number||''}</td><td style="text-align:center">${r.done?'✅':'⏳'}</td>`;
    body.appendChild(tr);
  }
  // итоги + лимитные подсказки
  const free = MONTH_LIMIT - plan;
  el(`#kp_plan_${k}`).textContent = fmt(plan);
  el(`#kp_fact_${k}`).textContent = fmt(fact);
  el(`#kp_free_${k}`).textContent = free>=0?fmt(free):`Превышение: ${fmt(-free)}`;
  const st = el(`#kp_status_${k}`);
  if(st){ if(plan>MONTH_LIMIT){ st.textContent='Превышение'; st.className='badge pill-bad'; }
          else if(plan>MONTH_LIMIT*0.85){ st.textContent='Почти лимит'; st.className='badge pill-warn'; }
          else { st.textContent='OK'; st.className='badge pill-ok'; } }
}

function showTab(tabId){
  document.querySelectorAll('#monthTabs .tab').forEach(b=>b.classList.toggle('active', b.dataset.tab===tabId));
  document.querySelectorAll('#monthsHolder .tabpage').forEach(p=>p.style.display = p.id===`m_${tabId}`?'block':'none');
}

// === ЛИМИТ: проверка влияния текущего договора ===
function monthlyTotalsExcluding(excludeId){
  const totals = new Map(); // key -> sum
  for(const c of contracts){
    if(c.id===excludeId) continue;
    const entries = plannedEntriesForContract(c);
    for(const e of entries){ totals.set(e.key, (totals.get(e.key)||0) + e.amt); }
  }
  return totals; 
}

function checkLimitImpact(c){
  const row = el(`#row_${c.id}`); if(row) row.classList.remove('row-exceed');
  const base = monthlyTotalsExcluding(c.id);
  // months affected by this contract
  const affected = plannedEntriesForContract(c).map(e=>e.key);
  for(const k of new Set(affected)){
    const before = base.get(k)||0;
    const add = plannedEntriesForContract(c).filter(e=>e.key===k).reduce((s,e)=>s+e.amt,0);
    if(before <= MONTH_LIMIT && (before + add) > MONTH_LIMIT){
      if(row) row.classList.add('row-exceed');
      break;
    }
  }
}

// === KPI + РЕНДЕР ===
function recalcAll(){
  const total = contracts.filter(c=>c.number).length;
  const active = contracts.filter(c=>c.number && !c.done).length;
  const notExec = contracts.reduce((s,c)=>s+(c.notExec||0),0);
  el('#kpi-total').textContent = total;
  el('#kpi-active').textContent = active;
  el('#kpi-notexec').textContent = fmt(notExec);
  rebuildMonths();
}

// === PERSISTENCE ===
function saveLocal(){ localStorage.setItem('goz_reg_v1', JSON.stringify(contracts)); alert('Сохранено'); }
function loadLocal(){ try{ const d=JSON.parse(localStorage.getItem('goz_reg_v1')||'[]'); if(Array.isArray(d)){ contracts = d; renderRegistry(); alert('Загружено'); } }catch(e){ alert('Ошибка загрузки'); } }
function clearAll(){ if(confirm('Очистить все данные?')){ contracts=[]; renderRegistry(); el('#monthTabs').innerHTML=''; el('#monthsHolder').querySelectorAll('.tabpage').forEach(p=>p.remove()); }}

// === WIRE ===
el('#addRowBtn').onclick = ()=> addRow();
el('#saveBtn').onclick = saveLocal;
el('#loadBtn').onclick = loadLocal;
el('#clearBtn').onclick = clearAll;

// === DEMO ROW ===
addRow({ number:'ГОЗ-001/2025', date:'2025-01-15', subject:'Поставка', contractor:'ООО «ТехПром»', ptype:'regular', total:5200000,
  adv1:1500000, adv1Date:'2025-09-05', adv1Done:false,
  fin:2000000,  finDate:'2025-11-10',  finDone:false });
</script>
</body>
</html>

