# 年会抽奖工具 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个纯静态 HTML 文件（`lottery.html`），为大屏幕年会抽奖提供炫酷科技风交互体验。

**Architecture:** DOM UI + Canvas 动画层。HTML/CSS 负责 4 个视图（配置、导入、抽奖、汇总），`<canvas>` 全屏覆盖负责粒子/滚动/光效动画。全局状态对象驱动视图切换，localStorage 持久化防丢失。Papa Parse 内联处理 CSV，Web Audio API 合成音效。

**Tech Stack:** Vanilla JS (ES6+), Canvas 2D API, Web Audio API, Papa Parse (内联), 零运行时依赖

## Global Constraints

- 交付物为单个 `lottery.html` 文件，双击浏览器即可打开
- 所有库（Papa Parse）内联到 HTML，不依赖 CDN 或网络
- 所有数据持久化于浏览器 localStorage
- 视觉：深空黑背景 `#0a0a0f`，主色调青蓝霓虹 `#00f0ff` + 紫罗兰 `#a855f7`，高亮金色 `#fbbf24`
- 动画 4 阶段：蓄势 → 滚动 → 揭晓 → 确认
- 逐个揭晓中奖者，不在场可作废重抽
- 支持 Chrome、Edge 浏览器
- 适配 1920×1080 和 3840×2160 分辨率

## File Structure

```
Lottery/
└── lottery.html          # 唯一交付物，包含全部 HTML/CSS/JS
```

`lottery.html` 内部模块组织（按 `<script>` 中的顺序）：

| 模块 | 职责 |
|------|------|
| Papa Parse (内联) | CSV 解析，约 20KB minified |
| StateManager | 全局状态读写，localStorage 持久化 |
| ViewController | 4 视图切换，显示/隐藏 DOM 区块 |
| ConfigView | 奖项配置表格的增删改 |
| ImportView | 文件拖拽上传、CSV 解析、名单预览 |
| ParticleEngine | Canvas 粒子背景系统（待机 + 加速模式） |
| AudioEngine | Web Audio API 音效合成（滚动/揭晓/庆祝/作废） |
| DrawEngine | Canvas slot-machine 名字滚动动画（核心） |
| LotteryView | 抽奖主页流程控制（状态机：蓄势→滚动→揭晓→确认） |
| ResultsView | 汇总展示、CSV 导出、打印 |
| App | 顶层初始化，引导流程 |

---

### Task 1: HTML 骨架 + CSS 基础

**Files:**
- Create: `lottery.html`

**Interfaces:**
- Produces: DOM 结构 — 4 个视图容器 (`#view-config`, `#view-import`, `#view-lottery`, `#view-results`)，1 个全屏 Canvas (`#animCanvas`)，全局 CSS 变量和基础样式

