<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>BREACH:// — network intrusion console</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;700&family=JetBrains+Mono:wght@400;500;700&display=swap" rel="stylesheet">
<style>
  :root {
    --bg-0: #050709;
    --panel: rgba(255,255,255,0.035);
    --panel-border: rgba(255,255,255,0.09);
    --cyan: #2fd9ee;
    --violet: #a985fa;
    --text: #dbe4ec;
    --text-dim: #8b96a5;
    --green: #3ddc97;
    --amber: #f5b544;
    --red: #f2566b;
    --font-head: 'Space Grotesk', sans-serif;
    --font-mono: 'JetBrains Mono', monospace;
  }

  * { box-sizing: border-box; }

  html, body {
    margin: 0;
    padding: 0;
    background: var(--bg-0);
    color: var(--text);
    font-family: var(--font-mono);
    min-height: 100vh;
  }

  body {
    background-image:
      radial-gradient(circle at 15% 0%, rgba(47,217,238,0.10), transparent 45%),
      radial-gradient(circle at 85% 100%, rgba(169,133,250,0.10), transparent 45%),
      repeating-linear-gradient(0deg, rgba(255,255,255,0.025) 0px, rgba(255,255,255,0.025) 1px, transparent 1px, transparent 34px);
    padding: 28px 20px 60px;
  }

  .wrap { max-width: 1080px; margin: 0 auto; }

  .topbar {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    flex-wrap: wrap;
    gap: 14px;
    margin-bottom: 16px;
  }

  .brand-title {
    font-family: var(--font-head);
    font-weight: 700;
    font-size: 30px;
    letter-spacing: 0.5px;
    animation: bootFlicker 1.4s ease-out 1;
  }
  .brand-title .grad {
    background: linear-gradient(90deg, var(--cyan), var(--violet));
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
  }
  .brand-sub { font-size: 12.5px; color: var(--text-dim); margin-top: 2px; }

  .stats { display: flex; align-items: center; gap: 20px; }
  .stat { display: flex; flex-direction: column; align-items: flex-end; }
  .stat-label { font-size: 10.5px; color: var(--text-dim); }
  .stat-value { font-family: var(--font-head); font-size: 17px; font-weight: 700; color: var(--text); }

  .restart-btn {
    font-family: var(--font-mono);
    font-size: 11px;
    color: var(--text-dim);
    background: transparent;
    border: 1px solid var(--panel-border);
    border-radius: 7px;
    padding: 7px 12px;
    cursor: pointer;
    transition: all 0.2s ease;
  }
  .restart-btn:hover { color: var(--cyan); border-color: rgba(47,217,238,0.4); }

  .objective {
    background: var(--panel);
    border: 1px solid var(--panel-border);
    backdrop-filter: blur(18px);
    -webkit-backdrop-filter: blur(18px);
    border-radius: 10px;
    padding: 10px 16px;
    font-size: 13px;
    color: var(--cyan);
    margin-bottom: 18px;
  }

  .grid {
    display: grid;
    grid-template-columns: 0.85fr 1.25fr;
    gap: 18px;
    align-items: start;
  }
  @media (max-width: 760px) {
    .grid { grid-template-columns: 1fr; }
    .stats { gap: 16px; }
  }

  .panel {
    background: var(--panel);
    border: 1px solid var(--panel-border);
    backdrop-filter: blur(22px);
    -webkit-backdrop-filter: blur(22px);
    border-radius: 14px;
    padding: 16px;
    box-shadow: 0 20px 60px rgba(0,0,0,0.35);
  }
  .panel-title { font-size: 11.5px; color: var(--text-dim); margin-bottom: 10px; }

  #netSvg { width: 100%; height: auto; display: block; }

  .edge { stroke: rgba(255,255,255,0.08); stroke-width: 2; transition: stroke 0.4s ease; }
  .edge.active { stroke: url(#edgeGradient); }

  .node-hex {
    fill: rgba(255,255,255,0.02);
    stroke: rgba(255,255,255,0.16);
    stroke-width: 1.6;
    transition: all 0.4s ease;
  }
  .node.locked .node-hex { cursor: not-allowed; }
  .node.reachable .node-hex,
  .node.breached .node-hex { cursor: pointer; }
  .node.reachable .node-hex {
    stroke: var(--violet);
    fill: rgba(169,133,250,0.08);
    filter: drop-shadow(0 0 6px rgba(169,133,250,0.55));
    animation: pulseGlow 2.1s ease-in-out infinite;
  }
  .node.reachable:hover .node-hex { fill: rgba(169,133,250,0.18); }
  .node.breached .node-hex {
    stroke: var(--green);
    fill: rgba(61,220,151,0.14);
    filter: drop-shadow(0 0 7px rgba(61,220,151,0.55));
  }
  .node.breached:hover .node-hex { fill: rgba(61,220,151,0.24); }

  .node-label { font-family: var(--font-mono); font-size: 10.5px; fill: var(--text-dim); text-anchor: middle; pointer-events: none; }
  .node.breached .node-label { fill: var(--green); }
  .node.reachable .node-label { fill: var(--violet); }

  .connect-ring {
    fill: none;
    stroke: var(--cyan);
    stroke-width: 1.2;
    stroke-dasharray: 4 5;
    animation: spin 6s linear infinite;
    transform-origin: center;
    pointer-events: none;
  }

  .legend { display: flex; gap: 16px; margin-top: 8px; flex-wrap: wrap; }
  .legend-item { font-size: 10.5px; color: var(--text-dim); display: flex; align-items: center; gap: 6px; }
  .dot { width: 8px; height: 8px; border-radius: 2px; display: inline-block; }
  .dot.breached { background: var(--green); }
  .dot.reachable { background: var(--violet); }
  .dot.locked { background: rgba(255,255,255,0.2); }

  .terminal-output {
    height: 380px;
    overflow-y: auto;
    font-size: 12.8px;
    line-height: 1.65;
    padding-right: 6px;
  }
  .terminal-output::-webkit-scrollbar { width: 6px; }
  .terminal-output::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.15); border-radius: 4px; }

  .line { white-space: pre-wrap; word-break: break-word; }
  .line.head { color: var(--cyan); margin-top: 6px; }
  .line.ok { color: var(--green); }
  .line.warn { color: var(--amber); }
  .line.err { color: var(--red); }
  .line.dim { color: var(--text-dim); }
  .line.boot { color: var(--text-dim); }

  .crack-block { margin: 8px 0 10px; }
  .crack-hash { color: var(--violet); letter-spacing: 1.5px; font-size: 12.5px; }
  .crack-bar { height: 6px; background: rgba(255,255,255,0.08); border-radius: 4px; margin: 6px 0 4px; overflow: hidden; }
  .crack-fill { height: 100%; width: 0%; background: linear-gradient(90deg, var(--cyan), var(--violet)); transition: width 0.08s linear; }
  .crack-pct { font-size: 11px; color: var(--text-dim); }

  @keyframes pulseGlow {
    0%, 100% { filter: drop-shadow(0 0 4px rgba(169,133,250,0.4)); }
    50% { filter: drop-shadow(0 0 12px rgba(169,133,250,0.75)); }
  }
  @keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
  @keyframes bootFlicker {
    0% { opacity: 0.2; } 8% { opacity: 1; } 11% { opacity: 0.3; } 18% { opacity: 1; } 100% { opacity: 1; }
  }
