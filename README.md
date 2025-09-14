<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Desael - Digit Predator (Single File)</title>
  <style>
    /* Minimal Tailwind-like utility classes used inline for single-file hosting */
    :root{--bg:#0f1724;--card:#0b1220;--muted:#9aa4b2;--accent:#10b981;--danger:#ef4444}
    html,body{height:100%;margin:0;font-family:Inter,ui-sans-serif,system-ui,-apple-system,'Segoe UI',Roboto,'Helvetica Neue',Arial}
    body{background:linear-gradient(180deg,#071023 0%, #071a2a 100%);color:#e6eef6;padding:18px}
    .container{max-width:1100px;margin:0 auto}
    .card{background:var(--card);border-radius:12px;padding:14px;margin-bottom:12px;box-shadow:0 6px 20px rgba(2,6,23,0.6)}
    h1{font-size:20px;margin:0 0 8px}
    .controls{display:flex;gap:8px;flex-wrap:wrap}
    label{font-size:12px;color:var(--muted)}
    select,input{background:#071124;border:1px solid #0b2a3a;color:#dff1ff;padding:8px;border-radius:8px}
    button{background:linear-gradient(90deg,var(--accent),#06b6d4);border:none;color:#032; padding:8px 12px;border-radius:10px;cursor:pointer}
    .btn-danger{background:linear-gradient(90deg,var(--danger),#f97316);color:#fff}
    .bars{display:grid;grid-template-columns:repeat(10,1fr);gap:8px;margin-top:12px}
    .bar{background:#081526;border-radius:8px;padding:8px;text-align:center;position:relative}
    .bar .digit{font-weight:700;font-size:14px}
    .bar .pct{font-size:12px;color:var(--muted)}
    .bar .level{height:80px;background:linear-gradient(180deg,#05263a,#02131a);border-radius:6px;display:flex;align-items:flex-end;justify-content:center;margin-top:8px;overflow:hidden}
    .bar .fill{width:100%;height:0%;background:green;border-radius:6px 6px 0 0;transition:height 300ms ease, background 300ms}
    .highlight{outline:3px solid rgba(16,185,129,0.22)}
    .signal{display:flex;gap:12px;align-items:center;margin-top:12px}
    .signal .chip{padding:10px 12px;border-radius:10px;background:#042233}
    .log{height:150px;overflow:auto;background:#031221;padding:8px;border-radius:8px;margin-top:10px;font-size:13px}
    .footer{font-size:12px;color:var(--muted);margin-top:8px}
    .small{font-size:12px;color:var(--muted)}
  </style>
</head>
<body>
  <div class="container">
    <div class="card">
      <h1>Desael - Digit Predator (Ready for TiinyHost)</h1>
      <div class="small">Single-file HTML — drag & drop to TiinyHost. Connect to a Deriv digit stream or simulate ticks locally.</div>
    </div>

    <div class="card">
      <div class="controls">
        <div>
          <label>WebSocket URL</label><br>
          <input id="wsUrl" style="width:320px" placeholder="wss://ws.binaryws.com/websockets/v3?app_id=1089" />
        </div>
        <div>
          <label>Volatility / Market</label><br>
          <select id="volatility">
            <option>Random (simulated)</option>
            <option>Volatility 10</option>
            <option>Volatility 25</option>
            <option>Volatility 50</option>
          </select>
        </div>
        <div>
          <label>Trade type</label><br>
          <select id="tradeType">
            <option value="odd">Odd</option>
            <option value="even">Even</option>
            <option value="overunder">Over/Under</option>
            <option value="matches">Matches</option>
          </select>
        </div>
        <div>
          <label>Tick window (seconds)</label><br>
          <select id="tickWindow">
            <option>5</option>
            <option selected>10</option>
            <option>20</option>
          </select>
        </div>
        <div>
          <label>Signal sensitivity</label><br>
          <select id="sensitivity">
            <option value="low">Low</option>
            <option value="normal" selected>Normal</option>
            <option value="high">High</option>
          </select>
        </div>
        <div style="display:flex;align-items:flex-end;gap:8px">
          <button id="connectBtn">Start</button>
          <button id="stopBtn" class="btn-danger">Stop</button>
        </div>
      </div>

      <div class="signal" id="signalRow">
        <div class="chip" id="mainSignal">Signal: <strong>—</strong></div>
        <div class="chip" id="entryChip">Entry: —</div>
        <div class="chip" id="confidenceChip">Confidence: —</div>
        <div class="chip" id="timerChip">Timer: 0s</div>
      </div>

      <div class="bars" id="bars">
        <!-- bars 0-9 created by JS -->
      </div>

      <div class="log" id="log"></div>

      <div class="footer">Notes: Uses last 100 ticks (or fewer) to compute digit frequencies. Over/Under follows your custom rules where possible: "Under 5" prefers green on odd entry digits (5,7,9) and red on evens; "Over 4" is the inverse. Matches shows the most frequent repeat candidates.</div>
    </div>
  </div>

  <script>
    // Single-file Digit Predator logic
    const barsEl = document.getElementById('bars');
    const logEl = document.getElementById('log');
    const mainSignal = document.getElementById('mainSignal');
    const entryChip = document.getElementById('entryChip');
    const confidenceChip = document.getElementById('confidenceChip');
    const timerChip = document.getElementById('timerChip');

    // Setup bars
    const digits = Array.from({length:10},(_,i)=>i);
    const barState = {};
    digits.forEach(d=>{ barState[d]=0; const div = document.createElement('div'); div.className='bar'; div.id='bar-'+d; div.innerHTML=`<div class="digit">${d}</div><div class="pct">0%</div><div class="level"><div class="fill" style="height:0%"></div></div>`; barsEl.appendChild(div); });

    // Tick storage & params
    let ticks = [];
    const MAX_TICKS = 200;
    let ws = null;
    let running = false;
    let simulate = true; // default to simulate until user provides websocket
    let timer = 0; let timerInterval=null;

    const sensitivityMap = {low:0.06, normal:0.12, high:0.2};

    function log(msg){ const at = new Date().toLocaleTimeString(); logEl.innerHTML = `[${at}] ${msg}\n` + logEl.innerHTML; }

    function pushTick(d){ ticks.push({d, t:Date.now()}); if(ticks.length>MAX_TICKS) ticks.shift(); updateStats(); }

    function updateStats(){
      const counts = Array(10).fill(0);
      ticks.forEach(x=>counts[x.d]++);
      const total = Math.max(1, ticks.length);
      digits.forEach(d=>{
        const pct = Math.round( (counts[d]/total)*100 );
        const el = document.querySelector('#bar-'+d+' .pct');
        const fill = document.querySelector('#bar-'+d+' .fill');
        el.textContent = pct + '%';
        fill.style.height = Math.max(4, pct*0.8) + '%';
        // color fill green if high, red if low
        const thresh = 10; // visual
        if(pct >= 15) fill.style.background = 'linear-gradient(180deg,#10b981,#06b6d4)'; else fill.style.background = 'linear-gradient(180deg,#ef4444,#fb923c)';
      });
      computeSignal(counts, total);
    }

    function computeSignal(counts, total){
      // compute basic stats
      const pct = (d)=>counts[d]/Math.max(1,total);
      const oddTotal = digits.filter(d=>d%2===1).reduce((s,d)=>s+counts[d],0);
      const evenTotal = digits.filter(d=>d%2===0).reduce((s,d)=>s+counts[d],0);

      const tradeType = document.getElementById('tradeType').value;
      const sens = sensitivityMap[document.getElementById('sensitivity').value] || 0.12;

      // Highest digit candidate
      let topDigit = 0; for(let i=1;i<10;i++) if(counts[i]>counts[topDigit]) topDigit=i;
      const topPct = pct(topDigit);

      // Even/Odd signals
      if(tradeType==='odd' || tradeType==='even'){
        const wantOdd = tradeType==='odd';
        const sideTotal = wantOdd ? oddTotal : evenTotal;
        const sidePct = sideTotal/Math.max(1,total);
        let signal = sidePct > 0.6 ? (wantOdd? 'Strong Odd':'Strong Even') : (sidePct > 0.52 ? (wantOdd? 'Odd Fav':'Even Fav') : 'Neutral');
        mainSignal.innerHTML = `Signal: <strong>${signal}</strong>`;
        // entry candidate: top digit matching parity and preferred set
        let entry = null;
        // prefer digits 5/7/9 for Odd, 0/2/4 for Even per your rules
        const preferred = wantOdd ? [9,7,5] : [0,2,4];
        for(const d of preferred) if(counts[d] > 0) { entry = d; break; }
        if(entry===null){ // fallback: nearest top digit with parity
          for(let i=9;i>=0;i--){ if(i%2 === (wantOdd?1:0) && counts[i]>0){ entry=i; break; } }
        }
        entryChip.innerHTML = `Entry: ${entry===null? '—': entry}`;
        confidenceChip.innerHTML = `Confidence: ${(sidePct*100).toFixed(1)}%`;
        highlightEntry(entry);
        return;
      }

      if(tradeType==='overunder'){
        // Implement Under 5 and Over 4 logic from user's advanced rules (simplified and practical)
        // Under 5: prefer green on odd digits 5,7,9 and red on evens; require at least two odd digits above threshold
        const oddHigh = [5,7,9].filter(d=>pct(d) > sens);
        const evens = [0,2,4];
        const evensPresent = evens.some(d=>pct(d) > 0.05);
        const oddAbove = oddHigh.length;
        const totalOddPct = [5,7,9].reduce((s,d)=>s+pct(d),0);

        const evenHigh = [0,2,4].filter(d=>pct(d) > sens);
        const oddAbove4 = [5,7,9].filter(d=>pct(d) > sens);

        // Decide Under5 or Over4
        let decision = 'Neutral'; let entry=null; let conf=0;
        if(oddAbove >= 2 && totalOddPct > 0.25 && evensPresent){ decision='Under 5'; entry = oddHigh[0]; conf = totalOddPct; }
        else if(evenHigh.length >= 2 && ([5,7,9].reduce((s,d)=>s+counts[d],0)/Math.max(1,total)) < 0.4){ decision='Over 4'; entry = evenHigh[0]; conf = evenHigh.reduce((s,d)=>s+pct(d),0); }
        else {
          // fallback: compare even vs odd overall
          const oddSide = oddTotal/Math.max(1,total);
          const evenSide = evenTotal/Math.max(1,total);
          if(oddSide - evenSide > sens) { decision='Under 5'; entry = [5,7,9].find(d=>counts[d]); conf = oddSide; }
          else if(evenSide - oddSide > sens) { decision='Over 4'; entry = [0,2,4].find(d=>counts[d]); conf = evenSide; }
        }

        mainSignal.innerHTML = `Signal: <strong>${decision}</strong>`;
        entryChip.innerHTML = `Entry: ${entry===null? '—': entry}`;
        confidenceChip.innerHTML = `Confidence: ${(conf*100).toFixed(1)}%`;
        highlightEntry(entry);
        return;
      }

      if(tradeType==='matches'){
        // Show most frequent digits and probability of repeat
        const sorted = digits.slice().sort((a,b)=>counts[b]-counts[a]);
        const top = sorted[0]; const top2 = sorted[1];
        const prob = pct(top);
        mainSignal.innerHTML = `Signal: <strong>Match Candidate ${top}</strong>`;
        entryChip.innerHTML = `Entry: ${top}`;
        confidenceChip.innerHTML = `Confidence: ${(prob*100).toFixed(1)}%`;
        highlightEntry(top);
        return;
      }

      mainSignal.innerHTML = `Signal: <strong>—</strong>`;
      entryChip.innerHTML = `Entry: —`;
      confidenceChip.innerHTML = `Confidence: —`;
    }

    function highlightEntry(d){ // highlight one bar
      document.querySelectorAll('.bar').forEach(el=>el.classList.remove('highlight'));
      if(d===undefined || d===null) return;
      const el = document.getElementById('bar-'+d);
      if(el) el.classList.add('highlight');
    }

    // Timer display used to show remaining tick window
    function startTimer(){ clearInterval(timerInterval); timer=Number(document.getElementById('tickWindow').value); timerChip.textContent = `Timer: ${timer}s`;
      timerInterval = setInterval(()=>{ timer--; timerChip.textContent = `Timer: ${timer}s`; if(timer<=0){ timer = Number(document.getElementById('tickWindow').value); timerChip.textContent = `Timer: ${timer}s`; /* reset window */ } },1000);
    }
    function stopTimer(){ clearInterval(timerInterval); timerChip.textContent = `Timer: 0s`; }

    // WebSocket handling: expects messages containing a digit e.g. {"tick": {"quote": 123.45, "epoch": 163..., "id":... , "digit": 7 }} or user can set custom handling
    function connect(){
      const url = document.getElementById('wsUrl').value.trim();
      if(!url){ log('No websocket URL provided — running in simulated mode.'); simulate=true; startSimulation(); return; }
      try{
        ws = new WebSocket(url);
        ws.onopen = ()=>{ log('WebSocket connected'); simulate=false; };
        ws.onmessage = (ev)=>{
          try{
            const data = JSON.parse(ev.data);
            // Deriv sends ticks within data.tick and digit under data.tick.quote % 10 often - adapt to common shape
            let digit = null;
            if(data && data.tick && (typeof data.tick.quote === 'number')){
              digit = String(Math.floor(data.tick.quote)).slice(-1); digit = Number(digit);
            } else if(data && data.proposal && typeof data.proposal.display_value === 'number'){
              digit = data.proposal.display_value % 10;
            } else if(data && data.digit !== undefined) digit = Number(data.digit);
            if(digit===null || isNaN(digit)) return;
            pushTick(digit);
            log('Received tick: '+digit);
          }catch(e){ console.error(e); }
        };
        ws.onclose = ()=>{ log('WebSocket closed — switching to simulation'); startSimulation(); };
        ws.onerror = (e)=>{ log('WebSocket error'); console.error(e); }
      }catch(e){ log('Failed to connect websocket — simulation mode'); startSimulation(); }
    }

    let simInterval=null;
    function startSimulation(){ stopSimulation(); simInterval = setInterval(()=>{ const d = Math.floor(Math.random()*10); pushTick(d); }, 600); }
    function stopSimulation(){ if(simInterval) clearInterval(simInterval); simInterval=null; }

    document.getElementById('connectBtn').addEventListener('click', ()=>{
      if(running) return; running=true; document.getElementById('connectBtn').disabled=true; document.getElementById('stopBtn').disabled=false; startTimer(); connect(); log('Started');
    });
    document.getElementById('stopBtn').addEventListener('click', ()=>{
      running=false; document.getElementById('connectBtn').disabled=false; document.getElementById('stopBtn').disabled=true; if(ws) ws.close(); stopSimulation(); stopTimer(); log('Stopped');
    });

    // init
    document.getElementById('stopBtn').disabled=true;
    updateStats();

    // expose a download option for quick export (user can copy the page source) - provides a blob download
    function downloadHTML(){ const html = '<!doctype html>\n' + document.documentElement.outerHTML; const blob = new Blob([html], {type:'text/html'}); const url = URL.createObjectURL(blob); const a = document.createElement('a'); a.href = url; a.download = 'desael_digit_predator.html'; document.body.appendChild(a); a.click(); a.remove(); URL.revokeObjectURL(url); }

    // Keyboard shortcut: Ctrl+S to download
    window.addEventListener('keydown', (e)=>{ if((e.ctrlKey||e.metaKey) && e.key.toLowerCase()==='s'){ e.preventDefault(); downloadHTML(); } });

  </script>
</body>
</html>
