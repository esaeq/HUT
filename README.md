<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>霓虹突擊：無限競技場 (NEON STRIKE)</title>
<script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; user-select: none; -webkit-user-select: none; touch-action: none; }
  body { background: #070913; color: #e2e8f0; font-family: 'Segoe UI', system-ui, -apple-system, sans-serif; overflow: hidden; width: 100vw; height: 100vh; }
  canvas { display: block; width: 100%; height: 100%; }

  /* UI 層級 */
  .screen { position: fixed; inset: 0; display: flex; flex-direction: column; z-index: 10; pointer-events: none; }
  .interactive { pointer-events: auto; }
  .hidden { display: none !important; }

  /* 大廳樣式 */
  #lobby-screen { background: radial-gradient(circle at center, rgba(7,9,19,0.55) 0%, rgba(7,9,19,0.92) 100%); padding: 16px; justify-content: space-between; overflow-y: auto; }
  .top-bar { display: flex; justify-content: space-between; align-items: center; gap: 12px; flex-wrap: wrap; }
  .title-box h1 { font-size: clamp(1.5rem, 4vw, 2.5rem); font-weight: 900; letter-spacing: 2px; background: linear-gradient(90deg, #00f0ff, #ff007f); -webkit-background-clip: text; -webkit-text-fill-color: transparent; text-shadow: 0 0 20px rgba(0,240,255,0.3); }
  .title-box p { font-size: 0.8rem; color: #94a3b8; }
  .stats-pill { background: rgba(15, 23, 42, 0.85); border: 1px solid rgba(0, 240, 255, 0.3); border-radius: 999px; padding: 6px 14px; font-size: 0.85rem; display: flex; gap: 12px; align-items: center; }
  .stats-pill input { background: transparent; border: none; border-bottom: 1px solid #00f0ff; color: #fff; width: 90px; font-weight: bold; outline: none; text-align: center; touch-action: auto; user-select: auto; }

  .lobby-main { display: grid; grid-template-columns: 1.2fr 1fr; gap: 16px; max-width: 1100px; width: 100%; margin: auto; }
  @media (max-width: 768px) { .lobby-main { grid-template-columns: 1fr; } }

  .panel { background: rgba(15, 23, 42, 0.82); backdrop-filter: blur(10px); border: 1px solid rgba(148, 163, 184, 0.2); border-radius: 16px; padding: 16px; box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
  .panel h2 { font-size: 1rem; color: #00f0ff; margin-bottom: 12px; display: flex; align-items: center; gap: 6px; text-transform: uppercase; letter-spacing: 1px; }

  .tabs { display: flex; gap: 8px; margin-bottom: 14px; }
  .tab-btn { flex: 1; padding: 10px; border-radius: 10px; border: 1px solid rgba(255,255,255,0.15); background: rgba(30,41,59,0.6); color: #cbd5e1; font-weight: bold; cursor: pointer; transition: 0.2s; }
  .tab-btn.active { background: linear-gradient(135deg, #0284c7, #00f0ff); color: #050811; border-color: #00f0ff; box-shadow: 0 0 15px rgba(0,240,255,0.4); }

  .control-group { margin-bottom: 12px; }
  .control-group label { display: block; font-size: 0.8rem; color: #94a3b8; margin-bottom: 6px; }
  .btn-row { display: flex; gap: 6px; flex-wrap: wrap; }
  .opt-btn { padding: 6px 12px; border-radius: 8px; border: 1px solid rgba(255,255,255,0.15); background: rgba(30,41,59,0.8); color: #e2e8f0; font-size: 0.85rem; cursor: pointer; }
  .opt-btn.active { background: #ff007f; border-color: #ff007f; color: #fff; font-weight: bold; }

  .room-box { display: flex; gap: 8px; margin-top: 8px; flex-wrap: wrap; }
  .room-input { flex: 1; min-width: 110px; padding: 10px; border-radius: 10px; border: 1px solid #334155; background: #0f172a; color: #fff; font-size: 1rem; text-align: center; letter-spacing: 2px; font-weight: bold; touch-action: auto; user-select: auto; }
  .action-btn { padding: 10px 16px; border-radius: 10px; border: none; font-weight: bold; cursor: pointer; transition: 0.15s; font-size: 0.9rem; }
  .btn-cyan { background: #00f0ff; color: #050811; }
  .btn-pink { background: #ff007f; color: #fff; }
  .btn-amber { background: #f59e0b; color: #050811; }
  .btn-green { background: linear-gradient(135deg, #10b981, #059669); color: #fff; font-size: 1.1rem; width: 100%; padding: 14px; margin-top: 8px; box-shadow: 0 0 20px rgba(16,185,129,0.4); }
  .action-btn:active { transform: scale(0.96); }

  /* 聊天室 */
  .chat-container { display: flex; flex-direction: column; height: 220px; }
  .chat-messages { flex: 1; overflow-y: auto; background: rgba(7, 9, 19, 0.7); border-radius: 10px; padding: 8px; font-size: 0.85rem; display: flex; flex-direction: column; gap: 4px; margin-bottom: 8px; border: 1px solid rgba(255,255,255,0.08); }
  .chat-msg { line-height: 1.35; word-break: break-all; }
  .chat-msg.sys { color: #38bdf8; font-style: italic; }
  .chat-msg.cheat { color: #fbbf24; font-weight: bold; }
  .chat-input-row { display: flex; gap: 6px; }
  .chat-input-row input { flex: 1; padding: 8px 10px; border-radius: 8px; border: 1px solid #334155; background: #0f172a; color: #fff; font-size: 0.85rem; touch-action: auto; user-select: auto; }

  /* 規則卡 */
  .rules-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 8px; max-width: 1100px; width: 100%; margin: 0 auto; }
  @media (max-width: 640px) { .rules-grid { grid-template-columns: repeat(2, 1fr); } }
  .rule-card { background: rgba(15, 23, 42, 0.75); border: 1px solid rgba(255,255,255,0.1); border-radius: 10px; padding: 8px 10px; font-size: 0.75rem; }
  .rule-card strong { color: #00f0ff; display: block; font-size: 0.85rem; margin-bottom: 2px; }

  /* 遊戲內 HUD */
  #hud-screen { justify-content: space-between; padding: 12px; }
  .hud-top { display: flex; justify-content: space-between; align-items: flex-start; gap: 8px; }
  .leaderboard { background: rgba(15, 23, 42, 0.75); border: 1px solid rgba(255,255,255,0.15); border-radius: 10px; padding: 8px 12px; font-size: 0.8rem; min-width: 170px; }
  .lb-row { display: flex; justify-content: space-between; gap: 10px; padding: 2px 0; }
  .lb-row.me { color: #00f0ff; font-weight: bold; }

  .hud-center-info { background: rgba(15, 23, 42, 0.8); border: 1px solid rgba(0,240,255,0.3); border-radius: 999px; padding: 6px 16px; font-weight: bold; font-size: 0.9rem; display: flex; gap: 12px; align-items: center; }
  .hud-actions { display: flex; gap: 6px; flex-wrap: wrap; justify-content: flex-end; }
  .mini-btn { background: rgba(15, 23, 42, 0.85); border: 1px solid rgba(255,255,255,0.25); color: #fff; padding: 6px 10px; border-radius: 8px; font-size: 0.8rem; font-weight: bold; cursor: pointer; }
  .mini-btn.cmd-btn { border-color: #fbbf24; color: #fbbf24; box-shadow: 0 0 10px rgba(251,191,36,0.4); }

  /* 局內聊天懸浮區 */
  .ingame-chat { position: fixed; top: 65px; left: 12px; width: 240px; max-width: 60vw; z-index: 15; }
  .ingame-chat-log { max-height: 95px; overflow-y: auto; background: rgba(7, 9, 19, 0.55); border-radius: 8px; padding: 6px; font-size: 0.75rem; margin-bottom: 4px; pointer-events: auto; }
  .quick-phrases { display: flex; gap: 4px; flex-wrap: wrap; margin-bottom: 4px; }
  .qp-btn { background: rgba(30,41,59,0.8); border: 1px solid rgba(255,255,255,0.2); color: #e2e8f0; font-size: 0.7rem; padding: 3px 6px; border-radius: 6px; cursor: pointer; }

  /* 觀戰列 */
  .spec-bar { position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); background: rgba(15,23,42,0.9); border: 1px solid #f59e0b; border-radius: 999px; padding: 8px 16px; display: flex; gap: 10px; align-items: center; z-index: 20; }

  /* 管理員作弊面板 */
  #admin-panel { position: fixed; top: 55px; right: 12px; width: 270px; max-height: 80vh; overflow-y: auto; background: rgba(10, 15, 30, 0.94); border: 1px solid #fbbf24; border-radius: 14px; padding: 12px; z-index: 50; box-shadow: 0 0 25px rgba(251,191,36,0.3); }
  #admin-panel h3 { color: #fbbf24; font-size: 0.95rem; margin-bottom: 8px; display: flex; justify-content: space-between; align-items: center; }
  .cheat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 6px; }
  .cheat-toggle { padding: 7px 6px; border-radius: 8px; border: 1px solid #475569; background: #1e293b; color: #cbd5e1; font-size: 0.75rem; font-weight: bold; cursor: pointer; text-align: center; }
  .cheat-toggle.on { background: #fbbf24; color: #050811; border-color: #fef08a; box-shadow: 0 0 10px rgba(251,191,36,0.5); }

  /* 結算彈窗 */
  #gameover-modal { position: fixed; inset: 0; background: rgba(5, 8, 17, 0.85); display: flex; align-items: center; justify-content: center; z-index: 40; padding: 16px; }
  .modal-card { background: #0f172a; border: 2px solid #00f0ff; border-radius: 18px; padding: 24px; max-width: 400px; width: 100%; text-align: center; }
  .modal-card h2 { font-size: 1.8rem; color: #00f0ff; margin-bottom: 8px; }
  .modal-stats { background: rgba(30,41,59,0.6); border-radius: 12px; padding: 12px; margin: 14px 0; font-size: 0.9rem; display: flex; flex-direction: column; gap: 6px; }
</style>
</head>
<body>

<canvas id="gameCanvas"></canvas>

<!-- 大廳畫面 -->
<div id="lobby-screen" class="screen">
  <div class="top-bar interactive">
    <div class="title-box">
      <h1>NEON STRIKE // 無限競技場</h1>
      <p>三搖桿極速射擊 • 無限射程 • 命中回彈 +0.5</p>
    </div>
    <div class="stats-pill">
      <span>ID: <input type="text" id="player-name-input" value="特工_01" maxlength="10"></span>
      <span>勝場: <strong id="stat-wins" style="color:#10b981">0</strong></span>
      <span>擊殺: <strong id="stat-kills" style="color:#f59e0b">0</strong></span>
    </div>
  </div>

  <div class="lobby-main">
    <!-- 左側：對戰設定 -->
    <div class="panel interactive">
      <h2>⚔️ 競技場配置</h2>
      <div class="tabs">
        <button class="tab-btn active" id="tab-pve" onclick="switchModeTab('pve')">單人 PVE / 大亂鬥</button>
        <button class="tab-btn" id="tab-pvp" onclick="switchModeTab('pvp')">線上連線 PVP</button>
      </div>

      <!-- PVE 設定 -->
      <div id="pve-settings">
        <div class="control-group">
          <label>參戰人數（1v1v1... 多人混戰大亂鬥）</label>
          <div class="btn-row" id="bot-count-btns">
            <button class="opt-btn active" data-val="1" onclick="selectOpt('botCount', 1, this)">1v1 單挑 (2人)</button>
            <button class="opt-btn" data-val="3" onclick="selectOpt('botCount', 3, this)">4人混戰</button>
            <button class="opt-btn" data-val="7" onclick="selectOpt('botCount', 7, this)">8人大亂鬥</button>
            <button class="opt-btn" data-val="15" onclick="selectOpt('botCount', 15, this)">16人超大亂鬥</button>
          </div>
        </div>
        <div class="control-group">
          <label>AI 強度</label>
          <div class="btn-row" id="ai-diff-btns">
            <button class="opt-btn" data-val="easy" onclick="selectOpt('aiDiff', 'easy', this)">新兵</button>
            <button class="opt-btn active" data-val="normal" onclick="selectOpt('aiDiff', 'normal', this)">老手</button>
            <button class="opt-btn" data-val="ace" onclick="selectOpt('aiDiff', 'ace', this)">王牌</button>
          </div>
        </div>
      </div>

      <!-- PVP 連線設定 -->
      <div id="pvp-settings" class="hidden">
        <div class="control-group">
          <label>線上即時連線（支援 2 人至無上限多人同房大亂鬥）</label>
          <div class="room-box">
            <button class="action-btn btn-cyan" onclick="createOnlineRoom()">創建房間</button>
            <input type="text" id="join-room-code" class="room-input" placeholder="4位房號" maxlength="4" inputmode="numeric">
            <button class="action-btn btn-pink" onclick="joinOnlineRoom(false)">加入參戰</button>
            <button class="action-btn btn-amber" onclick="joinOnlineRoom(true)">進入觀戰席</button>
          </div>
          <p id="pvp-status-text" style="font-size:0.8rem; color:#38bdf8; margin-top:8px;">狀態：未連線（輸入4位數房號即可與好友連線或觀戰）</p>
        </div>
      </div>

      <!-- 通用：復活次數設定 -->
      <div class="control-group">
        <label>每人復活次數上限（0 = 一命淘汰制，∞ = 無限死鬥模式）</label>
        <div class="btn-row" id="respawn-btns">
          <button class="opt-btn" onclick="selectOpt('maxRespawns', 0, this)">0 次 (淘汰)</button>
          <button class="opt-btn" onclick="selectOpt('maxRespawns', 1, this)">1 次</button>
          <button class="opt-btn active" onclick="selectOpt('maxRespawns', 3, this)">3 次</button>
          <button class="opt-btn" onclick="selectOpt('maxRespawns', 5, this)">5 次</button>
          <button class="opt-btn" onclick="selectOpt('maxRespawns', 999, this)">∞ 無限死鬥</button>
        </div>
      </div>

      <button class="action-btn btn-green" id="start-game-btn" onclick="startLocalOrHostGame()">🚀 立即進入戰場</button>
    </div>

    <!-- 右側：大廳聊天室 -->
    <div class="panel interactive">
      <h2>💬 戰術頻道 & 聊天室</h2>
      <div class="chat-container">
        <div class="chat-messages" id="lobby-chat-log"></div>
        <div class="chat-input-row">
          <input type="text" id="lobby-chat-input" placeholder="輸入訊息或代碼..." onkeydown="if(event.key==='Enter') sendChat('lobby')">
          <button class="action-btn btn-cyan" onclick="sendChat('lobby')">發送</button>
        </div>
      </div>
    </div>
  </div>

  <!-- 底部規則速覽 -->
  <div class="rules-grid">
    <div class="rule-card"><strong>❤️ 3 滴血 / 1 彈 1 血</strong>拾取隨機綠色急救包回復 1 滴血</div>
    <div class="rule-card"><strong>∞ 無限射程子彈</strong>子彈飛行距離無限，撞牆或出界消失</div>
    <div class="rule-card"><strong>🎯 命中 +0.5 發彈藥</strong>彈匣上限 3 發，神準連射滾雪球</div>
    <div class="rule-card"><strong>⚡ 1秒 5倍速衝刺</strong>冷卻 10 秒，期間可穿透子彈閃避</div>
  </div>
</div>

<!-- 遊戲內 HUD -->
<div id="hud-screen" class="screen hidden">
  <div class="hud-top">
    <div class="leaderboard interactive" id="leaderboard-box"></div>
    <div class="hud-center-info">
      <span id="hud-mode-label">混戰模式</span>
      <span id="hud-alive-label" style="color:#10b981">存活: 2</span>
    </div>
    <div class="hud-actions interactive">
      <button class="mini-btn cmd-btn hidden" id="hud-cmd-btn" onclick="toggleAdminPanel()">[CMD] 指令</button>
      <button class="mini-btn" id="hud-toggle-spec-btn" onclick="toggleSpectateSelf()">切換觀戰</button>
      <button class="mini-btn" onclick="toggleInGameChat()">💬 聊天</button>
      <button class="mini-btn" style="border-color:#ef4444;color:#fca5a5" onclick="returnToLobby()">離開</button>
    </div>
  </div>

  <!-- 局內微縮聊天室 -->
  <div class="ingame-chat" id="ingame-chat-box">
    <div class="ingame-chat-log" id="ingame-chat-log"></div>
    <div class="quick-phrases interactive">
      <button class="qp-btn" onclick="sendQuickChat('好準！')">好準！</button>
      <button class="qp-btn" onclick="sendQuickChat('差一點！')">差一點！</button>
      <button class="qp-btn" onclick="sendQuickChat('666')">666</button>
      <button class="qp-btn" onclick="sendQuickChat('再來一局！')">再來一局！</button>
    </div>
    <div class="chat-input-row interactive hidden" id="ingame-chat-input-wrap">
      <input type="text" id="ingame-chat-input" placeholder="輸入聊天或代碼..." onkeydown="if(event.key==='Enter') sendChat('ingame')">
      <button class="mini-btn btn-cyan" style="color:#000" onclick="sendChat('ingame')">送出</button>
    </div>
  </div>

  <!-- 觀戰控制列 -->
  <div class="spec-bar interactive hidden" id="spectator-bar">
    <span style="color:#f59e0b;font-weight:bold;font-size:0.85rem;">👁️ 觀戰席</span>
    <button class="mini-btn" onclick="cycleSpectateTarget(-1)">◀ 上一位</button>
    <span id="spec-target-name" style="font-size:0.85rem;min-width:80px;text-align:center;">上帝全景</span>
    <button class="mini-btn" onclick="cycleSpectateTarget(1)">下一位 ▶</button>
    <button class="mini-btn btn-amber" onclick="setGlobalOverview()">全景視角</button>
  </div>
</div>

<!-- 管理員作弊面板 (/YEE123456 解鎖) -->
<div id="admin-panel" class="interactive hidden">
  <h3>
    <span>⚡ 管理員指令面板</span>
    <button class="mini-btn" onclick="toggleAdminPanel()">✕</button>
  </h3>
  <div class="cheat-grid">
    <button class="cheat-toggle" id="cht-god" onclick="toggleCheat('god')">🛡️ 無敵金身</button>
    <button class="cheat-toggle" id="cht-infAmmo" onclick="toggleCheat('infAmmo')">🔥 無限彈藥</button>
    <button class="cheat-toggle" id="cht-noCdDash" onclick="toggleCheat('noCdDash')">⚡ 零冷卻衝刺</button>
    <button class="cheat-toggle" id="cht-speed" onclick="toggleCheat('speed')">👟 2倍極速</button>
    <button class="cheat-toggle" id="cht-autoAim" onclick="toggleCheat('autoAim')">🎯 自動瞄準</button>
    <button class="cheat-toggle" id="cht-homing" onclick="toggleCheat('homing')">🌀 子彈轉彎追蹤</button>
    <button class="cheat-toggle" id="cht-wallBullet" onclick="toggleCheat('wallBullet')">🧱 子彈穿牆</button>
    <button class="cheat-toggle" id="cht-wallWalk" onclick="toggleCheat('wallWalk')">👻 人物穿牆</button>
    <button class="cheat-toggle" id="cht-esp" onclick="toggleCheat('esp')">👁️ 全圖透視</button>
    <button class="cheat-toggle" id="cht-giant" onclick="toggleSizeCheat()">🦖 體型切換</button>
    <button class="cheat-toggle" onclick="spawnCheatHeals()">💚 召喚3補包</button>
    <button class="cheat-toggle" onclick="clearAllBullets()">🧹 清空全場彈幕</button>
    <button class="cheat-toggle" style="grid-column: span 2;" onclick="addExtraRespawns()">➕ 增加 +3 次復活機會</button>
  </div>
</div>

<!-- 結算畫面 -->
<div id="gameover-modal" class="hidden">
  <div class="modal-card interactive">
    <h2 id="go-title">勝利！</h2>
    <p id="go-subtitle" style="color:#94a3b8;font-size:0.9rem;">競技場最後生還者</p>
    <div class="modal-stats" id="go-stats"></div>
    <div style="display:flex;gap:10px;justify-content:center;">
      <button class="action-btn btn-cyan" style="flex:1" onclick="startLocalOrHostGame()">再來一局</button>
      <button class="action-btn btn-pink" style="flex:1" onclick="returnToLobby()">返回大廳</button>
    </div>
  </div>
</div>

<script>
const canvas