</style>
</head>
<body>
  <div class="wrap">
    <header class="topbar">
      <div>
        <div class="brand-title">BREACH<span class="grad">://</span></div>
        <div class="brand-sub">network intrusion console</div>
      </div>
      <div class="stats">
        <div class="stat">
          <span class="stat-label">credits</span>
          <span class="stat-value" id="credits">0</span>
        </div>
        <div class="stat">
          <span class="stat-label">rank</span>
          <span class="stat-value" id="rank">SCRIPT KID</span>
        </div>
        <button class="restart-btn" id="restartBtn">restart</button>
      </div>
    </header>

    <div class="objective" id="objective">establishing connection...</div>

    <main class="grid">
      <section class="panel">
        <div class="panel-title">network map — click a node to hack it</div>
        <svg id="netSvg" viewBox="0 0 480 400" preserveAspectRatio="xMidYMid meet">
          <defs>
            <linearGradient id="edgeGradient" x1="0%" y1="0%" x2="100%" y2="0%">
              <stop offset="0%" stop-color="#2fd9ee"/>
              <stop offset="100%" stop-color="#a985fa"/>
            </linearGradient>
          </defs>
          <g id="graphLayer"></g>
        </svg>
        <div class="legend">
          <span class="legend-item"><i class="dot breached"></i>breached</span>
          <span class="legend-item"><i class="dot reachable"></i>click to hack</span>
          <span class="legend-item"><i class="dot locked"></i>locked</span>
        </div>
      </section>

      <section class="panel">
        <div class="panel-title">terminal output</div>
        <div class="terminal-output" id="terminal">
          <div class="line boot">Initializing local proxy routing...</div>
          <div class="line head">> READY FOR INTRUSION SELECTION</div>
        </div>
      </section>
    </main>
  </div>

  <script>
    // Paste or write your interactive JavaScript logic right here!
  </script>
</body>
</html>
