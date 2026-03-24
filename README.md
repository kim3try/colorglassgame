<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>컬러 소트 퍼즐 - 리셋 기능 수정</title>
<style>
*{box-sizing:border-box;margin:0;padding:0;}
body{background:#1a1040;display:flex;justify-content:center;align-items:flex-start;min-height:100vh;padding:20px;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;}
#root{background:#2d2456;min-height:620px;width:100%;max-width:480px;border-radius:20px;overflow:hidden;position:relative;box-shadow: 0 20px 50px rgba(0,0,0,0.5);}
#topbar{display:flex;justify-content:space-between;align-items:center;padding:16px 20px 8px;background:rgba(0,0,0,0.2);}
#level-txt{font-size:16px;font-weight:600;color:#fff;letter-spacing: 0.5px;}
#btn-row{display:flex;gap:8px;}
.hbtn{background:rgba(255,255,255,0.15);border:none;border-radius:20px;padding:8px 16px;font-size:13px;color:#fff;cursor:pointer;transition:all 0.2s;display:flex;align-items:center;gap:4px;}
.hbtn:hover{background:rgba(255,255,255,0.25);transform: translateY(-1px);}
.hbtn:active{transform: translateY(0);}
#msg{text-align:center;font-size:13px;color:rgba(255,255,255,0.7);min-height:22px;padding:10px 0 4px;font-weight: 300;}
#bottles-grid{display:flex;flex-wrap:wrap;justify-content:center;align-items:flex-end;gap:12px;padding:10px 16px 24px;}
.bwrap{display:flex;flex-direction:column;align-items:center;cursor:pointer;transition:transform 0.2s cubic-bezier(.34,1.56,.64,1);}
.bwrap.sel{transform:translateY(-18px);}
.bwrap.shake{animation:shk 0.4s;}
@keyframes shk{0%,100%{transform:translateX(0)}20%{transform:translateX(-7px)}40%{transform:translateX(7px)}60%{transform:translateX(-4px)}80%{transform:translateX(4px)}}
.bwrap.pour{animation:pr 0.35s ease-in-out;}
@keyframes pr{0%{transform:rotate(0)}40%{transform:rotate(-20deg) translateY(-10px)}100%{transform:rotate(0)}}
#win-overlay{display:none;position:absolute;inset:0;background:rgba(20,15,50,0.8);align-items:center;justify-content:center;z-index:20;border-radius:20px;backdrop-filter: blur(4px);}
#win-overlay.show{display:flex;}
#win-card{background:#3a2f7a;border:2px solid rgba(255,255,255,0.15);border-radius:24px;padding:2.5rem;text-align:center;box-shadow: 0 15px 40px rgba(0,0,0,0.4);}
#win-stars{font-size:32px;margin-bottom:10px;letter-spacing:8px;color: #ffd700;}
#win-t{font-size:24px;font-weight:700;color:#fff;margin-bottom:6px;}
#win-s{font-size:14px;color:rgba(255,255,255,0.6);margin-bottom:1.5rem;}
#next-btn{background:linear-gradient(135deg,#5dcaa5,#1d9e75);border:none;border-radius:24px;padding:12px 32px;font-size:16px;color:#fff;cursor:pointer;font-weight:600;box-shadow: 0 4px 15px rgba(29,158,117,0.3);}
#next-btn:hover{filter: brightness(1.1);transform: scale(1.05);}
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
      <div id="win-t">참 잘했어요!</div>
      <div id="win-s"></div>
      <button id="next-btn">다음 레벨 시작</button>
    </div>
  </div>
</div>

<script>
/**
 * 컬러 정의
 */
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

/**
 * 레벨 난이도 설정
 */
const LEVELS=[
  {colorCount:3, emptyExtra:1, shuffles: 40},
  {colorCount:4, emptyExtra:1, shuffles: 60},
  {colorCount:4, emptyExtra:2, shuffles: 80},
  {colorCount:5, emptyExtra:2, shuffles: 120},
  {colorCount:6, emptyExtra:2, shuffles: 150},
  {colorCount:7, emptyExtra:2, shuffles: 200},
  {colorCount:8, emptyExtra:2, shuffles: 250},
];

const MAX=4;
let bottles=[], initialBottles=[], selected=null, history=[], levelIdx=0, moves=0;

function colorByKey(k){return COLORS.find(c=>c.key===k);}

/**
 * 역순 뒤섞기 알고리즘 (Solvable Level Generation)
 */
function genSolvableLevel(li) {
  const cfg = LEVELS[li];
  const colorKeys = COLORS.slice(0, cfg.colorCount).map(c => c.key);
  let result = [];
  for (let i = 0; i < cfg.colorCount; i++) {
    result.push(new Array(MAX).fill(colorKeys[i]));
  }
  for (let i = 0; i < cfg.emptyExtra; i++) {
    result.push([]);
  }

  let successfulShuffles = 0;
  const targetShuffles = cfg.shuffles;
  const totalBottles = result.length;

  while (successfulShuffles < targetShuffles) {
    let fromIdx = Math.floor(Math.random() * totalBottles);
    let toIdx = Math.floor(Math.random() * totalBottles);
    if (fromIdx !== toIdx && result[fromIdx].length > 0 && result[toIdx].length < MAX) {
      const color = result[fromIdx].pop();
      result[toIdx].push(color);
      successfulShuffles++;
    }
  }
  return result;
}

function clone(b){return b.map(x=>[...x]);}

/**
 * 게임 규칙 로직
 */
function canPour(fi, ti) {
  if (fi === ti) return false;
  const source = bottles[fi];
  const target = bottles[ti];
  if (source.length === 0) return false;
  if (target.length >= MAX) return false;
  if (target.length === 0) return true;
  return source[source.length - 1] === target[target.length - 1];
}

function doPour(fi, ti) {
  const topColor = bottles[fi][bottles[fi].length - 1];
  while (
    bottles[fi].length > 0 && 
    bottles[fi][bottles[fi].length - 1] === topColor && 
    bottles[ti].length < MAX
  ) {
    bottles[ti].push(bottles[fi].pop());
  }
}

function isBottleDone(b) {
  if (b.length === 0) return true;
  if (b.length !== MAX) return false;
  return b.every(c => c === b[0]);
}

function checkWin() {
  return bottles.every(isBottleDone);
}

function setMsg(t, d=1800) {
  const e = document.getElementById('msg');
  e.textContent = t;
  if (d > 0) setTimeout(() => { if (e.textContent === t) e.textContent = ''; }, d);
}

/**
 * 그래픽 렌더링 (SVG)
 */
function drawBottleSVG(layers, isSel, done) {
  const BW=54, BH=130;
  const nW=22, nH=24, bW=BW-8, bH=BH-nH-18;
  const lH=bH/MAX;
  const bX=4, bY=nH+8;
  const cpId='cp'+Math.random().toString(36).slice(2);
  let s=`<defs><clipPath id="${cpId}"><rect x="${bX}" y="${bY}" width="${bW}" height="${bH}" rx="10"/></clipPath></defs>`;
  s+=`<rect x="${bX}" y="${bY}" width="${bW}" height="${bH}" rx="10" fill="rgba(255,255,255,0.08)" stroke="rgba(255,255,255,0.15)" stroke-width="1.5"/>`;
  for(let i=0; i<MAX; i++){
    const colorKey = layers[i];
    if(!colorKey) continue;
    const c = colorByKey(colorKey);
    const ly = bY + bH - (i+1)*lH;
    s+=`<rect x="${bX}" y="${ly}" width="${bW}" height="${lH+1}" fill="${c.fill}" clip-path="url(#${cpId})"/>`;
    s+=`<rect x="${bX}" y="${ly}" width="${bW}" height="2" fill="rgba(255,255,255,0.1)" clip-path="url(#${cpId})"/>`;
  }
  const strokeCol = done && layers.length > 0 ? '#5ce0c4' : (isSel ? '#c4a3ff' : 'rgba(255,255,255,0.25)');
  s+=`<rect x="${bX}" y="${bY}" width="${bW}" height="${bH}" rx="10" fill="none" stroke="${strokeCol}" stroke-width="${isSel||done?2.5:1.2}"/>`;
  const nX=(BW-nW)/2;
  s+=`<rect x="${nX}" y="4" width="${nW}" height="${nH+4}" rx="6" fill="rgba(255,255,255,0.08)" stroke="${strokeCol}" stroke-width="${isSel||done?2.5:1.2}"/>`;
  const corkW=nW+4, corkH=8, corkX=(BW-corkW)/2;
  s+=`<rect x="${corkX}" y="0" width="${corkW}" height="${corkH}" rx="3" fill="#b18c53" stroke="#7a5320" stroke-width="1"/>`;
  s+=`<rect x="${bX+6}" y="${bY+6}" width="4" height="${bH-12}" rx="2" fill="rgba(255,255,255,0.12)"/>`;
  if(done && layers.length > 0){
    s+=`<circle cx="${BW/2}" cy="${bY-15}" r="9" fill="#5ce0c4"/>
        <text x="${BW/2}" y="${bY-11}" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a5245">✓</text>`;
  }
  return `<svg width="${BW}" height="${BH}" viewBox="0 0 ${BW} ${BH}" xmlns="http://www.w3.org/2000/svg">${s}</svg>`;
}

function render() {
  const grid = document.getElementById('bottles-grid');
  grid.innerHTML = '';
  bottles.forEach((layers, idx) => {
    const isSel = selected === idx;
    const done = isBottleDone(layers);
    const wrap = document.createElement('div');
    wrap.className = 'bwrap' + (isSel ? ' sel' : '');
    wrap.dataset.idx = idx;
    wrap.innerHTML = drawBottleSVG(layers, isSel, done);
    wrap.addEventListener('click', () => onClick(idx));
    grid.appendChild(wrap);
  });
}

function onClick(idx) {
  if (selected === null) {
    if (bottles[idx].length === 0) { setMsg('빈 병이에요!'); return; }
    if (isBottleDone(bottles[idx]) && bottles[idx].length > 0) { setMsg('이미 정리된 병이에요'); return; }
    selected = idx;
    render();
  } else {
    if (selected === idx) { selected = null; render(); return; }
    if (canPour(selected, idx)) {
      history.push(clone(bottles));
      if (history.length > 15) history.shift();
      const fi = selected;
      selected = null;
      const fromEl = document.querySelector(`.bwrap[data-idx="${fi}"]`);
      if (fromEl) {
        fromEl.classList.add('pour');
        setTimeout(() => fromEl.classList.remove('pour'), 350);
      }
      setTimeout(() => {
        doPour(fi, idx);
        moves++;
        render();
        if (checkWin()) setTimeout(showWin, 500);
      }, 200);
    } else {
      const fromEl = document.querySelector(`.bwrap[data-idx="${selected}"]`);
      if (fromEl) {
        fromEl.classList.remove('shake');
        void fromEl.offsetWidth;
        fromEl.classList.add('shake');
        setTimeout(() => fromEl.classList.remove('shake'), 400);
      }
      setMsg('여기에 부을 수 없어요');
    }
  }
}

function showWin() {
  const s = moves <= (LEVELS[levelIdx].colorCount * 5) ? '⭐⭐⭐' : moves <= (LEVELS[levelIdx].colorCount * 8) ? '⭐⭐' : '⭐';
  document.getElementById('win-stars').textContent = s;
  document.getElementById('win-s').textContent = `Level ${levelIdx+1} 성공! 총 ${moves}번 이동함`;
  document.getElementById('win-overlay').classList.add('show');
}

/**
 * 레벨 초기화 및 시작
 */
function initLevel(li) {
  levelIdx = li;
  // 새로운 레벨 데이터 생성
  bottles = genSolvableLevel(li);
  // [추가] 초기 상태 저장 (리셋 시 사용)
  initialBottles = clone(bottles);
  
  selected = null;
  history = [];
  moves = 0;
  
  document.getElementById('level-txt').textContent = `Level ${li+1}`;
  document.getElementById('win-overlay').classList.remove('show');
  setMsg('색깔을 깔끔하게 정리해 보세요!', 2500);
  render();
}

/**
 * 리셋 기능: 현재 레벨의 "시작 상태"로 되돌림
 */
function resetCurrentLevel() {
  if (moves === 0) {
    setMsg('이미 시작 상태입니다');
    return;
  }
  // 저장해둔 초기 상태로 복구
  bottles = clone(initialBottles);
  selected = null;
  history = [];
  moves = 0;
  render();
  setMsg('↺ 처음 상태로 리셋되었습니다');
}

/**
 * 이벤트 리스너
 */
document.getElementById('undo-btn').addEventListener('click', () => {
  if (!history.length) { setMsg('더 이상 되돌릴 수 없어요'); return; }
  bottles = history.pop();
  selected = null;
  moves = Math.max(0, moves - 1);
  render();
  setMsg('↩ 한 수 무르기');
});

document.getElementById('reset-btn').addEventListener('click', () => {
  resetCurrentLevel();
});

document.getElementById('next-btn').addEventListener('click', () => {
  initLevel((levelIdx + 1) % LEVELS.length);
});

// 초기 실행
window.onload = () => initLevel(0);
</script>
</body>
</html>
