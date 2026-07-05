<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>원 그리기 인식 데모</title>
<style>
body{font-family:sans-serif;background:#111;color:#eee;text-align:center;padding:20px}
canvas{background:#f2ede4;touch-action:none;cursor:crosshair;border:1px solid #444}
#result{margin-top:14px;font-size:15px;line-height:1.8}
button{margin-top:10px;padding:8px 16px;cursor:pointer}
</style>
</head>
<body>
<h2>원 그리기 인식 테스트 (가이드 없음)</h2>
<canvas id="c" width="600" height="380"></canvas>
<div id="result">아무 곳에서나 자유롭게 원을 그려보세요</div>
<button onclick="reset()">다시 그리기</button>

<script>
// ============================================================
// 1. 캔버스 & 입력 수집 (가이드 없음, 자유 드로잉)
// ============================================================
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
let drawn = [];
let drawing = false;
let lastResult = null;

function getPos(e){
  const r = canvas.getBoundingClientRect();
  const s = e.touches ? e.touches[0] : e;
  return { x: s.clientX - r.left, y: s.clientY - r.top, t: performance.now() };
}
canvas.addEventListener('pointerdown', e=>{
  drawing = true; drawn = [getPos(e)]; lastResult = null;
  canvas.setPointerCapture(e.pointerId);
  drawBackground(); 
});
canvas.addEventListener('pointermove', e=>{
  if(!drawing) return;
  drawn.push(getPos(e));
  render();
});
canvas.addEventListener('pointerup', e=>{
  if(!drawing) return;
  drawing = false;
  if(drawn.length > 12) judge();
  else document.getElementById('result').textContent = '너무 짧아요. 원을 한 바퀴 그려보세요';
});

function drawBackground(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle='#f2ede4'; ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.strokeStyle='rgba(0,0,0,.04)'; ctx.lineWidth=.5;
  for(let x=0;x<canvas.width;x+=24){ctx.beginPath();ctx.moveTo(x,0);ctx.lineTo(x,canvas.height);ctx.stroke();}
  for(let y=0;y<canvas.height;y+=24){ctx.beginPath();ctx.moveTo(0,y);ctx.lineTo(canvas.width,y);ctx.stroke();}
}
function drawPath(pts, color, width, dash){
  ctx.strokeStyle=color; ctx.lineWidth=width; ctx.lineCap='round'; ctx.lineJoin='round';
  if(dash) ctx.setLineDash(dash); else ctx.setLineDash([]);
  ctx.beginPath(); ctx.moveTo(pts[0].x,pts[0].y);
  for(let i=1;i<pts.length;i++) ctx.lineTo(pts[i].x,pts[i].y);
  ctx.stroke(); ctx.setLineDash([]);
}
function render(){
  drawBackground();
  if(drawn.length>1) drawPath(drawn, '#0a0a0f', 2.5);
  if(lastResult){
    // 판정 후: 사용자가 그린 원에 가장 가까운 '이상적인 원'을 겹쳐 보여줌
    const {cx, cy, r} = lastResult.fit;
    ctx.strokeStyle='rgba(39,174,96,.8)'; ctx.lineWidth=2; ctx.setLineDash([6,5]);
    ctx.beginPath(); ctx.arc(cx, cy, r, 0, Math.PI*2); ctx.stroke(); ctx.setLineDash([]);
    ctx.fillStyle='rgba(39,174,96,.9)';
    ctx.beginPath(); ctx.arc(cx, cy, 3, 0, Math.PI*2); ctx.fill();

    // 튀어나온 방향(빨강) / 들어간 방향(파랑) 화살표
    const b = lastResult.bulge;
    if(b.outHour!=null) drawDirArrow(cx, cy, r, b.outAngleFromTop, '#e74c3c', '+'+b.outAmount);
    if(b.inHour!=null)  drawDirArrow(cx, cy, r, b.inAngleFromTop,  '#3498db', ''+b.inAmount);
  }
}
// 중심에서 바깥으로, 각도(윗쪽=0, 시계방향)만큼 회전한 방향에 화살표+라벨 그리기
function drawDirArrow(cx, cy, r, angleFromTop, color, label){
  const rad = angleFromTop * Math.PI/180;
  const dx = Math.sin(rad), dy = -Math.cos(rad); // 0도=위쪽
  const x1 = cx + dx*(r*0.3), y1 = cy + dy*(r*0.3);
  const x2 = cx + dx*(r*1.25), y2 = cy + dy*(r*1.25);
  ctx.strokeStyle = color; ctx.fillStyle = color; ctx.lineWidth = 2.5; ctx.lineCap='round';
  ctx.beginPath(); ctx.moveTo(x1,y1); ctx.lineTo(x2,y2); ctx.stroke();
  const ah = 8, aang = Math.atan2(y2-y1, x2-x1);
  ctx.beginPath();
  ctx.moveTo(x2, y2);
  ctx.lineTo(x2-ah*Math.cos(aang-Math.PI/6), y2-ah*Math.sin(aang-Math.PI/6));
  ctx.lineTo(x2-ah*Math.cos(aang+Math.PI/6), y2-ah*Math.sin(aang+Math.PI/6));
  ctx.closePath(); ctx.fill();
  ctx.font='bold 11px sans-serif';
  ctx.fillText(label, x2+dx*10-8, y2+dy*10+4);
}

// ============================================================
// 2. 최소자승법으로 사용자가 그린 점들에 가장 가까운 원 찾기 (Kasa fit)
//    → 중심(cx,cy)과 반지름(r) 추정
// ============================================================
function fitCircle(pts){
  const n = pts.length;
  let mx=0, my=0;
  pts.forEach(p=>{mx+=p.x; my+=p.y;});
  mx/=n; my/=n;

  let Suu=0, Suv=0, Svv=0, Suuu=0, Svvv=0, Suvv=0, Svuu=0;
  pts.forEach(p=>{
    const u=p.x-mx, v=p.y-my;
    Suu+=u*u; Suv+=u*v; Svv+=v*v;
    Suuu+=u*u*u; Svvv+=v*v*v; Suvv+=u*v*v; Svuu+=v*u*u;
  });
  const A = Suu, B = Suv, C = Suv, D = Svv;
  const E = 0.5*(Suuu+Suvv), F = 0.5*(Svvv+Svuu);
  const det = A*D - B*C;
  if(Math.abs(det) < 1e-9) return { cx:mx, cy:my, r: 0 };
  const uc = (E*D - B*F)/det;
  const vc = (A*F - E*C)/det;
  const cx = uc+mx, cy = vc+my;
  const r = Math.sqrt(uc*uc + vc*vc + (Suu+Svv)/n);
  return { cx, cy, r };
}

// ============================================================
// 3. 채점 로직 (핵심: 원 인식)
// ============================================================

// 원형도: 각 점이 추정 중심에서 추정 반지름만큼 떨어져 있는지 (나쁜 쪽 35%만 반영)
function calcRoundness(pts, cx, cy, r){
  if(r < 1) return 0;
  const diffs = pts.map(p => Math.abs(Math.hypot(p.x-cx, p.y-cy) - r));
  diffs.sort((a,b)=>b-a);
  const k = Math.max(1, Math.round(diffs.length*0.35));
  const worst = diffs.slice(0,k).reduce((a,b)=>a+b,0)/k;
  const ratio = worst / r; // 반지름에 대한 상대 오차
  return Math.min(100, Math.max(0, Math.round(100 - ratio*260)));
}

// 완성도: 중심 기준 각도로 봤을 때 360도를 얼마나 빠짐없이 훑었는지
// (한 바퀴 다 그렸는지, 중간에 끊긴 부분은 없는지)
function calcCoverage360(pts, cx, cy){
  const angles = pts.map(p => {
    let a = Math.atan2(p.y-cy, p.x-cx) * 180/Math.PI;
    if(a < 0) a += 360;
    return a;
  }).sort((a,b)=>a-b);
  let maxGap = 0;
  for(let i=1;i<angles.length;i++) maxGap = Math.max(maxGap, angles[i]-angles[i-1]);
  maxGap = Math.max(maxGap, 360 - angles[angles.length-1] + angles[0]); // 마지막→처음 랩어라운드
  return Math.min(100, Math.max(0, Math.round(100 - maxGap/1.8)));
}

// 마감(닫힘): 시작점과 끝점이 반지름 대비 얼마나 가까이 맞물렸는지
function calcClosure(pts, r){
  if(r < 1) return 0;
  const gap = Math.hypot(pts[0].x-pts[pts.length-1].x, pts[0].y-pts[pts.length-1].y);
  const ratio = gap / r;
  return Math.min(100, Math.max(0, Math.round(100 - ratio*70)));
}

// 돌출 방향: 이상적인 원보다 어느 쪽으로 튀어나왔는지(볼록) / 들어갔는지(오목)를 시계 방향으로 표시
// 12방향(시계)으로 각도를 나눠 구간별 평균 오차(거리-반지름)를 구하고, 가장 큰(+)/가장 작은(-) 구간을 찾음
function calcBulgeDirection(pts, cx, cy, r){
  const bins = new Array(12).fill(0).map(()=>({ sum:0, count:0 }));
  pts.forEach(p=>{
    const dx = p.x-cx, dy = p.y-cy;
    let angleFromTop = Math.atan2(dx, -dy) * 180/Math.PI; // 0=12시, 시계방향 증가
    if(angleFromTop < 0) angleFromTop += 360;
    const bin = Math.floor(angleFromTop/30) % 12;
    const dev = Math.hypot(dx,dy) - r; // +면 바깥으로 튀어나옴, -면 안으로 들어감
    bins[bin].sum += dev; bins[bin].count++;
  });
  const avgs = bins.map(b => b.count ? b.sum/b.count : null);
  let outIdx=-1, outVal=-Infinity, inIdx=-1, inVal=Infinity;
  avgs.forEach((v,i)=>{
    if(v===null) return;
    if(v>outVal){ outVal=v; outIdx=i; }
    if(v<inVal){ inVal=v; inIdx=i; }
  });
  const hourOf = i => (i+1); // bin0=12시~1시 구간 → 대표 1시, bin11=11시~12시 → 12시
  return {
    outHour: outIdx>=0 ? (outIdx===11?12:outIdx+1) : null, outAmount: outIdx>=0 ? Math.round(outVal) : 0,
    inHour:  inIdx>=0  ? (inIdx===11?12:inIdx+1)   : null, inAmount:  inIdx>=0  ? Math.round(inVal)  : 0,
    outAngleFromTop: outIdx>=0 ? outIdx*30+15 : null,
    inAngleFromTop:  inIdx>=0  ? inIdx*30+15  : null
  };
}

// 속도 점수(0~100): 구간별 px/s 계산 후 60퍼센타일 값으로 평가 (빠를수록 높음)
function calcSpeed(pts){
  if(pts.length < 4) return 0;
  const vels = [];
  for(let i=1;i<pts.length;i++){
    const dt = (pts[i].t - pts[i-1].t)/1000;
    if(dt<=0) continue;
    const d = Math.hypot(pts[i].x-pts[i-1].x, pts[i].y-pts[i-1].y);
    vels.push(d/dt);
  }
  if(!vels.length) return 0;
  vels.sort((a,b)=>a-b);
  const v = vels[Math.min(vels.length-1, Math.floor(vels.length*0.6))];
  return Math.min(100, Math.max(0, Math.round(v/6)));
}

// ============================================================
// 4. 종합 판정
// ============================================================
function judge(){
  const fit = fitCircle(drawn);
  const roundness = calcRoundness(drawn, fit.cx, fit.cy, fit.r);
  const coverage  = calcCoverage360(drawn, fit.cx, fit.cy);
  const closure   = calcClosure(drawn, fit.r);
  const speed     = calcSpeed(drawn);
  const bulge     = calcBulgeDirection(drawn, fit.cx, fit.cy, fit.r);

  // 원형도(형태)가 핵심, 완성도·마감은 보조, 마지막에 속도 배율 적용
  let score = Math.round(roundness*0.55 + coverage*0.25 + closure*0.20);
  const spdMulti = speed>=80?1.00 : speed>=65?0.92 : speed>=50?0.85 : 0.80;
  score = Math.min(100, Math.max(0, Math.round(score * spdMulti)));

  lastResult = { fit, roundness, coverage, closure, speed, score, bulge };
  render();

  const spdColor = speed>=80?'#4fc3f7':speed>=50?'#ffb74d':'#e57373';
  const bulgeLine = bulge.outHour!=null
    ? `<span style="color:#e57373">${bulge.outHour}시 방향으로 +${bulge.outAmount}px 튀어나옴</span> · <span style="color:#64b5f6">${bulge.inHour}시 방향으로 ${bulge.inAmount}px 들어감</span>`
    : '';
  document.getElementById('result').innerHTML =
    `<b style="font-size:24px;color:${score>=80?'#4caf50':score>=60?'#ffb74d':'#e57373'}">${score}점</b>
     <span style="font-size:13px;color:${spdColor}"> (속도 ${speed})</span><br>
     원형도 ${roundness} · 완성도 ${coverage} · 마감 ${closure} · 속도 ${speed}<br>
     ${bulgeLine}<br>
     <span style="font-size:12px;color:#888">추정 반지름 ${Math.round(fit.r)}px (초록 점선 = 가장 가까운 이상적 원)</span>`;
}

function reset(){
  drawn = []; lastResult = null;
  drawBackground();
  document.getElementById('result').textContent = '아무 곳에서나 자유롭게 원을 그려보세요';
}
reset();
</script>
</body>
</html>
