[Uploading index.html…]()
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>분수 문제 생성기 • 그림 클릭 선택 & 제출 시 정답 표시</title>
<style>
  :root{ --bg:#f7fafc; --ink:#0f172a; --muted:#475569; --line:#e2e8f0; --accent:#2563eb; --good:#34d399; --warn:#f59e0b; }
  html,body{margin:0;height:100%;font-family:ui-sans-serif,-apple-system,Segoe UI,Roboto,"Noto Sans KR",Helvetica,Arial,sans-serif;color:var(--ink);background:var(--bg)}
  .wrap{max-width:1100px;margin:20px auto;padding:12px;position:relative}
  .grid{display:grid;grid-template-columns:1fr 320px;gap:12px}
  .board{background:white;border:1px solid var(--line);border-radius:16px;box-shadow:0 2px 4px rgb(0 0 0/7%);padding:12px}
  h1{font-size:18px;margin:0 0 8px}
  .controls{display:flex;flex-wrap:wrap;gap:8px;margin:6px 0 10px}
  button,select,input[type=text]{border:1px solid var(--line);background:white;border-radius:12px;padding:8px 12px;box-shadow:0 1px 2px rgb(0 0 0/6%);font-size:14px}
  button{cursor:pointer}
  button:hover{background:#f8fafc}
  .problem{font-size:20px;font-weight:700;margin:8px 0 6px}
  .hint{font-size:13px;color:var(--muted)}
  .canvas{width:100%;border:1px dashed var(--line);border-radius:12px;display:flex;align-items:flex-start;justify-content:center;background:white;margin-bottom:10px;overflow:hidden}
  .answer{margin-top:8px;padding:10px;border:1px solid var(--line);border-radius:12px;background:#f8fafc}
  .feedback{margin-top:8px;font-weight:600}
.score{position:absolute;right:12px;top:12px;background:#eef2ff;border:1px solid #c7d2fe;color:#334155;border-radius:12px;padding:6px 10px;font-weight:600}
</style>
</head>
<body>
<div class="wrap">
  <div id="score" class="score">맞춘 문제: 0개</div>
  <h1>분수 문제 생성기 <span class="hint">그림 클릭으로 선택 · 제출 시 정답 색칠</span></h1>
  <div class="grid">
    <div class="board">
      <div id="problem" class="problem">문제가 여기에 표시됩니다</div>
      <div class="controls">
        <button id="btnNew">새 문제</button>
        <select id="level">
          <option value="easy">난이도: 쉬움 (1/2, 1/3, 1/4)</option>
          <option value="normal" selected>난이도: 보통 (분자가 1 또는 2)</option>
          <option value="hard">난이도: 어려움 (그 외 진분수)</option>
        </select>
        <select id="itemType">
          <option value="any" selected>그림: 섞어서</option>
          <option value="bread">그림: 빵</option>
          <option value="doll">그림: 인형</option>
          <option value="apple">그림: 사과</option>
          <option value="star">그림: 별</option>
          <option value="block">그림: 블록</option>
        </select>
      </div>
      <div id="canvas" class="canvas"></div>
      <div>
        <input type="text" id="userAnswer" placeholder="정답(개수) 입력" style="width:140px;text-align:center" />
        <button id="btnSubmit">제출하기</button>
      </div>
      <div id="feedback" class="feedback"></div>
      <div class="answer" id="answerBox" style="display:none"></div>
    </div>
    <div class="board">
      <div class="hint" style="line-height:1.6">
        • 그림을 <b>클릭</b>하면 색이 켜졌다 꺼집니다(학생 선택). 새 문제를 누르면 <b>선택이 초기화</b>됩니다.<br/>
        • 제출하기를 누르면 <b>정답 공개 + 그림 정답 색칠 + 채점</b>이 동시에 이루어집니다.<br/>
        • 쉬움·보통: <b>분모 개수만큼</b> 가로 배치 / 어려움: <b>랜덤 배치</b>(한 줄 최대 10개). 총 개수는 <b>36 이하</b>이며 분모의 배수로만 출제됩니다.<br/>
        • 난이도: 쉬움(1/2·1/3·1/4) / 보통(분자 1·2) / 어려움(그 외 진분수).
      </div>
    </div>
  </div>
</div>

<script>
(function(){
  const canvas = document.getElementById('canvas');
  const problemEl = document.getElementById('problem');
  const answerBox = document.getElementById('answerBox');
  const btnNew = document.getElementById('btnNew');
  const levelSel = document.getElementById('level');
  const itemSel = document.getElementById('itemType');
  const userAnswer = document.getElementById('userAnswer');
  const btnSubmit = document.getElementById('btnSubmit');
  const feedback = document.getElementById('feedback');

  // 점수 집계
  let correctCount = 0;
  function updateScore(){
    const el = document.getElementById('score');
    if(el) el.textContent = `맞춘 문제: ${correctCount}개`;
  }

  const nouns = {
    bread: {name:'빵', draw: drawBread},
    doll:  {name:'인형', draw: drawDoll},
    apple: {name:'사과', draw: drawApple},
    star:  {name:'별', draw: drawStar},
    block: {name:'블록', draw: drawBlock},
  };
  const nounKeys = Object.keys(nouns);

  let state = null;            // {nounKey, total, num, den, answer}
  let studentSelected = [];    // [bool, ...] 길이=total
  let revealed = false;        // 제출 후 정답 공개 여부

  function rnd(a,b){ return Math.floor(Math.random()*(b-a+1))+a; }
  function pick(arr){ return arr[rnd(0,arr.length-1)]; }

  function newProblem(){
    feedback.textContent = '';
    userAnswer.value = '';
    answerBox.style.display='none';
    revealed = false;

    // 난이도별 분수 풀
    const level = levelSel.value;
    let fracPool = [];
    if (level==='easy') {
      // 쉬움: 1/2, 1/3, 1/4
      fracPool = [[1,2],[1,3],[1,4]];
    } else if (level==='normal') {
      // 보통: 분자가 1 또는 2인 모든 진분수 (분모 2~12)
      for (let den=2; den<=12; den++){
        fracPool.push([1,den]);
        if (den>=3) fracPool.push([2,den]);
      }
    } else {
      // 어려움: 나머지 진분수 (분자가 3 이상), 분모 2~12
      for (let den=2; den<=12; den++){
        for (let num=3; num<den; num++){
          fracPool.push([num,den]);
        }
      }
    }

    const [num, den] = pick(fracPool);
    // 총 개수: 분모의 배수, 36 이하, 최소 분모*2 이상
    const multiples = [];
    for (let t = den*2; t <= 36; t++) if (t % den === 0) multiples.push(t);
    const total = pick(multiples);

    const answer = total * num / den; // 정수 보장
    const nounKey = (itemSel.value==='any') ? pick(nounKeys) : itemSel.value;

    let layoutCols;
    if (level === 'hard') {
      layoutCols = Math.min(10, Math.max(2, Math.min(total, rnd(2,10))));
    } else {
      layoutCols = Math.max(1, den); // 분모만큼 가로 배치
    }

    state = {nounKey, total, num, den, answer, layoutCols};
    studentSelected = Array(total).fill(false);
    renderProblem(false);
  }

  function renderProblem(showAnswer){
    if(!state) return;
    const {nounKey, total, num, den, answer} = state;
    const noun = nouns[nounKey].name;
    problemEl.innerHTML = `${noun} ${total}개의 <b>${num}/${den}</b>는 몇 개인가요?`;

    // 레이아웃 계산 (분모 개수만큼 가로 배치)
    const cols = state.layoutCols || Math.max(1, den); // 분모 개수씩 가로 배치
    const rows = Math.ceil(total/cols); // total은 분모의 배수
    const cellW = 80, cellH = 80;
    const vbW = cols*cellW + 80;
    const vbH = rows*cellH + 80;   // 상하 여백

    canvas.innerHTML = '';
    const svg = svgEl('svg', {viewBox: `0 0 ${vbW} ${vbH}`, width:'100%', height:'100%'});
    canvas.appendChild(svg);
    canvas.style.height = (vbH + 20) + 'px';

    const startX = (vbW - cols*cellW)/2 + 40;
    const startY = (vbH - rows*cellH)/2 + 20;
    const draw = nouns[nounKey].draw;

    for(let i=0;i<total;i++){
      const r = Math.floor(i/cols), c = i%cols;
      const x = startX + c*cellW, y = startY + r*cellH;
      const selected = showAnswer ? (i < answer) : !!studentSelected[i];
      const g = svgEl('g', {transform:`translate(${x},${y})`});
      g.style.cursor = revealed ? 'default' : 'pointer';
      draw(g, selected);
      if(!revealed){
        // 학생 선택 토글
        g.addEventListener('click', ()=>{
          studentSelected[i] = !studentSelected[i];
          // 선택 수를 입력칸에 자동 반영 (선택 기반 풀이가 편하도록)
          const count = studentSelected.filter(Boolean).length;
          if(document.activeElement !== userAnswer){ userAnswer.value = String(count); }
          renderProblem(false);
        });
      }
      svg.appendChild(g);
    }

    if(showAnswer){
      answerBox.style.display='block';
      answerBox.innerHTML = `정답: <b>${answer}</b>개 <span class="hint">(${total} × ${num}/${den} = ${answer})</span>`;
    }
  }

  function checkAnswer(){
    if(!state) return;
    // 입력이 비어 있으면 학생 선택 개수를 자동 사용
    const selectedCount = studentSelected.filter(Boolean).length;
    if(userAnswer.value.trim()===''){ userAnswer.value = String(selectedCount); }

    const user = Number(userAnswer.value.trim());
    if(!Number.isFinite(user)){
      feedback.textContent = '숫자를 입력하세요.';
      feedback.style.color = 'var(--warn)';
      return;
    }
    const correct = state.answer;
    const ok = (user===correct);
    feedback.style.color = ok ? 'var(--good)' : 'red';
    feedback.textContent = ok ? `✅ 정답입니다! (선택: ${selectedCount}개)` : `❌ 아쉬워요. 정답은 ${correct}개 (선택: ${selectedCount}개)`;
    if(ok && !revealed){ correctCount++; updateScore(); }
    revealed = true;
    renderProblem(true);
  }

  // ===== SVG 유틸 & 그림 =====
  function svgEl(tag, attrs){ const el=document.createElementNS('http://www.w3.org/2000/svg',tag); for(const k in attrs) el.setAttribute(k, attrs[k]); return el; }
  function drawOutline(g){ const r=svgEl('rect',{x:0,y:0,width:70,height:70,rx:12,ry:12,fill:'white',stroke:'#e5e7eb'}); g.appendChild(r);} 
  function drawBread(g,on){ drawOutline(g); const body=svgEl('rect',{x:10,y:24,width:50,height:34,rx:10,ry:10,fill:on?'#fde68a':'#e5e7eb',stroke:'#0f172a','stroke-width':1}); const top=svgEl('path',{d:'M16,26 C16,14 54,14 54,26 Z',fill:on?'#f59e0b':'#cbd5e1',stroke:'#0f172a','stroke-width':1}); g.appendChild(body); g.appendChild(top);} 
  function drawDoll(g,on){ drawOutline(g); const head=svgEl('circle',{cx:35,cy:26,r:10,fill:on?'#93c5fd':'#e5e7eb',stroke:'#0f172a','stroke-width':1}); const body=svgEl('rect',{x:25,y:36,width:20,height:20,rx:6,fill:on?'#60a5fa':'#cbd5e1',stroke:'#0f172a','stroke-width':1}); g.appendChild(head); g.appendChild(body);} 
  function drawApple(g,on){ drawOutline(g); const a=svgEl('circle',{cx:30,cy:40,r:12,fill:on?'#fca5a5':'#e5e7eb',stroke:'#0f172a','stroke-width':1}); const b=svgEl('circle',{cx:42,cy:40,r:12,fill:on?'#f87171':'#cbd5e1',stroke:'#0f172a','stroke-width':1}); const stem=svgEl('rect',{x:34,y:22,width:2,height:8,fill:'#0f172a'}); const leaf=svgEl('ellipse',{cx:40,cy:24,rx:6,ry:3,fill:on?'#34d399':'#94a3b8',stroke:'#0f172a','stroke-width':1}); g.appendChild(a); g.appendChild(b); g.appendChild(stem); g.appendChild(leaf);} 
  function drawStar(g,on){ drawOutline(g); const star=svgEl('path',{d:'M35 14 L40 28 L56 28 L43 36 L48 50 L35 41 L22 50 L27 36 L14 28 L30 28 Z',fill:on?'#fbbf24':'#e5e7eb',stroke:'#0f172a','stroke-width':1}); g.appendChild(star);} 
  function drawBlock(g,on){ drawOutline(g); const cube=svgEl('rect',{x:18,y:22,width:34,height:34,rx:6,fill:on?'#86efac':'#e5e7eb',stroke:'#0f172a','stroke-width':1}); g.appendChild(cube);} 

  btnNew.addEventListener('click', newProblem);
  btnSubmit.addEventListener('click', checkAnswer);
  userAnswer.addEventListener('keydown', (e)=>{ if(e.key==='Enter') checkAnswer(); });
  levelSel.addEventListener('change', newProblem);
  itemSel.addEventListener('change', newProblem);

  updateScore();
  newProblem();
})();
</script>
</body>
</html>