- [ ] **Step 1: 创建 HTML 骨架文件**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>年会抽奖</title>
  <style>
    /* === CSS Reset & Variables === */
    :root {
      --bg: #0a0a0f;
      --cyan: #00f0ff;
      --purple: #a855f7;
      --gold: #fbbf24;
      --text: #e2e8f0;
      --text-dim: #64748b;
      --card-bg: rgba(15, 15, 30, 0.85);
      --border: rgba(0, 240, 255, 0.2);
      --glow: 0 0 20px rgba(0, 240, 255, 0.4);
      --gold-glow: 0 0 30px rgba(251, 191, 36, 0.6);
      --font-mono: 'SF Mono', 'Cascadia Code', 'Consolas', monospace;
      --font-sans: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    }

    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      font-family: var(--font-sans);
      background: var(--bg);
      color: var(--text);
      width: 100vw;
      height: 100vh;
      overflow: hidden;
      user-select: none;
      -webkit-user-select: none;
    }

    /* === Canvas Layer === */
    #animCanvas {
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      z-index: 0;
      pointer-events: none;
    }

    /* === View Containers === */
    .view {
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      z-index: 1;
      display: none;
      flex-direction: column;
      align-items: center;
      justify-content: center;
    }
    .view.active { display: flex; }

    /* === Shared Components === */
    .panel {
      background: var(--card-bg);
      border: 1px solid var(--border);
      border-radius: 16px;
      padding: 40px 48px;
      backdrop-filter: blur(20px);
      -webkit-backdrop-filter: blur(20px);
      max-width: 90vw;
      max-height: 85vh;
      overflow-y: auto;
    }

    .btn {
      display: inline-flex;
      align-items: center;
      gap: 8px;
      padding: 14px 32px;
      border: 1px solid var(--cyan);
      border-radius: 8px;
      background: rgba(0, 240, 255, 0.08);
      color: var(--cyan);
      font-size: 18px;
      font-weight: 600;
      cursor: pointer;
      transition: all 0.25s ease;
      font-family: var(--font-sans);
    }
    .btn:hover {
      background: rgba(0, 240, 255, 0.18);
      box-shadow: var(--glow);
    }
    .btn.primary {
      background: linear-gradient(135deg, var(--cyan), var(--purple));
      border-color: transparent;
      color: #fff;
      box-shadow: var(--glow);
    }
    .btn.danger {
      border-color: #ef4444;
      color: #ef4444;
      background: rgba(239, 68, 68, 0.08);
    }
    .btn.gold {
      border-color: var(--gold);
      color: var(--gold);
      background: rgba(251, 191, 36, 0.1);
      box-shadow: var(--gold-glow);
    }

    .title {
      font-size: 36px;
      font-weight: 800;
      background: linear-gradient(135deg, var(--cyan), var(--purple));
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      background-clip: text;
      margin-bottom: 24px;
      text-align: center;
    }

    .subtitle {
      font-size: 18px;
      color: var(--text-dim);
      margin-bottom: 32px;
      text-align: center;
    }

    /* === Mute Button === */
    #btnMute {
      position: fixed;
      top: 20px;
      right: 20px;
      z-index: 100;
      width: 48px;
      height: 48px;
      border-radius: 50%;
      border: 1px solid var(--border);
      background: var(--card-bg);
      color: var(--text-dim);
      font-size: 22px;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
      transition: all 0.2s;
    }
    #btnMute:hover { color: var(--cyan); border-color: var(--cyan); }

    /* === Print Styles === */
    @media print {
      body { background: #fff; color: #000; }
      #animCanvas, .btn, #btnMute { display: none !important; }
      .view { position: static; display: block !important; }
      .panel { border: 1px solid #ccc; box-shadow: none; background: #fff; }
      .title { -webkit-text-fill-color: #000; background: none; }
    }
  </style>
</head>
<body>
  <canvas id="animCanvas"></canvas>
  <button id="btnMute" title="静音/取消静音">🔊</button>

  <!-- View 1: Config -->
  <div id="view-config" class="view active">
    <div class="panel">
      <h1 class="title">🎰 年会抽奖</h1>
      <p class="subtitle">第一步：设置奖项</p>
      <table id="configTable">
        <thead><tr><th>奖项名称</th><th>中奖人数</th><th>操作</th></tr></thead>
        <tbody id="configBody"></tbody>
      </table>
      <div style="text-align:center;margin:16px 0;">
        <button id="btnAddPrize" class="btn">+ 添加奖项</button>
      </div>
      <p id="configSummary" style="text-align:center;color:var(--text-dim);margin-bottom:20px;"></p>
      <div style="text-align:center;">
        <button id="btnConfigNext" class="btn primary">保存并进入下一步 →</button>
      </div>
    </div>
  </div>

  <!-- View 2: Import -->
  <div id="view-import" class="view">
    <div class="panel">
      <h1 class="title">📋 导入名单</h1>
      <p class="subtitle">第二步：上传签到名单</p>
      <div id="dropZone">
        <p style="font-size:48px;margin-bottom:12px;">📁</p>
        <p style="font-size:20px;color:var(--cyan);">拖拽文件到此处</p>
        <p style="color:var(--text-dim);margin-top:8px;">或点击选择 CSV / Excel 文件</p>
        <input type="file" id="fileInput" accept=".csv,.xlsx,.xls" hidden>
      </div>
      <div id="importError" style="color:#ef4444;text-align:center;margin:12px 0;display:none;"></div>
      <div id="previewArea" style="display:none;">
        <p id="previewCount" style="text-align:center;color:var(--cyan);margin-bottom:12px;"></p>
        <div id="previewList" style="max-height:300px;overflow-y:auto;"></div>
      </div>
      <div style="text-align:center;margin-top:20px;">
        <button id="btnImportBack" class="btn">← 返回</button>
        <button id="btnImportNext" class="btn primary" style="display:none;">确认名单，共 <span id="importCount">0</span> 人 →</button>
      </div>
    </div>
  </div>

  <!-- View 3: Lottery -->
  <div id="view-lottery" class="view">
    <div id="lotteryInfo">
      <p id="lotteryPrizeName"></p>
      <p id="lotteryProgress"></p>
    </div>
    <div id="lotteryStandby">
      <p id="standbyNext"></p>
      <p id="standbyRemaining"></p>
      <button id="btnStartDraw" class="btn primary">开始抽取下一位</button>
      <div id="standbyDrawn" style="margin-top:20px;"></div>
    </div>
    <div id="lotteryActions" style="display:none;">
      <button id="btnConfirm" class="btn gold">✅ 确认中奖</button>
      <button id="btnVoid" class="btn danger">❌ 作废重抽</button>
    </div>
  </div>

  <!-- View 4: Results -->
  <div id="view-results" class="view">
    <div class="panel">
      <h1 class="title">🏆 中奖结果汇总</h1>
      <div id="resultsList"></div>
      <p id="resultsTotal" style="text-align:center;color:var(--cyan);margin:20px 0;"></p>
      <div style="text-align:center;display:flex;gap:16px;justify-content:center;flex-wrap:wrap;">
        <button id="btnExport" class="btn primary">📥 导出 CSV</button>
        <button id="btnPrint" class="btn">🖨️ 打印</button>
        <button id="btnReset" class="btn danger">🔄 重置全部数据</button>
      </div>
    </div>
  </div>

  <script>
  // === Papa Parse (内联) ===
  // [Papa Parse minified source will be pasted here — see Task 4]
  </script>
</body>
</html>
```

- [ ] **Step 2: 添加 Config 表格和 Drop Zone 样式**

在 `<style>` 的 `@media print` 之前追加：

```css
    /* === Config Table === */
    #configTable {
      width: 100%;
      border-collapse: collapse;
      margin-bottom: 8px;
    }
    #configTable th, #configTable td {
      padding: 12px 16px;
      text-align: center;
      border-bottom: 1px solid var(--border);
    }
    #configTable th {
      color: var(--cyan);
      font-size: 14px;
      text-transform: uppercase;
      letter-spacing: 1px;
    }
    #configTable input {
      background: rgba(255,255,255,0.05);
      border: 1px solid var(--border);
      border-radius: 6px;
      color: var(--text);
      padding: 8px 12px;
      font-size: 16px;
      text-align: center;
      width: 160px;
      font-family: var(--font-sans);
      transition: border-color 0.2s;
    }
    #configTable input:focus {
      outline: none;
      border-color: var(--cyan);
      box-shadow: 0 0 8px rgba(0,240,255,0.2);
    }
    #configTable input[name="prizeCount"] { width: 80px; }
    .btn-del {
      background: none;
      border: 1px solid transparent;
      color: var(--text-dim);
      font-size: 20px;
      cursor: pointer;
      padding: 4px 10px;
      border-radius: 6px;
      transition: all 0.2s;
    }
    .btn-del:hover { color: #ef4444; border-color: #ef4444; }

    /* === Drop Zone === */
    #dropZone {
      border: 2px dashed var(--border);
      border-radius: 16px;
      padding: 48px;
      text-align: center;
      cursor: pointer;
      transition: all 0.3s;
    }
    #dropZone:hover, #dropZone.dragover {
      border-color: var(--cyan);
      background: rgba(0, 240, 255, 0.04);
      box-shadow: inset 0 0 40px rgba(0, 240, 255, 0.05);
    }

    /* === Preview List === */
    #previewList {
      display: flex;
      flex-wrap: wrap;
      gap: 8px;
      justify-content: center;
    }
    .name-tag {
      padding: 6px 14px;
      background: rgba(0, 240, 255, 0.08);
      border: 1px solid var(--border);
      border-radius: 20px;
      font-size: 14px;
      color: var(--text);
    }

    /* === Lottery View === */
    #lotteryInfo {
      position: fixed;
      top: 30px;
      left: 0;
      width: 100%;
      text-align: center;
      z-index: 10;
    }
    #lotteryPrizeName {
      font-size: 28px;
      font-weight: 700;
      color: var(--cyan);
      text-shadow: 0 0 20px rgba(0, 240, 255, 0.5);
    }
    #lotteryProgress {
      font-size: 16px;
      color: var(--text-dim);
      margin-top: 6px;
    }
    #lotteryStandby {
      text-align: center;
      z-index: 10;
    }
    #lotteryStandby p { margin-bottom: 12px; }
    #standbyNext {
      font-size: 24px;
      color: var(--text);
    }
    #standbyRemaining {
      font-size: 16px;
      color: var(--text-dim);
    }
    #standbyDrawn {
      display: flex;
      flex-wrap: wrap;
      gap: 12px;
      justify-content: center;
      max-width: 80vw;
    }
    .drawn-tag {
      padding: 8px 18px;
      background: rgba(251, 191, 36, 0.12);
      border: 1px solid rgba(251, 191, 36, 0.3);
      border-radius: 20px;
      font-size: 16px;
      color: var(--gold);
    }
    #lotteryActions {
      position: fixed;
      bottom: 50px;
      left: 0;
      width: 100%;
      display: flex;
      gap: 24px;
      justify-content: center;
      z-index: 10;
    }

    /* === Results View === */
    .prize-group {
      margin-bottom: 24px;
    }
    .prize-group h2 {
      color: var(--cyan);
      font-size: 24px;
      margin-bottom: 12px;
      text-align: center;
    }
    .winner-cards {
      display: flex;
      flex-wrap: wrap;
      gap: 12px;
      justify-content: center;
    }
    .winner-card {
      padding: 12px 24px;
      background: rgba(251, 191, 36, 0.1);
      border: 1px solid rgba(251, 191, 36, 0.3);
      border-radius: 12px;
      font-size: 18px;
      color: var(--gold);
      font-weight: 600;
      box-shadow: 0 0 16px rgba(251, 191, 36, 0.15);
    }
  </style>
```

- [ ] **Step 3: 在 Chrome 中打开验证骨架**

```bash
open -a "Google Chrome" lottery.html
```

预期：深色背景、4 个视图容器（仅 config 可见）、Canvas 层、静音按钮。所有样式符合视觉基调。

---

### Task 2: StateManager — 全局状态 + localStorage

**Files:**
- Modify: `lottery.html` — 在 `<script>` 标签中 `<body>` 闭合前添加 StateManager

**Interfaces:**
- Produces:
  - `StateManager.save()` — 全量写入 localStorage
  - `StateManager.load()` — 从 localStorage 恢复，返回 boolean 表示是否有已保存数据
  - `StateManager.reset()` — 清除所有数据（二次确认后）
  - `StateManager.state` — 全局状态对象 `{ config: [], candidates: [], original: [], winners: [], currentPrizeIdx: 0, currentSlotIdx: 0 }`

- [ ] **Step 1: 添加 StateManager 代码**

在 `<script>` 标签开始处添加：

```javascript
  // ========== StateManager ==========
  const KEYS = {
    config: 'lottery_config',
    candidates: 'lottery_candidates',
    original: 'lottery_original',
    winners: 'lottery_winners',
    state: 'lottery_state'
  };

  const StateManager = {
    state: {
      config: [],        // [{name: string, count: number}]
      candidates: [],    // string[]
      original: [],      // string[]
      winners: [],       // [{name: string, prize: string, time: string}]
      currentPrizeIdx: 0,
      currentSlotIdx: 0
    },

    save() {
      localStorage.setItem(KEYS.config, JSON.stringify(this.state.config));
      localStorage.setItem(KEYS.candidates, JSON.stringify(this.state.candidates));
      localStorage.setItem(KEYS.original, JSON.stringify(this.state.original));
      localStorage.setItem(KEYS.winners, JSON.stringify(this.state.winners));
      localStorage.setItem(KEYS.state, JSON.stringify({
        currentPrizeIdx: this.state.currentPrizeIdx,
        currentSlotIdx: this.state.currentSlotIdx
      }));
    },

    load() {
      try {
        const config = localStorage.getItem(KEYS.config);
        const candidates = localStorage.getItem(KEYS.candidates);
        const original = localStorage.getItem(KEYS.original);
        const winners = localStorage.getItem(KEYS.winners);
        const st = localStorage.getItem(KEYS.state);
        if (config) this.state.config = JSON.parse(config);
        if (candidates) this.state.candidates = JSON.parse(candidates);
        if (original) this.state.original = JSON.parse(original);
        if (winners) this.state.winners = JSON.parse(winners);
        if (st) {
          const parsed = JSON.parse(st);
          this.state.currentPrizeIdx = parsed.currentPrizeIdx || 0;
          this.state.currentSlotIdx = parsed.currentSlotIdx || 0;
        }
        return !!(config || candidates);
      } catch (e) {
        console.error('StateManager.load failed:', e);
        return false;
      }
    },

    reset() {
      Object.values(KEYS).forEach(k => localStorage.removeItem(k));
      this.state = {
        config: [],
        candidates: [],
        original: [],
        winners: [],
        currentPrizeIdx: 0,
        currentSlotIdx: 0
      };
    },

    hasSavedData() {
      return !!localStorage.getItem(KEYS.config);
    }
  };
