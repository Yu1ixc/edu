<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>教學小工具｜抽號器＋比較題（長方形／三角形／圖片）</title>
<style>
:root{
  --bg:#f6f7fb; --panel:#fff; --primary:#2563eb;
  --yellow:#ffcc33; --purple:#a066ff;
}
*{box-sizing:border-box}
body{margin:0;font-family:"Noto Sans TC","Microsoft JhengHei",system-ui,Arial,sans-serif;background:var(--bg);}
.container{max-width:1100px;margin:24px auto;padding:16px;}
.grid{display:grid;gap:16px;}
@media(min-width:960px){.grid{grid-template-columns:220px 1fr}}

.card{background:var(--panel);border-radius:14px;box-shadow:0 6px 20px rgba(0,0,0,.08);padding:16px;position:relative;}
.card.left{display:flex;flex-direction:column;justify-content:center;}
.card.left h2{text-align:center;margin:0 0 12px;}

.badge{display:flex;align-items:center;justify-content:center;height:64px;border-radius:12px;background:#eef2ff;font-weight:800;font-size:28px;border:2px dashed #c7d2fe;}
.btn{border:none;padding:10px 14px;border-radius:10px;font-weight:600;cursor:pointer;transition:.15s;box-shadow:0 3px 10px rgba(0,0,0,.1)}
.btn.primary{background:var(--primary);color:#fff;}
.btn.ghost{background:#e5e7eb;color:#111827;}
.progress{width:100%;height:8px;background:#eef2ff;border-radius:999px;overflow:hidden;margin-top:8px;}
.progress>span{display:block;width:0;height:100%;background:#6366f1;transition:width .06s linear;}

.stage{position:relative;display:flex;flex-direction:column;align-items:center;gap:12px;min-height:480px;padding:12px;border:2px dashed #e5e7eb;border-radius:12px;background:#fafafa;}
.prompt{font-size:26px;font-weight:700;text-align:center;min-height:60px;}
.canvas{display:flex;align-items:center;justify-content:center;gap:40px;width:100%;max-width:860px;min-height:260px;}
.label{position:absolute;bottom:8px;left:8px;background:rgba(255,255,255,.9);border-radius:8px;padding:4px 8px;font-weight:700;font-size:14px;}

.rect{position:relative;display:flex;align-items:center;justify-content:center;border-radius:8px;}
.rect.red{background:#ef4444}.rect.blue{background:#2563eb}

.answers{display:flex;gap:12px;flex-wrap:wrap;justify-content:center;}
.answers .btn{min-width:140px;font-size:18px;padding:12px 18px;border-radius:12px;}
.answers .btn.red{background:#ef4444;color:#fff}
.answers .btn.blue{background:#2563eb;color:#fff}
.answers .btn.yellow{background:var(--yellow);color:#000;}
.answers .btn.purple{background:var(--purple);color:#fff;}

.overlay{position:absolute;inset:0;display:none;align-items:center;justify-content:center;background:rgba(255,255,255,.9);border-radius:10px;z-index:5}
.overlay.show{display:flex}
.icon{width:80px;height:80px;border-radius:50%;}
.icon.circle{border:6px solid #10b981;}
.icon.cross{border:6px solid #ef4444;position:relative;}
.icon.cross::before,.icon.cross::after{content:"";position:absolute;top:50%;left:50%;width:38px;height:6px;background:#ef4444;transform-origin:center;}
.icon.cross::before{transform:translate(-50%,-50%) rotate(45deg);}
.icon.cross::after{transform:translate(-50%,-50%) rotate(-45deg);}

.control-bar{display:flex;justify-content:center;gap:10px}
.hidden{display:none !important;}
.imgBox{position:relative;display:flex;align-items:center;justify-content:center;border-radius:8px;overflow:hidden}
.imgBox img{display:block;max-width:100%;height:auto}
</style>
</head>
<body>
<div class="container">
  <div class="grid">
    <!-- 抽號器 -->
    <section class="card left">
      <h2>抽號器</h2>
      <div class="badge" id="ticket">—</div>
      <div class="progress"><span id="progressBar"></span></div>
      <div style="text-align:center;margin-top:10px;">
        <button class="btn primary" id="drawBtn">抽號碼</button>
        <button class="btn ghost" id="resetBtn">清空</button>
      </div>
    </section>

    <!-- 題目區 -->
    <section class="card">
      <div class="stage">
        <div class="overlay" id="overlay"></div>
        <div class="prompt" id="prompt">按「抽題」開始</div>
        <div class="canvas" id="canvas"></div>

        <div class="answers hidden" id="answers">
          <button class="btn red" id="btnA">選 A</button>
          <button class="btn blue" id="btnB">選 B</button>
        </div>

        <div class="control-bar">
          <button class="btn ghost" id="idleBtn">抽題</button>
          <button class="btn ghost hidden" id="backBtn">返回抽題</button>
        </div>
      </div>
    </section>
  </div>
</div>

<script>
/* ===== 快捷 ===== */
const $ = sel => document.querySelector(sel);

/* ===== WebAudio：拉霸機「滾輪聲」＋提示音 ===== */
let audioCtx=null, reelNoise=null, reelGain=null, tickTimer=null;
function ctx(){ return audioCtx || (audioCtx = new (window.AudioContext||window.webkitAudioContext)()); }

// 拉霸機滾輪聲：帶濾波的噪音 + 高頻「叮」tick
function startSlotRolling(){
  const C = ctx();
  // 白噪音
  const buffer = C.createBuffer(1, C.sampleRate*1, C.sampleRate);
  const data = buffer.getChannelData(0);
  for (let i=0;i<data.length;i++) data[i] = Math.random()*2-1;
  reelNoise = C.createBufferSource(); reelNoise.buffer = buffer; reelNoise.loop = true;

  // 濾波＆音量
  const bp = C.createBiquadFilter(); bp.type = "bandpass"; bp.frequency.value = 1800; bp.Q.value = 1.2;
  reelGain = C.createGain(); reelGain.gain.value = 0.08;

  reelNoise.connect(bp).connect(reelGain).connect(C.destination);
  reelNoise.start();

  // 高頻叮叮：每 ~120ms 出一個短促的正弦
  tickTimer = setInterval(()=>{
    const o = C.createOscillator();
    const g = C.createGain();
    o.type = "sine";
    // 小幅隨機音高，像轉輪卡齒聲
    o.frequency.value = 1000 + Math.random()*400;
    g.gain.value = 0.12;
    o.connect(g).connect(C.destination);
    const t = C.currentTime;
    // 短包絡
    g.gain.setValueAtTime(0.0,t);
    g.gain.linearRampToValueAtTime(0.12,t+0.01);
    g.gain.exponentialRampToValueAtTime(0.001,t+0.09);
    o.start(t); o.stop(t+0.1);
  },120);
}
function stopSlotRolling(){
  if (reelNoise){
    try{
      const C = ctx();
      reelGain.gain.exponentialRampToValueAtTime(0.0001, C.currentTime+0.08);
      reelNoise.stop(C.currentTime+0.09);
    }catch(e){}
    reelNoise=null; reelGain=null;
  }
  if (tickTimer){ clearInterval(tickTimer); tickTimer=null; }
}
function beep(freq=880,dur=0.12,vol=0.16){
  const C = ctx();
  const o=C.createOscillator(), g=C.createGain();
  o.type="sine"; o.frequency.value=freq; g.gain.value=vol;
  o.connect(g).connect(C.destination);
  const t=C.currentTime; o.start(t); o.stop(t+dur);
}
function success(){ beep(880,0.12); setTimeout(()=>beep(1175,0.14),150); }
function fail(){ beep(300,0.10,0.18); setTimeout(()=>beep(240,0.12,0.18),160); }

/* ===== 抽號器：3 秒進度條 + 拉霸聲 ===== */
const ticketEl = $('#ticket'), bar = $('#progressBar'), drawBtn = $('#drawBtn'), resetBtn = $('#resetBtn');
drawBtn.addEventListener('click', ()=>{
  const D = 3000, t0 = performance.now();
  startSlotRolling();
  const timer = setInterval(()=>{
    const p = Math.min((performance.now()-t0)/D,1);
    ticketEl.textContent = Math.floor(2 + Math.random()*25);
    bar.style.width = (p*100)+'%';
    if (p>=1){
      clearInterval(timer);
      stopSlotRolling();
      ticketEl.textContent = Math.floor(2 + Math.random()*25);
    }
  }, 60);
});
resetBtn.addEventListener('click', ()=>{ ticketEl.textContent='—'; bar.style.width='0%'; });

/* ===== 題目邏輯（rect / tri / img）每輪各一次 ===== */
const promptEl=$('#prompt'), canvas=$('#canvas'), answers=$('#answers'), btnA=$('#btnA'), btnB=$('#btnB');
const idleBtn=$('#idleBtn'), backBtn=$('#backBtn'), overlay=$('#overlay');

const ALL = ['rect','tri','img'];
let queue = [];
function shuffle(a){ for(let i=a.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [a[i],a[j]]=[a[j],a[i]]; } return a; }
function resetQueue(){ queue = shuffle(ALL.slice()); }
resetQueue();

let currentType=null, currentPayload=null;

function setIdle(){
  promptEl.textContent='按「抽題」開始';
  canvas.innerHTML=''; answers.classList.add('hidden');
  idleBtn.classList.remove('hidden');
  backBtn.classList.add('hidden');
}
function startNext(){
  if (queue.length===0) resetQueue();
  currentType = queue.shift();
  promptEl.textContent='哪個圖形更大';
  answers.classList.remove('hidden');
  idleBtn.classList.add('hidden');
  backBtn.classList.add('hidden');
  if (currentType==='rect') renderRect();
  else if (currentType==='tri') renderTri();
  else renderImg();
}

/* --- 題1：長方形（紅A小／藍B大） --- */
function renderRect(){
  canvas.innerHTML='';
  btnA.className='btn red'; btnB.className='btn blue';
  const aW=80,aH=60, bW=240,bH=160;
  const A=document.createElement('div'); A.className='rect red'; A.style.width=aW+'px'; A.style.height=aH+'px'; A.innerHTML='<span class="label">A</span>';
  const B=document.createElement('div'); B.className='rect blue'; B.style.width=bW+'px'; B.style.height=bH+'px'; B.innerHTML='<span class="label">B</span>';
  canvas.append(A,B);
  currentPayload={};
}

/* --- 題2：三角形（黃A小／紫B大；無外框） --- */
function renderTri(){
  canvas.innerHTML='';
  btnA.className='btn yellow'; btnB.className='btn purple';
  const A=triangle(110,85,'var(--yellow)','A');
  const B=triangle(270,200,'var(--purple)','B');
  canvas.append(A,B);
  currentPayload={};
}
function triangle(w,h,color,label){
  const wrap=document.createElement('div'); wrap.className='imgBox'; wrap.style.width=w+'px'; wrap.style.height=h+'px';
  const svg=document.createElementNS('http://www.w3.org/2000/svg','svg'); svg.setAttribute('width',w); svg.setAttribute('height',h);
  const poly=document.createElementNS('http://www.w3.org/2000/svg','polygon'); poly.setAttribute('points',`0,${h} ${w/2},0 ${w},${h}`); poly.setAttribute('fill',getComputedStyle(document.documentElement).getPropertyValue(color).trim()||color);
  svg.appendChild(poly);
  const lbl=document.createElement('span'); lbl.className='label'; lbl.textContent=label;
  wrap.append(svg,lbl); return wrap;
}

/* --- 題3：圖片大小（A=月曆紙小、B=色紙大） --- */
function renderImg(){
  canvas.innerHTML='';
  btnA.className='btn red'; btnB.className='btn blue'; // 這題用通用按鈕色
  const A = makeImageBox('月曆紙.jpg', 180, 140, 'A');   // 小
  const B = makeImageBox('色紙.jpg', 360, 260, 'B');    // 大
  canvas.append(A,B);
  currentPayload={}; // 保留格式以利未來擴充
}
function makeImageBox(src,w,h,label){
  const box=document.createElement('div'); box.className='imgBox'; box.style.width=w+'px'; box.style.height=h+'px';
  const img=new Image(); img.src = src; img.alt = label;
  img.style.width='100%'; img.style.height='100%'; img.style.objectFit='cover';
  const lbl=document.createElement('span'); lbl.className='label'; lbl.textContent=label;
  box.append(img,lbl); return box;
}

/* ===== 作答與覆蓋層 ===== */
function showOverlay(ok){
  overlay.classList.add('show');
  overlay.innerHTML = ok
    ? `<div class="icon circle"></div><p style="color:#065f46;font-size:24px;margin-top:10px;">答對了</p>`
    : `<div class="icon cross"></div><p style="color:#b91c1c;font-size:24px;margin-top:10px;">答錯了</p>`;
  setTimeout(()=>{
    overlay.classList.remove('show'); overlay.innerHTML='';
    if (ok){
      setIdle();
    }else{
      // 答錯：留在題目畫面，隱藏抽題、顯示返回
      idleBtn.classList.add('hidden');
      backBtn.classList.remove('hidden');
    }
  }, 1200);
}
$('#btnA').addEventListener('click', ()=>{ fail(); showOverlay(false); }); // A 永遠較小
$('#btnB').addEventListener('click', ()=>{ success(); showOverlay(true); });

/* ===== 控制列 ===== */
idleBtn.addEventListener('click', startNext);
backBtn.addEventListener('click', setIdle);

/* ===== 初始 ===== */
setIdle();

/* ===== 快捷鍵：空白抽題、Enter選B、D抽號碼 ===== */
window.addEventListener('keydown',(e)=>{
  if (e.key===' '){ e.preventDefault(); if (!idleBtn.classList.contains('hidden')) startNext(); }
  else if (e.key==='Enter'){ $('#btnB').click(); }
  else if (e.key.toLowerCase()==='d'){ drawBtn.click(); }
});
</script>
</body>
</html>
