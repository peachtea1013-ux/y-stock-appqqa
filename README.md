<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<title>工場在庫管理PWA</title>

<link rel="manifest" href="manifest.json">
<meta name="theme-color" content="#007bff">
<meta name="apple-mobile-web-app-capable" content="yes">

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<style>
:root{--primary:#007bff;--danger:#dc3545;--success:#28a745}
body{margin:0;font-family:sans-serif;background:#f4f7f6;padding-bottom:90px}
button{min-height:48px;font-size:1rem;border:none;border-radius:8px}
.card{background:#fff;border-radius:12px;padding:15px;margin-bottom:10px}
.alert{background:#fff5f5;border:1px solid #feb2b2}
.zero{color:var(--danger);font-weight:bold}
.bottom-nav{position:fixed;bottom:0;width:100%;display:flex;background:#fff;border-top:1px solid #ddd}
.nav-item{flex:1;text-align:center;padding:12px}
.nav-item.active{color:var(--primary);font-weight:bold}
</style>
</head>

<body>
<div id="mainContent" style="display:none">

<div id="page-stock">
  <div id="shortageArea"></div>
  <div id="stockList"></div>
</div>

<div id="page-chart" style="display:none">
  <button onclick="chartMode='qty';updateChart()">数量</button>
  <button onclick="chartMode='yen';updateChart()">金額</button>
  <canvas id="usageChart"></canvas>
</div>

<nav class="bottom-nav">
  <div class="nav-item active" onclick="showPage('stock')">在庫</div>
  <div class="nav-item" onclick="showPage('chart')">分析</div>
</nav>
</div>

<script>
/* ===== 認証 ===== */
async function hash(t){
  const b=await crypto.subtle.digest("SHA-256",new TextEncoder().encode(t));
  return Array.from(new Uint8Array(b)).map(x=>x.toString(16).padStart(2,"0")).join("");
}

async function checkAuth(){
  const now=Date.now();
  if(localStorage.auth_time && now-localStorage.auth_time<8*60*60*1000){
    mainContent.style.display="block";render();return;
  }
  if(!localStorage.pass){
    localStorage.pass=await hash(prompt("初期パス設定"));
  }
  if(await hash(prompt("パスワード"))===localStorage.pass){
    localStorage.auth_time=now;
    mainContent.style.display="block";render();
  }else alert("認証失敗");
}
window.onload=checkAuth;

/* ===== データ ===== */
let items=JSON.parse(localStorage.items||"[]");
let logs=JSON.parse(localStorage.logs||"[]");
let chartMode="qty";

/* ===== 表示 ===== */
function showPage(p){
  page-stock.style.display=p==='stock'?'block':'none';
  page-chart.style.display=p==='chart'?'block':'none';
  document.querySelectorAll('.nav-item').forEach(n=>n.classList.remove('active'));
  event.target.classList.add('active');
  if(p==='chart')updateChart();
}

function render(){
  const shortage=items.filter(i=>i.count<=i.min);
  shortageArea.innerHTML=shortage.length?
    `<div class="card alert">⚠ 要補充：${shortage.map(s=>s.name).join(',')}</div>`:'';

  stockList.innerHTML=items.map(i=>`
  <div class="card ${i.count<=i.min?'alert':''}">
    <strong>${i.name}</strong>
    <div class="${i.count===0?'zero':''}">在庫：${i.count}</div>
    <input type="number" id="q-${i.id}" placeholder="数量">
    <button onclick="act(${i.id},-1)">使用</button>
    <button onclick="act(${i.id},1)" style="background:var(--success);color:#fff">納入</button>
  </div>`).join('');

  localStorage.items=JSON.stringify(items);
  localStorage.logs=JSON.stringify(logs);
}

/* ===== 在庫操作 ===== */
function act(id,m){
  const v=parseInt(document.getElementById(`q-${id}`).value);
  if(!v||v<=0)return alert("数量入力");
  const i=items.find(x=>x.id===id);
  if(m<0 && i.count<v && !confirm("在庫不足です。続行しますか？"))return;
  if(!confirm(`${v}個 実行しますか？`))return;
  i.count=Math.max(0,i.count+v*m);
  logs.unshift({month:new Date().toISOString().slice(0,7),qty:v,type:m<0?'使用':'納入',price:i.price});
  render();
}

/* ===== グラフ ===== */
let chart;
function updateChart(){
  const ctx=usageChart.getContext('2d');
  const months=[...new Set(logs.map(l=>l.month))];
  const data=months.map(m=>logs.filter(l=>l.month===m&&l.type==='使用')
    .reduce((s,l)=>s+(chartMode==='yen'?l.qty*l.price:l.qty),0));
  if(chart)chart.destroy();
  chart=new Chart(ctx,{type:'bar',data:{labels:months,datasets:[{data}]},options:{plugins:{legend:{display:false}}}});
}

/* ===== 初期 ===== */
if(items.length===0){
  items.push({id:1,name:"ボルトM8",count:50,min:10,price:20});
}
</script>

<script>
if('serviceWorker' in navigator){
  navigator.serviceWorker.register('sw.js');
}
</script>

</body>
</html>