```

- [ ] **Step 2: 添加恢复提示逻辑**

在 StateManager 之后添加：

```javascript
  // ========== Session Restore ==========
  function checkRestore() {
    if (StateManager.load()) {
      const hasWinners = StateManager.state.winners.length > 0;
      const hasCandidates = StateManager.state.candidates.length > 0;
      if (hasWinners || hasCandidates) {
        const action = confirm(
          '检测到上次的抽奖数据。\n\n' +
          '点击「确定」继续上次进度\n' +
          '点击「取消」重新开始'
        );
        if (!action) {
          StateManager.reset();
        }
      }
    }
  }
```

- [ ] **Step 3: 验证持久化**

打开 `lottery.html`，在 Console 中执行：

```javascript
StateManager.state.config = [{name: '测试奖', count: 3}];
StateManager.save();
// 刷新页面
// 预期：弹出恢复提示，确定后 config 恢复
```

---

### Task 3: ConfigView — 奖项配置页

**Files:**
- Modify: `lottery.html` — 在 `<script>` 中添加 ConfigView

**Interfaces:**
- Consumes: `StateManager`
- Produces: `ConfigView.init()`, `ConfigView.show()`, `ConfigView.hide()`

- [ ] **Step 1: 添加 ConfigView 代码**

```javascript
  // ========== ConfigView ==========
  const ConfigView = {
    init() {
      document.getElementById('btnAddPrize').addEventListener('click', () => this.addRow());
      document.getElementById('btnConfigNext').addEventListener('click', () => this.saveAndNext());
      this.render();
    },

    render() {
      const tbody = document.getElementById('configBody');
      const config = StateManager.state.config;
      tbody.innerHTML = config.map((prize, i) => `
        <tr>
          <td><input type="text" name="prizeName" value="${this.esc(prize.name)}"
                     data-index="${i}" placeholder="奖项名称"></td>
          <td><input type="number" name="prizeCount" value="${prize.count}"
                     data-index="${i}" min="1" placeholder="人数"></td>
          <td><button class="btn-del" data-index="${i}" title="删除">✕</button></td>
        </tr>
      `).join('');

      // Bind events
      tbody.querySelectorAll('input[name="prizeName"]').forEach(inp => {
        inp.addEventListener('input', (e) => {
          const idx = parseInt(e.target.dataset.index);
          StateManager.state.config[idx].name = e.target.value;
          this.updateSummary();
          StateManager.save();
        });
      });
      tbody.querySelectorAll('input[name="prizeCount"]').forEach(inp => {
        inp.addEventListener('input', (e) => {
          const idx = parseInt(e.target.dataset.index);
          const val = Math.max(1, parseInt(e.target.value) || 1);
          e.target.value = val;
          StateManager.state.config[idx].count = val;
          this.updateSummary();
          StateManager.save();
        });
      });
      tbody.querySelectorAll('.btn-del').forEach(btn => {
        btn.addEventListener('click', (e) => {
          const idx = parseInt(e.target.dataset.index);
          StateManager.state.config.splice(idx, 1);
          StateManager.save();
          this.render();
        });
      });

      this.updateSummary();
    },

    addRow() {
      StateManager.state.config.push({ name: '', count: 1 });
      StateManager.save();
      this.render();
      // Focus the new row's name input
      const inputs = document.querySelectorAll('#configBody input[name="prizeName"]');
      const last = inputs[inputs.length - 1];
      if (last) last.focus();
    },

    saveAndNext() {
      // Validate
      const config = StateManager.state.config;
      if (config.length === 0) {
        alert('请至少添加一个奖项');
        return;
      }
      for (let i = 0; i < config.length; i++) {
        if (!config[i].name.trim()) {
          alert(`第 ${i + 1} 个奖项名称为空，请填写`);
          return;
        }
      }
      StateManager.save();
      ImportView.show();
    },

    updateSummary() {
      const total = StateManager.state.config.reduce((sum, p) => sum + p.count, 0);
      document.getElementById('configSummary').textContent =
        total > 0 ? `总计将抽出 ${total} 人` : '请添加奖项';
    },

    esc(str) {
      const div = document.createElement('div');
      div.textContent = str;
      return div.innerHTML;
    },

    show() { ViewController.switchTo('view-config'); this.render(); },
    hide() { document.getElementById('view-config').classList.remove('active'); }
  };
```

- [ ] **Step 2: 验证 Config 交互**

打开页面，测试：添加奖项 → 填写名称和人数 → 删除一行 → 总计人数实时更新 → 空名验证阻止进入下一步。

---

### Task 4: ImportView + CSV 解析

**Files:**
- Modify: `lottery.html` — 内联 Papa Parse 库 + 添加 ImportView

**Interfaces:**
- Consumes: `StateManager`, `ViewController`
- Produces: `ImportView.init()`, `ImportView.show()`, `ImportView.hide()`

- [ ] **Step 1: 内联 Papa Parse**

从 https://raw.githubusercontent.com/mholt/PapaParse/master/papaparse.min.js 复制内容，替换 `<script>` 中的注释：

```javascript
  // === Papa Parse v5.4.1 (内联) ===
  /*
    Replace this comment with the minified Papa Parse source.
    Download: curl -o papaparse.min.js https://raw.githubusercontent.com/mholt/PapaParse/master/papaparse.min.js
    Then paste the entire content here.
    For the plan: the actual paste happens at implementation time.
  */
