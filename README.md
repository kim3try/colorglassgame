# colorglassgame
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>컬러 소트 퍼즐</title>
<style>
*{box-sizing:border-box;margin:0;padding:0;}
body{background:#1a1040;display:flex;justify-content:center;align-items:flex-start;min-height:100vh;padding:20px;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;}
#root{background:#2d2456;min-height:580px;width:100%;max-width:480px;border-radius:20px;overflow:hidden;position:relative;}
#topbar{display:flex;justify-content:space-between;align-items:center;padding:12px 16px 8px;background:rgba(0,0,0,0.2);}
#level-txt{font-size:15px;font-weight:500;color:#fff;}
#btn-row{display:flex;gap:8px;}
.hbtn{background:rgba(255,255,255,0.15);border:none;border-radius:20px;padding:6px 14px;font-size:12px;color:#fff;cursor:pointer;transition:background 0.15s;}
.hbtn:hover{background:rgba(255,255,255,0.25);}
#msg{text-align:center;font-size:13px;color:rgba(255,255,255,0.7);min-height:22px;padding:4px 0;}
#bottles-grid{display:flex;flex-wrap:wrap;justify-content:center;align-items:flex-end;gap:10px;padding:10px 12px 16px;}
.bwrap{display:flex;flex-direction:column;align-items:center;cursor:pointer;transition:transform 0.18s cubic-bezier(.34,1.56,.64,1);}
.bwrap.sel{transform:translateY(-16px);}
.bwrap.shake{animation:shk 0.4s;}
@keyframes shk{0%,100%{transform:translateX(0)}20%{transform:translateX(-7px)}40%{transform:translateX(7px)}60%{transform:translateX(-4px)}80%{transform:translateX(4px)}}
.bwrap.pour{animation:pr 0.35s ease-in-out;}
@keyframes pr{0%{transform:rotate(0)}40%{transform:rotate(-22deg) translateY(-10px)}100%{transform:rotate(0)}}
#win-overlay{display:none;position:absolute;inset:0;background:rgba(20,15,50,0.7);align-items:center;justify-content:center;z-index:20;border-radius:20px;}
#win-overlay.show{display:flex;}
#win-card{background:#3a2f7a;border:2px solid rgba(255,255,255,0.2);border-radius:20px;padding:2rem 2.5rem;text-align:center;}
#win-stars{font-size:28px;margin-bottom:6px;letter-spacing:6px;}
#win-t{font-size:22px;font-weight:500;color:#fff;margin-bottom:4px;}
#win-s{font-size:13px;color:rgba(255,255,255,0.6);margin-bottom:1.2rem;}
#next-btn{background:linear-gradient(135deg,#5dcaa5,#1d9e75);border:none;border-radius:20px;padding:10px 28px;font-size:15px;color:#fff;cursor:pointer;font-weight:500;}
#next-btn:hover{opacity:0.9;}
</style>
</head>
<body>
<div id="root">
  <div id="topbar">
    <span id="level-txt">Level 1</span>
    <div id="btn-row">
      <button class="hbtn" id="undo-btn">↩ 되돌리기</button>
      <button class="hbtn" id="reset-btn">↺ 리셋</button>
    </div>
  </div>
  <div id="msg"></div>
  <div id="bottles-grid"></div>
  <div id="win-overlay">
    <div id="win-card">
      <div id="win-stars"></div>
      <div id="win-t">클리어!</div>
      <div id="win-s"></div>
      <button id="next-btn">다음 레벨 →</button>
    </div>
  </div>
</div>

<script>
const COLORS=[
  {key:'red',   fill:'#e8453c',hi:'#ff7b74',lo:'#9e1c17'},
  {key:'blue',  fill:'#3a8ee6',hi:'#7dbfff',lo:'#1a5ca8'},
  {key:'green', fill:'#5ab52e',hi:'#90e85a',lo:'#2d7a10'},
  {key:'amber', fill:'#e89a1a',hi:'#ffcc60',lo:'#9e5f05'},
  {key:'purple',fill:'#8b5cf6',hi:'#c4a3ff',lo:'#4c2896'},
  {key:'teal',  fill:'#16a085',hi:'#5ce0c4',lo:'#0a5c4a'},
  {key:'pink',  fill:'#e0609a',hi:'#ff9fcc',lo:'#8c2255'},
  {key:'cyan',  fill:'#1ab8d8',hi:'#70e8ff',lo:'#0a7090'},
];

const LEVELS=[
  {colorCount:3,emptyExtra:1},
  {colorCount:4,emptyExtra:1},
  {colorCount:4,emptyExtra:2},
  {colorCount:5,emptyExtra:2},
  {colorCount:6,emptyExtra:2},
  {colorCount:7,emptyExtra:2},
  {colorCount:8,emptyExtra:2},
];

const MAX=4;
let bottles=[],selected=null,history=[],levelIdx=0,moves=0;

function colorByKey(k){return COLORS.find(c=>c.key===k);}
function shuffle(a){const r=[...a];for(let i=r.length-1;i>0;i--){const j=Math.floor(Math.random()*(i+1));[r[i],r[j]]=[r[j],r[i]];}return r;}

function genLevel(li){
  const cfg=LEVELS[li];
  const cols=COLORS.slice(0,cfg.colorCount);
  const pool=[];
  cols.forEach(c=>{for(let i=0;i<MAX;i++)pool.push(c.key);});
  const s=shuffle(pool);
  const result=[];
  for(let i=0;i<cfg.colorCount;i++) result.push(s.splice(0,MAX));
  for(let i=0;i<cfg.emptyExtra;i++) result.push([]);
  return shuffle(result);
}

function clone(b){return b.map(x=>[...x]);}
function canPour(fi,ti){if(fi===ti)return false;if(bottles[fi].length===0)return false;if(bottles[ti].length>=MAX)return false;if(bottles[ti].length===0)return true;return bottles[fi][bottles[fi].length-1]===bottles[ti][bottles[ti].length-1];}
function doPour(fi,ti){const top=bottles[fi][bottles[fi].length-1];while(bottles[fi].length>0&&bottles[fi][bottles[fi].length-1]===top&&bottles[ti].length<MAX){bottles[ti].push(bottles[fi].pop());}}
function isDone(b){if(b.length===0)return true;if(b.length!==MAX)return false;return b.every(c=>c===b[0]);}
function checkWin(){return bottles.every(isDone);}
function setMsg(t,d=1800){const e=document.getElementById('msg');e.textContent=t;if(d>0)setTimeout(()=>{if(e.textContent===t)e.textContent='';},d);}

function drawBottleSVG(layers,isSel,done){
  const BW=52,BH=130;
  const nW=20,nH=24,bW=BW-6,bH=BH-nH-16;
  const lH=bH/MAX;
  const bX=3,bY=nH+8;
  const cpId='cp'+Math.random().toString(36).slice(2);
  let s=`<defs><clipPath id="${cpId}"><rect x="${bX}" y="${bY}" width="${bW}" height="${bH}" rx="8"/></clipPath></defs>`;
  s+=`<rect x="${bX}" y="${bY}" width="${bW}" height="${bH}" rx="8" fill="rgba(255,255,255,0.12)" stroke="rgba(255,255,255,0.18)" stroke-width="1"/>`;
  for(let i=0;i<MAX;i++){const colorKey=layers[i];if(!colorKey)continue;const c=colorByKey(colorKey);const ly=bY+bH-(i+1)*lH;s+=`<rect x="${bX}" y="${ly}" width="${bW}" height="${lH}" fill="${c.fill}" clip-path="url(#${cpId})"/>`;}
  s+=`<rect x="${bX}" y="${bY}" width="${bW}" height="${bH}" rx="8" fill="none" stroke="${done&&layers.length>0?'#5ce0c4':isSel?'#c4a3ff':'rgba(255,255,255,0.22)'}" stroke-width="${isSel||done?2:1}"/>`;
  const nX=(BW-nW)/2;
  s+=`<rect x="${nX}" y="2" width="${nW}" height="${nH+6}" rx="5" fill="rgba(255,255,255,0.10)" stroke="${isSel?'#c4a3ff':done&&layers.length>0?'#5ce0c4':'rgba(255,255,255,0.22)'}" stroke-width="${isSel||done?2:1}"/>`;
  const corkW=nW+6,corkH=10,corkX=(BW-corkW)/2;
  s+=`<rect x="${corkX}" y="0" width="${corkW}" height="${corkH}" rx="3" fill="#c8a265" stroke="#8c6030" stroke-width="0.8"/>`;
  s+=`<line x1="${corkX+4}" y1="2" x2="${corkX+4}" y2="8" stroke="#a07840" stroke-width="0.7"/>`;
  s+=`<line x1="${corkX+8}" y1="2" x2="${corkX+8}" y2="8" stroke="#a07840" stroke-width="0.7"/>`;
  s+=`<rect x="${bX+6}" y="${bY+4}" width="5" height="${bH-10}" rx="3" fill="rgba(255,255,255,0.18)"/>`;
  if(done&&layers.length>0){s+=`<circle cx="${BW/2}" cy="${bY-12}" r="8" fill="#5ce0c4" opacity="0.9"/><text x="${BW/2}" y="${bY-8}" text-anchor="middle" font-size="10" fill="#fff">✓</text>`;}
  return `<svg width="${BW}" height="${BH}" viewBox="0 0 ${BW} ${BH}" xmlns="http://www.w3.org/2000/svg">${s}</svg>`;
}

function render(){
  const grid=document.getElementById('bottles-grid');
  grid.innerHTML='';
  bottles.forEach((layers,idx)=>{
    const isSel=selected===idx;const done=isDone(layers);
    const wrap=document.createElement('div');
    wrap.className='bwrap'+(isSel?' sel':'');
    wrap.dataset.idx=idx;
    wrap.innerHTML=drawBottleSVG(layers,isSel,done);
    wrap.addEventListener('click',()=>onClick(idx));
    grid.appendChild(wrap);
  });
}

function onClick(idx){
  if(selected===null){
    if(bottles[idx].length===0){setMsg('빈 병은 선택할 수 없어요');return;}
    if(isDone(bottles[idx])){setMsg('이미 완성된 병이에요');return;}
    selected=idx;render();
  } else {
    if(selected===idx){selected=null;render();return;}
    if(canPour(selected,idx)){
      history.push(clone(bottles));if(history.length>5)history.shift();
      const fi=selected;selected=null;
      const fromEl=document.querySelector(`.bwrap[data-idx="${fi}"]`);
      if(fromEl){fromEl.classList.add('pour');setTimeout(()=>fromEl.classList.remove('pour'),350);}
      setTimeout(()=>{doPour(fi,idx);moves++;render();if(checkWin())setTimeout(showWin,400);},200);
    } else {
      const fromEl=document.querySelector(`.bwrap[data-idx="${selected}"]`);
      if(fromEl){fromEl.classList.remove('shake');void fromEl.offsetWidth;fromEl.classList.add('shake');setTimeout(()=>fromEl.classList.remove('shake'),400);}
      setMsg('여기에 부을 수 없어요');
    }
  }
}

function showWin(){
  const s=moves<=(levelIdx+3)*4?'⭐⭐⭐':moves<=(levelIdx+3)*6?'⭐⭐':'⭐';
  document.getElementById('win-stars').textContent=s;
  document.getElementById('win-s').textContent=`Level ${levelIdx+1} 클리어! · 이동 ${moves}회`;
  document.getElementById('win-overlay').classList.add('show');
}

function initLevel(li){
  levelIdx=li;bottles=genLevel(li);selected=null;history=[];moves=0;
  document.getElementById('level-txt').textContent=`Level ${li+1}`;
  document.getElementById('win-overlay').classList.remove('show');
  setMsg('병을 터치해 같은 색끼리 정렬하세요!',3000);
  render();
}

document.getElementById('undo-btn').addEventListener('click',()=>{if(!history.length){setMsg('더 이상 되돌릴 수 없어요');return;}bottles=history.pop();selected=null;moves=Math.max(0,moves-1);render();setMsg('↩ 되돌렸어요');});
document.getElementById('reset-btn').addEventListener('click',()=>{if(!history.length)return;bottles=history[0];history=[];selected=null;moves=0;render();setMsg('↺ 처음부터 다시!');});
document.getElementById('next-btn').addEventListener('click',()=>{initLevel((levelIdx+1)%LEVELS.length);});

initLevel(0);
</script>
</body>
</html>