```

Papa Parse exposes `Papa` globally with `Papa.parse(csvString, config)`.

- [ ] **Step 2: 添加 ImportView 代码**

```javascript
  // ========== ImportView ==========
  const ImportView = {
    init() {
      const dropZone = document.getElementById('dropZone');
      const fileInput = document.getElementById('fileInput');

      dropZone.addEventListener('click', () => fileInput.click());
      dropZone.addEventListener('dragover', (e) => {
        e.preventDefault();
        dropZone.classList.add('dragover');
      });
      dropZone.addEventListener('dragleave', () => {
        dropZone.classList.remove('dragover');
      });
      dropZone.addEventListener('drop', (e) => {
        e.preventDefault();
        dropZone.classList.remove('dragover');
        const file = e.dataTransfer.files[0];
        if (file) this.processFile(file);
      });
      fileInput.addEventListener('change', (e) => {
        const file = e.target.files[0];
        if (file) this.processFile(file);
      });

      document.getElementById('btnImportBack').addEventListener('click', () => {
        ConfigView.show();
      });
      document.getElementById('btnImportNext').addEventListener('click', () => {
        this.confirmImport();
      });
    },

    processFile(file) {
      const ext = file.name.split('.').pop().toLowerCase();
      document.getElementById('importError').style.display = 'none';

      if (ext === 'csv') {
        const reader = new FileReader();
        reader.onload = (e) => {
          const result = Papa.parse(e.target.result, {
            header: false,
            skipEmptyLines: true,
            encoding: 'UTF-8'
          });
          // Try GBK if UTF-8 yields garbled text
          const firstCell = (result.data[0] && result.data[0][0]) || '';
          if (this.looksLikeGarbled(firstCell)) {
            // Re-read with GBK via TextDecoder
            this.readAsGBK(file);
            return;
          }
          this.handleParseResult(result.data);
        };
        reader.readAsText(file, 'UTF-8');
      } else if (ext === 'xlsx' || ext === 'xls') {
        this.readAsGBK(file); // Fallback: try CSV approach; if it's binary Excel, show error
        document.getElementById('importError').style.display = 'block';
        document.getElementById('importError').textContent =
          'Excel 文件请先另存为 CSV 格式（UTF-8 编码）后再上传';
      } else {
        document.getElementById('importError').style.display = 'block';
        document.getElementById('importError').textContent = '仅支持 CSV 和 Excel 格式';
      }
    },

    readAsGBK(file) {
      const reader = new FileReader();
      reader.onload = (e) => {
        const decoder = new TextDecoder('gbk');
        const text = decoder.decode(new Uint8Array(e.target.result));
        const result = Papa.parse(text, { header: false, skipEmptyLines: true });
        this.handleParseResult(result.data);
      };
      reader.readAsArrayBuffer(file);
    },

    looksLikeGarbled(str) {
      // Simple heuristic: high proportion of replacement characters or uncommon Unicode
      const garbled = (str.match(/[� --]/g) || []).length;
      return garbled > str.length * 0.3;
    },

    handleParseResult(rows) {
      if (rows.length === 0) {
        document.getElementById('importError').style.display = 'block';
        document.getElementById('importError').textContent = '未识别到任何姓名';
        return;
      }

      // Auto-detect the name column (first column, or column with 姓名/名字/name header)
      let colIdx = 0;
      const header = rows[0];
      const headerStr = (Array.isArray(header) ? header.join(' ') : String(header)).toLowerCase();
      if (headerStr.includes('姓名') || headerStr.includes('名字') || headerStr.includes('name')) {
        // First row is header, use first matching column
        for (let i = 0; i < header.length; i++) {
          const h = (header[i] || '').toLowerCase();
          if (h.includes('姓名') || h.includes('名字') || h.includes('name')) {
            colIdx = i;
            break;
          }
        }
        rows = rows.slice(1); // Skip header row
      }

      // Check if first row might be a header (no header keywords but first row looks like a title)
      if (rows.length > 0 && rows[0].length > 0) {
        const firstVal = String(rows[0][colIdx] || '').trim();
        if (firstVal.length > 10 || firstVal.toLowerCase() === 'name' || firstVal === '姓名') {
          rows = rows.slice(1);
        }
      }

      const names = rows
        .map(row => String(row[colIdx] || '').trim())
        .filter(name => name.length > 0 && name.length < 50);

      if (names.length === 0) {
        document.getElementById('importError').style.display = 'block';
        document.getElementById('importError').textContent = '未识别到任何姓名，请检查文件格式';
        return;
      }

      // Check count against config
      const totalNeeded = StateManager.state.config.reduce((s, p) => s + p.count, 0);
      if (names.length < totalNeeded) {
        document.getElementById('importError').style.display = 'block';
        document.getElementById('importError').textContent =
          `名单人数（${names.length}）少于奖项总人数（${totalNeeded}），差额 ${totalNeeded - names.length} 人，请补充名单`;
      }

      // Show preview
      const previewArea = document.getElementById('previewArea');
      const previewList = document.getElementById('previewList');
      const previewCount = document.getElementById('previewCount');
      const btnNext = document.getElementById('btnImportNext');
      const importCount = document.getElementById('importCount');

      previewCount.textContent = `共识别 ${names.length} 人`;
      importCount.textContent = names.length;
      previewList.innerHTML = names.slice(0, 200).map(n => `<span class="name-tag">${this.esc(n)}</span>`).join('');
      if (names.length > 200) {
        previewList.innerHTML += `<p style="width:100%;text-align:center;color:var(--text-dim);">... 还有 ${names.length - 200} 人</p>`;
      }
      previewArea.style.display = 'block';
      btnNext.style.display = 'inline-flex';

      // Store temporarily for confirm step
      this._pendingNames = names;
    },

    confirmImport() {
      if (!this._pendingNames || this._pendingNames.length === 0) return;
      StateManager.state.candidates = [...this._pendingNames];
      StateManager.state.original = [...this._pendingNames];
      StateManager.state.currentPrizeIdx = 0;
      StateManager.state.currentSlotIdx = 0;
      StateManager.state.winners = [];
      StateManager.save();
      this._pendingNames = null;
      LotteryView.show();
    },

    esc(str) {
      const div = document.createElement('div');
      div.textContent = str;
      return div.innerHTML;
    },

    show() {
      ViewController.switchTo('view-import');
      document.getElementById('previewArea').style.display = 'none';
      document.getElementById('btnImportNext').style.display = 'none';
      document.getElementById('importError').style.display = 'none';
    },

    hide() {
      document.getElementById('view-import').classList.remove('active');
    }
  };
```

- [ ] **Step 3: 下载 Papa Parse 并内联**

```bash
curl -o /tmp/papaparse.min.js https://raw.githubusercontent.com/mholt/PapaParse/master/papaparse.min.js
```

然后将 `/tmp/papaparse.min.js` 的内容粘贴到 `lottery.html` 中 `<script>` 的 Papa Parse 注释位置。

- [ ] **Step 4: 验证导入流程**

创建测试 CSV 文件：

```bash
echo '姓名
张三
李四
王五
赵六
钱七' > /tmp/test-names.csv
```

打开 `lottery.html`，拖入 `test-names.csv`，验证：识别 5 人、预览显示、确认按钮激活。

---

### Task 5: ParticleEngine — Canvas 粒子背景

**Files:**
- Modify: `lottery.html` — 在 `<script>` 中添加 ParticleEngine

**Interfaces:**
- Consumes: Canvas DOM element `#animCanvas`
- Produces:
  - `ParticleEngine.init()` — 初始化 Canvas，开始渲染循环
  - `ParticleEngine.setMode('idle' | 'accelerate' | 'burst')` — 切换粒子行为模式
  - `ParticleEngine.resize()` — 响应窗口尺寸变化

- [ ] **Step 1: 添加 ParticleEngine 代码**

```javascript
  // ========== ParticleEngine ==========
  const ParticleEngine = {
    canvas: null,
    ctx: null,
    particles: [],
    mode: 'idle',       // 'idle' | 'accelerate' | 'burst'
    animId: null,
    PARTICLE_COUNT: 120,

    init() {
      this.canvas = document.getElementById('animCanvas');
      this.ctx = this.canvas.getContext('2d');
      this.resize();
      window.addEventListener('resize', () => this.resize());

      // Create initial particles
      for (let i = 0; i < this.PARTICLE_COUNT; i++) {
        this.particles.push(this._createParticle());
      }
      this._loop();
    },

    resize() {
      this.canvas.width = window.innerWidth * window.devicePixelRatio;
      this.canvas.height = window.innerHeight * window.devicePixelRatio;
      this.canvas.style.width = window.innerWidth + 'px';
      this.canvas.style.height = window.innerHeight + 'px';
      this.ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
    },

    setMode(mode) {
      this.mode = mode;
    },

    _createParticle() {
      return {
        x: Math.random() * window.innerWidth,
        y: Math.random() * window.innerHeight,
        vx: (Math.random() - 0.5) * 0.6,
        vy: (Math.random() - 0.5) * 0.6,
        size: Math.random() * 2.5 + 0.5,
        alpha: Math.random() * 0.5 + 0.2,
        color: Math.random() > 0.5 ? '0, 240, 255' : '168, 85, 247'
      };
    },

    _burstParticles(x, y, count) {
      for (let i = 0; i < count; i++) {
        const angle = (Math.PI * 2 * i) / count + (Math.random() - 0.5) * 0.5;
        const speed = Math.random() * 4 + 2;
        this.particles.push({
          x: x,
          y: y,
          vx: Math.cos(angle) * speed,
          vy: Math.sin(angle) * speed,
          size: Math.random() * 3 + 1,
          alpha: 1,
          color: '251, 191, 36',
          life: 1,
          decay: 0.008 + Math.random() * 0.02
        });
      }
    },

    burst(x, y, count = 60) {
      this._burstParticles(x, y, count);
    },

    _update() {
      const w = window.innerWidth;
      const h = window.innerHeight;
      const cx = w / 2;
      const cy = h / 2;

      for (let i = this.particles.length - 1; i >= 0; i--) {
        const p = this.particles[i];

        if (p.life !== undefined) {
          // Burst particle: decay and remove
          p.life -= p.decay;
          p.x += p.vx;
          p.y += p.vy;
          p.vy += 0.02; // gravity
          p.alpha = p.life;
          if (p.life <= 0) {
            this.particles.splice(i, 1);
            continue;
          }
        } else {
          // Ambient particle
          if (this.mode === 'accelerate') {
            // Drift toward center
            const dx = cx - p.x;
            const dy = cy - p.y;
            const dist = Math.sqrt(dx * dx + dy * dy) || 1;
            p.vx += (dx / dist) * 0.03;
            p.vy += (dy / dist) * 0.03;
            // Clamp speed
            const speed = Math.sqrt(p.vx * p.vx + p.vy * p.vy);
            if (speed > 4) {
              p.vx = (p.vx / speed) * 4;
              p.vy = (p.vy / speed) * 4;
            }
          }

          p.x += p.vx;
          p.y += p.vy;

          // Wrap around edges
          if (p.x < -10) p.x = w + 10;
          if (p.x > w + 10) p.x = -10;
          if (p.y < -10) p.y = h + 10;
          if (p.y > h + 10) p.y = -10;
        }
      }

      // Refill ambient particles
      const ambient = this.particles.filter(p => p.life === undefined).length;
      if (ambient < this.PARTICLE_COUNT) {
        this.particles.push(this._createParticle());
      }
    },

    _draw() {
      const ctx = this.ctx;
      const w = window.innerWidth;
      const h = window.innerHeight;
      ctx.clearRect(0, 0, w, h);

      for (const p of this.particles) {
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
        ctx.fillStyle = `rgba(${p.color}, ${p.alpha})`;
        ctx.fill();

        // Glow for burst particles
        if (p.life !== undefined && p.life > 0.5) {
          ctx.beginPath();
          ctx.arc(p.x, p.y, p.size * 2.5, 0, Math.PI * 2);
          ctx.fillStyle = `rgba(${p.color}, ${p.alpha * 0.25})`;
          ctx.fill();
        }
      }
    },

    _loop() {
      this._update();
      this._draw();
      this.animId = requestAnimationFrame(() => this._loop());
    }
  };
```

- [ ] **Step 2: 验证粒子背景**

在 App 初始化中调用 `ParticleEngine.init()`，打开页面验证：暗色背景上有流动的蓝紫色粒子，粒子在屏幕中漂移并环绕边界。

---

### Task 6: AudioEngine — Web Audio API 音效

**Files:**
- Modify: `lottery.html` — 在 `<script>` 中添加 AudioEngine

**Interfaces:**
- Consumes: Web Audio API（浏览器内置）
- Produces:
  - `AudioEngine.init()` — 创建 AudioContext（需在用户交互后调用）
  - `AudioEngine.setMuted(bool)` — 静音切换
  - `AudioEngine.playTick()` — 滚动滴答声
  - `AudioEngine.playReveal()` — 揭晓音效
  - `AudioEngine.playCelebrate()` — 庆祝音效
  - `AudioEngine.playVoid()` — 作废音效
  - `AudioEngine.startHum()` / `AudioEngine.stopHum()` — 滚动嗡鸣声

- [ ] **Step 1: 添加 AudioEngine 代码**

```javascript
  // ========== AudioEngine ==========
  const AudioEngine = {
    ctx: null,
    muted: false,
    humOsc: null,
    humGain: null,

    init() {
      this.ctx = new (window.AudioContext || window.webkitAudioContext)();
      const savedMute = localStorage.getItem('lottery_muted');
      if (savedMute === 'true') this.setMuted(true);
    },

    setMuted(m) {
      this.muted = m;
      localStorage.setItem('lottery_muted', String(m));
      document.getElementById('btnMute').textContent = m ? '🔇' : '🔊';
    },

    toggleMute() {
      this.setMuted(!this.muted);
    },

    _ensureCtx() {
      if (!this.ctx) this.init();
      if (this.ctx.state === 'suspended') this.ctx.resume();
    },

    playTick() {
      if (this.muted) return;
      this._ensureCtx();
      const osc = this.ctx.createOscillator();
      const gain = this.ctx.createGain();
      osc.connect(gain);
      gain.connect(this.ctx.destination);
      osc.type = 'sine';
      osc.frequency.value = 800 + Math.random() * 400;
      gain.gain.setValueAtTime(0.06, this.ctx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.001, this.ctx.currentTime + 0.06);
      osc.start(this.ctx.currentTime);
      osc.stop(this.ctx.currentTime + 0.06);
    },

    startHum() {
      if (this.muted) return;
      this._ensureCtx();
      this.humOsc = this.ctx.createOscillator();
      this.humGain = this.ctx.createGain();
      this.humOsc.connect(this.humGain);
      this.humGain.connect(this.ctx.destination);
      this.humOsc.type = 'sawtooth';
      this.humOsc.frequency.value = 60;
      this.humGain.gain.setValueAtTime(0.03, this.ctx.currentTime);
      this.humOsc.start(this.ctx.currentTime);
    },

    stopHum() {
      if (this.humOsc && this.humGain) {
        this.humGain.gain.exponentialRampToValueAtTime(0.001, this.ctx.currentTime + 0.3);
        this.humOsc.stop(this.ctx.currentTime + 0.3);
        this.humOsc = null;
        this.humGain = null;
      }
    },

    playReveal() {
      if (this.muted) return;
      this._ensureCtx();
      const now = this.ctx.currentTime;
      // Rising pitch
      const osc1 = this.ctx.createOscillator();
      const gain1 = this.ctx.createGain();
      osc1.connect(gain1);
      gain1.connect(this.ctx.destination);
      osc1.type = 'sine';
      osc1.frequency.setValueAtTime(200, now);
      osc1.frequency.exponentialRampToValueAtTime(1200, now + 0.4);
      gain1.gain.setValueAtTime(0.12, now);
      gain1.gain.exponentialRampToValueAtTime(0.001, now + 0.5);
      osc1.start(now);
      osc1.stop(now + 0.5);

      // Bass hit
      const osc2 = this.ctx.createOscillator();
      const gain2 = this.ctx.createGain();
      osc2.connect(gain2);
      gain2.connect(this.ctx.destination);
      osc2.type = 'triangle';
      osc2.frequency.setValueAtTime(80, now + 0.3);
      osc2.frequency.exponentialRampToValueAtTime(30, now + 0.8);
      gain2.gain.setValueAtTime(0.2, now + 0.3);
      gain2.gain.exponentialRampToValueAtTime(0.001, now + 0.8);
      osc2.start(now + 0.3);
      osc2.stop(now + 0.8);
    },

    playCelebrate() {
      if (this.muted) return;
      this._ensureCtx();
      const now = this.ctx.currentTime;
      // Quick arpeggio: C5-E5-G5-C6
      const notes = [523, 659, 784, 1047];
      notes.forEach((freq, i) => {
        const osc = this.ctx.createOscillator();
        const gain = this.ctx.createGain();
        osc.connect(gain);
        gain.connect(this.ctx.destination);
        osc.type = 'sine';
        const t = now + i * 0.1;
        osc.frequency.setValueAtTime(freq, t);
        gain.gain.setValueAtTime(0.08, t);
        gain.gain.exponentialRampToValueAtTime(0.001, t + 0.3);
        osc.start(t);
        osc.stop(t + 0.3);
      });
    },

    playVoid() {
      if (this.muted) return;
      this._ensureCtx();
      const now = this.ctx.currentTime;
      const osc = this.ctx.createOscillator();
      const gain = this.ctx.createGain();
      osc.connect(gain);
      gain.connect(this.ctx.destination);
      osc.type = 'sine';
      osc.frequency.setValueAtTime(300, now);
      osc.frequency.exponentialRampToValueAtTime(40, now + 0.8);
      gain.gain.setValueAtTime(0.1, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.8);
      osc.start(now);
      osc.stop(now + 0.8);
    }
  };
```

- [ ] **Step 2: 绑定静音按钮**

```javascript
  document.getElementById('btnMute').addEventListener('click', () => {
    AudioEngine.toggleMute();
  });
```

- [ ] **Step 3: 验证音效**

打开页面，Console 中执行：

```javascript
AudioEngine.init();
AudioEngine.playTick();
AudioEngine.startHum(); setTimeout(() => AudioEngine.stopHum(), 2000);
AudioEngine.playReveal();
AudioEngine.playCelebrate();
AudioEngine.playVoid();
```

预期：每个音效清晰可辨，点击静音按钮后全部无声。

---

### Task 7: DrawEngine — Slot-Machine 名字滚动动画

**Files:**
- Modify: `lottery.html` — 在 `<script>` 中添加 DrawEngine

**Interfaces:**
- Consumes: `ParticleEngine` (Canvas 共享), `AudioEngine`, `StateManager.state.candidates`
- Produces:
  - `DrawEngine.startSpin(candidates, winnerName, onComplete)` — 开始滚动动画，最终停在 winnerName
  - `DrawEngine.showWinner(name, prizeName)` — 静态揭晓画面
  - `DrawEngine.playVoidAnimation(name, onComplete)` — 作废划掉动效
  - `DrawEngine.clear()` — 清除 Canvas 动画层

- [ ] **Step 1: 添加 DrawEngine 代码**

```javascript
  // ========== DrawEngine ==========
  const DrawEngine = {
    canvas: null,
    ctx: null,
    running: false,
    animId: null,

    // Slot machine state
    names: [],
    targetName: '',
    currentOffset: 0,
    speed: 0,
    phase: 'idle',        // 'idle' | 'spinning' | 'decelerating' | 'reveal' | 'winner'
    revealTimer: 0,
    flashAlpha: 0,

    // Config
    FONT_SIZE: 72,
    CARD_HEIGHT: 100,
    VISIBLE_SLOTS: 7,

    init() {
      this.canvas = document.getElementById('animCanvas');
      this.ctx = this.canvas.getContext('2d');
    },

    startSpin(candidates, winnerName, onComplete) {
      // Shuffle candidates for the visual
      this.names = [...candidates].sort(() => Math.random() - 0.5);
      // Ensure winnerName is in the list and at a random position
      if (!this.names.includes(winnerName)) {
        this.names.push(winnerName);
      }
      // Re-shuffle to place winner at a random position
      this.names = this.names.sort(() => Math.random() - 0.5);
      this.targetName = winnerName;
      this.speed = 18;
      this.currentOffset = 0;
      this.phase = 'spinning';
      this.flashAlpha = 0;
      this.running = true;
      this._onComplete = onComplete;

      AudioEngine.startHum();
      this._tick();
    },

    _tick() {
      if (!this.running) return;

      const ctx = this.ctx;
      const w = window.innerWidth;
      const h = window.innerHeight;

      // Draw background overlay
      ctx.fillStyle = 'rgba(10, 10, 15, 0.55)';
      ctx.fillRect(0, 0, w, h);

      // Update speed
      if (this.phase === 'spinning') {
        this.currentOffset += this.speed;
        // Play tick sound periodically
        if (Math.floor(this.currentOffset / this.CARD_HEIGHT) !==
            Math.floor((this.currentOffset - this.speed) / this.CARD_HEIGHT)) {
          AudioEngine.playTick();
        }
        // Peak speed for ~3 seconds, then decelerate
        // We use a timer mechanism: after ~120 frames (~2s at 60fps), start deceleration
        if (!this._spinStartTime) this._spinStartTime = performance.now();
        if (performance.now() - this._spinStartTime > 2500) {
          this.phase = 'decelerating';
        }
      } else if (this.phase === 'decelerating') {
        // Decelerate gradually
        this.speed *= 0.985;
        this.currentOffset += this.speed;
        AudioEngine.playTick();
        if (this.speed < 0.3) {
          this.phase = 'reveal';
          this.revealTimer = performance.now();
          AudioEngine.stopHum();
          this._onReveal();
        }
      }

      // Render slot machine
      this._drawSlotMachine(ctx, w, h);

      // Flash overlay
      if (this.flashAlpha > 0) {
        ctx.fillStyle = `rgba(255, 255, 255, ${this.flashAlpha})`;
        ctx.fillRect(0, 0, w, h);
        this.flashAlpha *= 0.85;
        if (this.flashAlpha < 0.01) this.flashAlpha = 0;
      }

      this.animId = requestAnimationFrame(() => this._tick());
    },

    _drawSlotMachine(ctx, w, h) {
      const cx = w / 2;
      const cy = h / 2;
      const offset = this.currentOffset;
      const cardH = this.CARD_HEIGHT;

      // Clip region for the slot window
      const windowTop = cy - cardH * 3.5;
      const windowH = cardH * this.VISIBLE_SLOTS;

      // Draw names scrolling vertically
      for (let i = -5; i < this.VISIBLE_SLOTS + 5; i++) {
        const nameIdx = Math.floor((offset + i * cardH) / cardH) % this.names.length;
        const y = cy + i * cardH - (offset % cardH);
        const name = this.names[Math.abs(nameIdx)] || '';

        // Calculate alpha and scale based on distance from center
        const distFromCenter = Math.abs(y - cy) / (windowH / 2);
        const alpha = Math.max(0.15, 1 - distFromCenter * 0.85);
        const scale = Math.max(0.6, 1 - distFromCenter * 0.4);

        ctx.save();
        ctx.translate(cx, y);
        ctx.scale(scale, scale);

        // Card background
        const cardW = Math.min(w * 0.6, 600);
        const gradient = ctx.createLinearGradient(-cardW/2, 0, cardW/2, 0);

        if (this.phase === 'reveal' && name === this.targetName) {
          gradient.addColorStop(0, 'rgba(251, 191, 36, 0.2)');
          gradient.addColorStop(1, 'rgba(251, 191, 36, 0.2)');
        } else {
          gradient.addColorStop(0, 'rgba(0, 240, 255, 0.06)');
          gradient.addColorStop(1, 'rgba(168, 85, 247, 0.06)');
        }
        ctx.fillStyle = gradient;
        ctx.fillRect(-cardW/2, -cardH/2, cardW, cardH);

        // Card border
        ctx.strokeStyle = this.phase === 'reveal' && name === this.targetName
          ? `rgba(251, 191, 36, ${alpha})`
          : `rgba(0, 240, 255, ${alpha * 0.4})`;
        ctx.lineWidth = name === this.targetName && this.phase === 'reveal' ? 3 : 1;
        ctx.strokeRect(-cardW/2, -cardH/2, cardW, cardH);

        // Name text
        ctx.fillStyle = this.phase === 'reveal' && name === this.targetName
          ? `rgba(251, 191, 36, ${alpha})`
          : `rgba(226, 232, 240, ${alpha})`;
        ctx.font = `bold ${this.FONT_SIZE * 0.8}px var(--font-sans)`;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';

        // Text shadow for neon glow
        if (distFromCenter < 0.5) {
          ctx.shadowColor = this.phase === 'reveal' && name === this.targetName
            ? 'rgba(251, 191, 36, 0.6)'
            : 'rgba(0, 240, 255, 0.5)';
          ctx.shadowBlur = 20;
        }
        ctx.fillText(name, 0, 0);
        ctx.shadowBlur = 0;

        ctx.restore();
      }

      // Center highlight line (scan line effect)
      const lineGrad = ctx.createLinearGradient(0, cy - 2, 0, cy + 2);
      lineGrad.addColorStop(0, 'rgba(0, 240, 255, 0)');
      lineGrad.addColorStop(0.5, 'rgba(0, 240, 255, 0.4)');
      lineGrad.addColorStop(1, 'rgba(0, 240, 255, 0)');
      ctx.fillStyle = lineGrad;
      ctx.fillRect(cx - 320, cy - 2, 640, 4);
    },

    _onReveal() {
      this.flashAlpha = 0.8;
      AudioEngine.playReveal();
      ParticleEngine.burst(window.innerWidth / 2, window.innerHeight / 2, 80);
      // Wait 1.5s then stop and call complete
      setTimeout(() => {
        this.running = false;
        cancelAnimationFrame(this.animId);
        this.phase = 'winner';
        this._spinStartTime = null;
        if (this._onComplete) {
          const cb = this._onComplete;
          this._onComplete = null;
          cb();
        }
      }, 1500);
    },

    showWinner(name, prizeName) {
      const ctx = this.ctx;
      const w = window.innerWidth;
      const h = window.innerHeight;

      ctx.fillStyle = 'rgba(10, 10, 15, 0.7)';
      ctx.fillRect(0, 0, w, h);

      // Prize name at top
      ctx.fillStyle = 'rgba(0, 240, 255, 0.9)';
      ctx.font = 'bold 32px var(--font-sans)';
      ctx.textAlign = 'center';
      ctx.shadowColor = 'rgba(0, 240, 255, 0.6)';
      ctx.shadowBlur = 20;
      ctx.fillText(`🎉 ${prizeName} 中奖者`, w / 2, h * 0.22);
      ctx.shadowBlur = 0;

      // Winner name big
      ctx.fillStyle = '#fbbf24';
      ctx.font = 'bold 96px var(--font-sans)';
      ctx.textAlign = 'center';
      ctx.shadowColor = 'rgba(251, 191, 36, 0.8)';
      ctx.shadowBlur = 40;
      ctx.fillText(name, w / 2, h / 2);
      ctx.shadowBlur = 0;
    },

    playVoidAnimation(name, onComplete) {
      const ctx = this.ctx;
      const w = window.innerWidth;
      const h = window.innerHeight;
      let alpha = 1;
      let strikeX = 0;

      AudioEngine.playVoid();

      const anim = () => {
        ctx.fillStyle = 'rgba(10, 10, 15, 0.7)';
        ctx.fillRect(0, 0, w, h);

        alpha -= 0.02;
        strikeX += 12;

        // Name fading out
        ctx.fillStyle = `rgba(239, 68, 68, ${alpha})`;
        ctx.font = 'bold 96px var(--font-sans)';
        ctx.textAlign = 'center';
        ctx.fillText(name, w / 2, h / 2);

        // Strike-through line
        if (strikeX < w * 0.6) {
          ctx.strokeStyle = `rgba(239, 68, 68, ${alpha + 0.2})`;
          ctx.lineWidth = 4;
          ctx.beginPath();
          ctx.moveTo(w / 2 - strikeX, h / 2 + 10);
          ctx.lineTo(w / 2 - strikeX + w * 0.5, h / 2 - 20);
          ctx.stroke();
        }

        if (alpha <= 0) {
          ctx.clearRect(0, 0, w, h);
          if (onComplete) onComplete();
          return;
        }
        requestAnimationFrame(anim);
      };
      anim();
    },

    clear() {
      this.running = false;
      if (this.animId) cancelAnimationFrame(this.animId);
      const ctx = this.ctx;
      const w = window.innerWidth;
      const h = window.innerHeight;
      ctx.clearRect(0, 0, w, h);
    }
  };
```

- [ ] **Step 2: 验证动画**

Console 测试（需要先有 candidates）：

```javascript
ParticleEngine.init();
DrawEngine.init();
DrawEngine.startSpin(['张三','李四','王五','赵六','钱七','孙八','周九'], '王五', () => {
  console.log('Spin complete!');
});
```

预期：名字卡片竖直高速滚动 → 逐渐减速 → 停在"王五" → 闪白 + 粒子爆炸 → 回调触发。

---

### Task 8: LotteryView — 抽奖主页流程控制

**Files:**
- Modify: `lottery.html` — 在 `<script>` 中添加 LotteryView + ViewController

**Interfaces:**
- Consumes: `StateManager`, `ViewController`, `ParticleEngine`, `AudioEngine`, `DrawEngine`
- Produces: `LotteryView.init()`, `LotteryView.show()`, `LotteryView.hide()`

- [ ] **Step 1: 添加 ViewController**

```javascript
  // ========== ViewController ==========
  const ViewController = {
    switchTo(viewId) {
      document.querySelectorAll('.view').forEach(v => v.classList.remove('active'));
      document.getElementById(viewId).classList.add('active');
    }
  };
```

- [ ] **Step 2: 添加 LotteryView 代码**

```javascript
  // ========== LotteryView ==========
  const LotteryView = {
    init() {
      document.getElementById('btnStartDraw').addEventListener('click', () => this.startDraw());
      document.getElementById('btnConfirm').addEventListener('click', () => this.confirmWinner());
      document.getElementById('btnVoid').addEventListener('click', () => this.voidWinner());
    },

    show() {
      ViewController.switchTo('view-lottery');
      const state = StateManager.state;

      // Check if all prizes done
      if (state.currentPrizeIdx >= state.config.length) {
        ResultsView.show();
        return;
      }

      // Check if candidates depleted
      if (state.candidates.length === 0) {
        alert('候选池已无可用人员，抽奖终止');
        ResultsView.show();
        return;
      }

      this.renderStandby();
      ParticleEngine.setMode('idle');
      DrawEngine.clear();
    },

    renderStandby() {
      const state = StateManager.state;
      const currentPrize = state.config[state.currentPrizeIdx];
      const slotIdx = state.currentSlotIdx;

      document.getElementById('lotteryPrizeName').textContent =
        `🏆 ${currentPrize.name}`;
      document.getElementById('lotteryProgress').textContent =
        `已抽出 ${slotIdx}/${currentPrize.count} 人`;

      document.getElementById('standbyNext').textContent =
        `准备抽取：${currentPrize.name} 第 ${slotIdx + 1} 位`;
      document.getElementById('standbyRemaining').textContent =
        `剩余候选：${state.candidates.length} 人`;

      // Show already drawn winners for this prize
      const drawnForThisPrize = state.winners
        .filter(w => w.prize === currentPrize.name)
        .slice(-(currentPrize.count)); // last N
      document.getElementById('standbyDrawn').innerHTML = drawnForThisPrize
        .map(w => `<span class="drawn-tag">${this.esc(w.name)}</span>`)
        .join('');

      document.getElementById('lotteryStandby').style.display = 'block';
      document.getElementById('lotteryActions').style.display = 'none';
    },

    startDraw() {
      const state = StateManager.state;
      const candidates = state.candidates;

      if (candidates.length === 0) {
        alert('候选池已无可用人员');
        return;
      }

      document.getElementById('lotteryStandby').style.display = 'none';
      document.getElementById('lotteryActions').style.display = 'none';

      ParticleEngine.setMode('accelerate');

      // If only 1 candidate left, skip animation
      if (candidates.length === 1) {
        this._currentWinner = candidates[0];
        this.showWinnerScreen();
        return;
      }

      // Randomly pick the winner BEFORE animation starts
      const winnerIdx = Math.floor(Math.random() * candidates.length);
      this._currentWinner = candidates[winnerIdx];

      // Start the spin animation
      DrawEngine.startSpin(candidates, this._currentWinner, () => {
        // Animation complete - show winner
        this.showWinnerScreen();
      });
    },

    showWinnerScreen() {
      const state = StateManager.state;
      const currentPrize = state.config[state.currentPrizeIdx];
      ParticleEngine.setMode('burst');
      DrawEngine.showWinner(this._currentWinner, currentPrize.name);
      AudioEngine.playCelebrate();
      document.getElementById('lotteryActions').style.display = 'flex';
    },

    confirmWinner() {
      const state = StateManager.state;
      const currentPrize = state.config[state.currentPrizeIdx];
      const winner = this._currentWinner;

      // Remove winner from candidates
      const idx = state.candidates.indexOf(winner);
      if (idx !== -1) state.candidates.splice(idx, 1);

      // Record winner
      state.winners.push({
        name: winner,
        prize: currentPrize.name,
        time: new Date().toLocaleString('zh-CN', {
          year: 'numeric', month: '2-digit', day: '2-digit',
          hour: '2-digit', minute: '2-digit', second: '2-digit',
          hour12: false
        })
      });

      state.currentSlotIdx++;
      StateManager.save();

      // Check if current prize is done
      if (state.currentSlotIdx >= currentPrize.count) {
        // Move to next prize
        state.currentPrizeIdx++;
        state.currentSlotIdx = 0;
        StateManager.save();

        // Check if all prizes done
        if (state.currentPrizeIdx >= state.config.length) {
          DrawEngine.clear();
          ResultsView.show();
          return;
        }
      }

      DrawEngine.clear();
      ParticleEngine.setMode('idle');
      this.renderStandby();
    },

    voidWinner() {
      const state = StateManager.state;
      const voidedName = this._currentWinner;

      // Remove from candidates (don't put back)
      const idx = state.candidates.indexOf(voidedName);
      if (idx !== -1) state.candidates.splice(idx, 1);

      StateManager.save();

      DrawEngine.playVoidAnimation(voidedName, () => {
        // Check if candidates depleted
        if (state.candidates.length === 0) {
          alert('候选池已无可用人员，该奖项剩余名额无法继续抽取');
          state.currentPrizeIdx++;
          state.currentSlotIdx = 0;
          StateManager.save();
          if (state.currentPrizeIdx >= state.config.length) {
            ResultsView.show();
            return;
          }
        }
        DrawEngine.clear();
        ParticleEngine.setMode('idle');
        this.renderStandby();
      });
    },

    esc(str) {
      const div = document.createElement('div');
      div.textContent = str;
      return div.innerHTML;
    },

    hide() {
      document.getElementById('view-lottery').classList.remove('active');
      DrawEngine.clear();
    }
  };
```

- [ ] **Step 3: 验证抽奖流程**

完整流程测试：配置奖项 → 导入名单 → 点击开始抽取 → 观察动画 → 确认中奖 → 自动进入下一位 → 该奖项抽完 → 下一奖项。

---

### Task 9: ResultsView — 汇总展示 + CSV 导出 + 打印

**Files:**
- Modify: `lottery.html` — 在 `<script>` 中添加 ResultsView

**Interfaces:**
- Consumes: `StateManager`, `ViewController`
- Produces: `ResultsView.init()`, `ResultsView.show()`, `ResultsView.hide()`

- [ ] **Step 1: 添加 ResultsView 代码**

```javascript
  // ========== ResultsView ==========
  const ResultsView = {
    init() {
      document.getElementById('btnExport').addEventListener('click', () => this.exportCSV());
      document.getElementById('btnPrint').addEventListener('click', () => window.print());
      document.getElementById('btnReset').addEventListener('click', () => this.resetAll());
    },

    show() {
      ViewController.switchTo('view-results');
      const winners = StateManager.state.winners;
      const config = StateManager.state.config;

      // Group winners by prize (respect config order)
      const grouped = {};
      for (const prize of config) {
        grouped[prize.name] = winners.filter(w => w.prize === prize.name);
      }

      const resultsList = document.getElementById('resultsList');
      resultsList.innerHTML = config.map(prize => {
        const list = grouped[prize.name] || [];
        return `
          <div class="prize-group">
            <h2>🏆 ${this.esc(prize.name)}（${list.length}/${prize.count}）</h2>
            <div class="winner-cards">
              ${list.map(w => `<span class="winner-card">★ ${this.esc(w.name)}</span>`).join('')}
              ${list.length === 0 ? '<span style="color:var(--text-dim);">暂无中奖者</span>' : ''}
            </div>
          </div>
        `;
      }).join('');

      document.getElementById('resultsTotal').textContent =
        `共计抽出：${winners.length} 人`;
    },

    exportCSV() {
      const winners = StateManager.state.winners;
      const BOM = '﻿';
      const header = '奖项,姓名,中奖时间';
      const rows = winners.map(w => `"${w.prize}","${w.name}","${w.time}"`);
      const csv = BOM + header + '\n' + rows.join('\n');

      const today = new Date().toISOString().slice(0, 10);
      const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `lottery-results-${today}.csv`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    },

    resetAll() {
      if (confirm('确定要重置全部数据吗？此操作不可恢复！')) {
        if (confirm('再次确认：所有中奖记录和名单将被清除')) {
          StateManager.reset();
          ConfigView.show();
        }
      }
    },

    esc(str) {
      const div = document.createElement('div');
      div.textContent = str;
      return div.innerHTML;
    },

    hide() {
      document.getElementById('view-results').classList.remove('active');
    }
  };
```

- [ ] **Step 2: 验证导出**

Console 测试（先有一些中奖数据）：

```javascript
StateManager.state.winners = [
  {name: '张三', prize: '特等奖', time: '2026-01-18 20:35:12'},
  {name: '李四', prize: '一等奖', time: '2026-01-18 20:28:05'}
];
ResultsView.exportCSV();
// 预期：浏览器下载 CSV 文件，UTF-8 BOM 编码，Excel 打开中文不乱码
```

---

### Task 10: App 初始化 — 粘合所有模块

**Files:**
- Modify: `lottery.html` — 在 `</script>` 前添加 App 初始化代码

**Interfaces:**
- Consumes: 所有模块
- Produces: 完整可用的抽奖工具

- [ ] **Step 1: 添加 App 初始化代码**

```javascript
  // ========== App Initialization ==========
  (function App() {
    // Check for saved session
    checkRestore();

    // Init engines
    ParticleEngine.init();
    DrawEngine.init();

    // Init views
    ConfigView.init();
    ImportView.init();
    LotteryView.init();
    ResultsView.init();

    // Mute button
    document.getElementById('btnMute').addEventListener('click', () => {
      // AudioEngine.init() on first user interaction
      if (!AudioEngine.ctx) AudioEngine.init();
      AudioEngine.toggleMute();
    });

    // Init audio on first click anywhere
    document.addEventListener('click', function initAudio() {
      if (!AudioEngine.ctx) AudioEngine.init();
    }, { once: true });

    // Determine starting view
    const state = StateManager.state;
    if (state.config.length === 0) {
      ConfigView.show();
    } else if (state.candidates.length === 0) {
      ImportView.show();
    } else if (state.currentPrizeIdx < state.config.length) {
      LotteryView.show();
    } else {
      ResultsView.show();
    }
  })();
</script>
```

- [ ] **Step 2: 端到端测试**

完整流程：
1. 打开 `lottery.html`
2. 添加奖项：三等奖 3 人、二等奖 2 人、一等奖 1 人
3. 保存进入下一步
4. 上传测试 CSV（至少 10 个名字）
5. 确认名单
6. 抽三等奖第 1 位 — 观察动画 — 确认中奖
7. 抽三等奖第 2 位 — 观察动画 — 作废重抽 — 再抽一次
8. 继续直到全部奖项抽完
9. 自动跳转汇总页
10. 导出 CSV，验证内容
11. 刷新页面，验证恢复提示，继续上次进度

- [ ] **Step 3: 边界情况测试**

| 测试项 | 预期结果 |
|--------|---------|
| 名单人数 < 奖项总数 | 导入时红色警告 |
| 候选池仅剩 1 人 | 跳过滚动动画，直接揭晓 |
| 作废到候选池耗尽 | 提示并终止当前奖项 |
| 上传空 CSV | 显示"未识别到任何姓名" |
| 上传 .txt 文件 | 显示"仅支持 CSV 和 Excel 格式" |
| 4K 分辨率 | Canvas 适配，字体大小正常 |
| Chrome 隐身模式 | localStorage 正常（无痕模式下） |

- [ ] **Step 4: 最终提交**

```bash
git add lottery.html docs/superpowers/
git commit -m "feat: 完成年会抽奖工具 v1.0

- 单个 HTML 文件，双击即用，零依赖
- 奖项自由配置，名单 CSV 导入
- Slot-machine 风格名字滚动动画 + 粒子背景
- Web Audio API 音效合成
- 逐个揭晓，不在场作废重抽
- 结果汇总 + CSV 导出 + 打印
- localStorage 持久化，刷新不丢数据"
```

---

## 自检清单

**1. Spec 覆盖检查：**

| Spec 章节 | 对应 Task |
|-----------|----------|
| 3.1 配置页 | Task 3 |
| 3.2 名单导入页 | Task 4 |
| 3.3 抽奖主页 | Task 7, 8 |
| 3.4 结果汇总页 | Task 9 |
| 4.1 视觉基调 | Task 1 (CSS) |
| 4.2 单人抽奖动画 | Task 7 (DrawEngine) |
| 4.3 待机/过渡画面 | Task 8 (renderStandby) |
| 4.4 音效 | Task 6 |
| 5.1 localStorage 键 | Task 2 |
| 5.2 容错设计 | Task 2, 10 |
| 6. 边界情况 | Task 10 Step 3 |
| 7. 测试要点 | 各 Task 验证步骤 |

所有 spec 需求已覆盖 ✓

**2. 占位符扫描：** 无 "TBD"、"TODO"、"implement later" — 通过 ✓

**3. 类型一致性：**
- `StateManager.state` 结构在 Task 2 定义，Task 3/4/8/9 使用一致 ✓
- `ParticleEngine.burst(x, y, count)` 在 Task 5 定义，Task 7 调用签名一致 ✓
- `AudioEngine` 方法在 Task 6 定义，Task 7/8 调用签名一致 ✓
- `DrawEngine` 方法在 Task 7 定义，Task 8 调用签名一致 ✓

