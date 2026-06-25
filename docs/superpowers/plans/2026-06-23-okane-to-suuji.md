# おかねとすうじ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 3桁の数と百/十/一円玉を声なし・指だけで往来し位取りを体感する教材アプリを完成させる。

**Architecture:** `logic.js` に純粋関数（数の分解・合成・正誤判定・出題）を集中させ `node --test` でテスト。`index.html` に全 CSS・DOM・イベント処理をインラインで書く（ビルド不要、GitHub Pages 直デプロイ）。かずよみ（~/Workspace/kazu-yomi/）の CSS・コイン画像・サウンドシステムを流用し、音声認識を削除してコイン操作と▲▼スピナーに置き換える。

**Tech Stack:** Vanilla HTML/CSS/JS, `node --test`（Node.js 20+）, GitHub Pages

## Global Constraints

- ビルドツール・npm 禁止（index.html 1ファイル＋ logic.js だけで動く）
- 声・SpeechRecognition は一切使わない
- 百の位は1〜9、十の位と一の位は0〜9（100〜999）
- 「できた！」ボタン押下でのみ正誤判定（自動判定しない）
- 不正解時は「× もういちど」のみ、位の指摘なし
- 数直線はバーの長さ（0から金額まで色が伸びる）で表現・入力には使わない
- コイン画像は ~/Workspace/kazu-yomi/imgs/ から流用
- 設計書: `docs/specs/2026-06-23-okane-to-suuji-design.md`

---

## ファイルマップ

| ファイル | 役割 |
|---------|------|
| `logic.js` | 純粋関数（splitDigits/coinsToNumber/checkAnswer/generateProblem）|
| `logic.test.mjs` | `node --test` 用テスト |
| `index.html` | 全 CSS・全 DOM・イベント処理・サウンド |
| `imgs/money_100.png` | かずよみから流用 |
| `imgs/money_10.png` | かずよみから流用 |
| `imgs/money_1.png` | かずよみから流用 |

---

## Task 1: Scaffolding + logic.js + テスト

**Files:**
- Create: `logic.js`
- Create: `logic.test.mjs`
- Copy: `imgs/money_100.png`, `imgs/money_10.png`, `imgs/money_1.png`

**Interfaces:**
- Produces:
  - `splitDigits(n: number): { h: number, t: number, o: number }` — n=438 → `{h:4, t:3, o:8}`
  - `coinsToNumber({ h, t, o }): number` — `{h:4,t:3,o:8}` → 438
  - `checkAnswer(answer: {h,t,o}, target: number): boolean`
  - `generateProblem(prev?: number): number` — 100〜999、prev と異なる値

- [ ] **Step 1: imgs をコピー**

```bash
cp ~/Workspace/kazu-yomi/imgs/money_100.png ~/Workspace/okane-to-suuji/imgs/
cp ~/Workspace/kazu-yomi/imgs/money_10.png  ~/Workspace/okane-to-suuji/imgs/
cp ~/Workspace/kazu-yomi/imgs/money_1.png   ~/Workspace/okane-to-suuji/imgs/
ls ~/Workspace/okane-to-suuji/imgs/
```

期待出力: `money_100.png  money_10.png  money_1.png`

- [ ] **Step 2: logic.test.mjs を書く（先に書く）**

`~/Workspace/okane-to-suuji/logic.test.mjs` を以下の内容で作成:

```mjs
import { test } from 'node:test';
import assert from 'node:assert/strict';
import pkg from './logic.js';
const { splitDigits, coinsToNumber, checkAnswer, generateProblem } = pkg;

// ── splitDigits ──────────────────────────────────────────
test('splitDigits: 438 → {h:4,t:3,o:8}', () => {
  assert.deepEqual(splitDigits(438), { h: 4, t: 3, o: 8 });
});
test('splitDigits: 100 → {h:1,t:0,o:0}', () => {
  assert.deepEqual(splitDigits(100), { h: 1, t: 0, o: 0 });
});
test('splitDigits: 999 → {h:9,t:9,o:9}', () => {
  assert.deepEqual(splitDigits(999), { h: 9, t: 9, o: 9 });
});
test('splitDigits: 205 → {h:2,t:0,o:5}', () => {
  assert.deepEqual(splitDigits(205), { h: 2, t: 0, o: 5 });
});

// ── coinsToNumber ────────────────────────────────────────
test('coinsToNumber: {h:4,t:3,o:8} → 438', () => {
  assert.equal(coinsToNumber({ h: 4, t: 3, o: 8 }), 438);
});
test('coinsToNumber: {h:1,t:0,o:0} → 100', () => {
  assert.equal(coinsToNumber({ h: 1, t: 0, o: 0 }), 100);
});
test('coinsToNumber: {h:9,t:9,o:9} → 999', () => {
  assert.equal(coinsToNumber({ h: 9, t: 9, o: 9 }), 999);
});

// ── checkAnswer ──────────────────────────────────────────
test('checkAnswer: 正解', () => {
  assert.equal(checkAnswer({ h: 4, t: 3, o: 8 }, 438), true);
});
test('checkAnswer: 不正解（十の位がちがう）', () => {
  assert.equal(checkAnswer({ h: 4, t: 2, o: 8 }, 438), false);
});
test('checkAnswer: 不正解（百の位がちがう）', () => {
  assert.equal(checkAnswer({ h: 3, t: 3, o: 8 }, 438), false);
});
test('checkAnswer: 0の位が両方0で正解', () => {
  assert.equal(checkAnswer({ h: 1, t: 0, o: 0 }, 100), true);
});

// ── generateProblem ──────────────────────────────────────
test('generateProblem: 100〜999 の範囲', () => {
  for (let i = 0; i < 100; i++) {
    const n = generateProblem();
    assert.ok(n >= 100 && n <= 999, `out of range: ${n}`);
  }
});
test('generateProblem: 百の位は1以上', () => {
  for (let i = 0; i < 100; i++) {
    const n = generateProblem();
    assert.ok(Math.floor(n / 100) >= 1, `hundreds digit is 0: ${n}`);
  }
});
test('generateProblem: prev と異なる値を返す', () => {
  let prev = 438;
  for (let i = 0; i < 30; i++) {
    const n = generateProblem(prev);
    assert.notEqual(n, prev);
    prev = n;
  }
});
```

- [ ] **Step 3: テストが失敗することを確認**

```bash
cd ~/Workspace/okane-to-suuji && node --test logic.test.mjs 2>&1 | head -5
```

期待: `Cannot find module './logic.js'` または `splitDigits is not a function`

- [ ] **Step 4: logic.js を書く**

`~/Workspace/okane-to-suuji/logic.js` を以下の内容で作成:

```js
(function (root) {
  function splitDigits(n) {
    return {
      h: Math.floor(n / 100) % 10,
      t: Math.floor(n / 10) % 10,
      o: n % 10,
    };
  }

  function coinsToNumber({ h, t, o }) {
    return h * 100 + t * 10 + o;
  }

  function checkAnswer(answer, target) {
    return coinsToNumber(answer) === target;
  }

  function generateProblem(prev) {
    let n;
    do {
      const h = 1 + Math.floor(Math.random() * 9);
      const t = Math.floor(Math.random() * 10);
      const o = Math.floor(Math.random() * 10);
      n = h * 100 + t * 10 + o;
    } while (n === prev);
    return n;
  }

  const api = { splitDigits, coinsToNumber, checkAnswer, generateProblem };
  if (typeof module !== 'undefined' && module.exports) module.exports = api;
  else Object.assign(root, api);
})(typeof window !== 'undefined' ? window : globalThis);
```

- [ ] **Step 5: テストを通す**

```bash
cd ~/Workspace/okane-to-suuji && node --test logic.test.mjs
```

期待: 全テスト PASS（`▶ generateProblem` 等が緑で出る）

- [ ] **Step 6: コミット**

```bash
cd ~/Workspace/okane-to-suuji && git init && git add logic.js logic.test.mjs imgs/
git commit -m "feat: add logic.js with tests, copy coin images"
```

---

## Task 2: スタート画面 HTML/CSS

**Files:**
- Create: `index.html`

**Interfaces:**
- Consumes: `logic.js` のグローバル（`splitDigits` 等）
- Produces: 静的スタート画面。モードボタンクリックでゲーム画面が出る準備（画面切替のみ、ゲームロジックは Task 3・4 で追加）

- [ ] **Step 1: index.html を作成（スタート画面 + CSS + 画面切替の骨格のみ）**

`~/Workspace/okane-to-suuji/index.html` を以下の内容で作成:

```html
<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>おかねとすうじ</title>
<style>
  :root {
    --bg: #f7f7f7;
    --panel: #ffffff;
    --shadow: 0 8px 20px rgba(0,0,0,.10);
    --radius: 16px;
    --dark: #111;
    --ok: #0f8a2b;
    --ng: #e0453a;
    --line: #222;

    /* 列カラー */
    --h-bg: #fff3e0; --h-fg: #e65100;
    --t-bg: #e3f2fd; --t-fg: #1565c0;
    --o-bg: #fce4ec; --o-fg: #c62828;
  }
  * { box-sizing: border-box; }
  body {
    margin: 0;
    font-family: system-ui, -apple-system, "Noto Sans JP", sans-serif;
    background: var(--bg);
    color: var(--dark);
  }
  .wrap { max-width: 520px; margin: 0 auto; padding: 12px; }
  .panel {
    background: var(--panel);
    border-radius: var(--radius);
    box-shadow: var(--shadow);
    padding: 14px 12px 20px;
  }

  /* タイトル行 */
  .titleRow { position:relative; min-height:44px; margin-bottom:14px; }
  .title { font-weight:950; font-size:20px; margin:0; text-align:center; line-height:44px; }
  .soundBtn {
    position:absolute; right:0; top:0;
    border:0; border-radius:14px;
    display:inline-flex; align-items:center; gap:6px;
    padding:10px 12px; font-size:14px; font-weight:900;
    cursor:pointer; background:#f1f1f1; color:var(--dark); line-height:1;
  }
  .soundBtn.off { background:#e9e9e9; color:#666; }

  /* ===== スタート画面 ===== */
  .startScreen {
    display:flex; flex-direction:column; align-items:center;
    padding:24px 16px 36px; gap:18px;
  }
  .startMsg { margin:0; font-size:1.05rem; font-weight:700; color:#555; text-align:center; }
  .howto {
    width:100%; background:#f5f7fa; border-radius:14px;
    padding:14px 16px; display:flex; flex-direction:column; gap:10px;
  }
  .howto-title { font-size:0.8rem; font-weight:700; color:#888; letter-spacing:.06em; margin:0; }
  .howto-steps { margin:0; padding:0; list-style:none; display:flex; flex-direction:column; gap:8px; }
  .howto-steps li { display:flex; align-items:flex-start; gap:10px; font-size:0.9rem; line-height:1.5; color:#333; }
  .howto-num {
    flex-shrink:0; width:22px; height:22px; border-radius:50%;
    background:var(--dark); color:#fff;
    font-size:0.75rem; font-weight:700;
    display:flex; align-items:center; justify-content:center;
    margin-top:1px;
  }
  .modeButtons { display:flex; flex-direction:column; gap:12px; width:100%; }
  .modeBtn {
    border:0; border-radius:16px; font-weight:950; cursor:pointer;
    padding:18px 20px; font-size:18px;
    color:#fff; box-shadow:0 4px 12px rgba(0,0,0,.15);
    transition:transform .1s, opacity .1s;
    text-align:center; line-height:1.3;
  }
  .modeBtn:active { transform:scale(.97); }
  .modeBtn.m1 { background:#e65100; }
  .modeBtn.m2 { background:#1565c0; }
  .modeBtn .sub { font-size:0.7rem; font-weight:700; opacity:.8; display:block; margin-top:2px; }

  /* ===== ゲーム画面 ===== */
  .gameScreen { display:none; }
  .gameScreen.active { display:block; }

  /* 位テーブル共通 */
  table.place { width:100%; border-collapse:collapse; table-layout:fixed; margin-bottom:10px; }
  table.place th {
    font-size:0.85rem; font-weight:700; padding:7px 4px; text-align:center;
    border-radius:8px 8px 0 0;
  }
  th.h { background:var(--h-bg); color:var(--h-fg); }
  th.t { background:var(--t-bg); color:var(--t-fg); }
  th.o { background:var(--o-bg); color:var(--o-fg); }
  table.place td { padding:4px 3px; vertical-align:middle; text-align:center; }
  td.h { background:var(--h-bg); }
  td.t { background:var(--t-bg); }
  td.o { background:var(--o-bg); }

  /* 問題数字（mode1） */
  .problemNum {
    text-align:center; margin:4px 0 12px;
    font-size:3.6rem; font-weight:950; letter-spacing:.06em; color:var(--dark);
  }

  /* コインエリア */
  .coinArea { min-height:80px; display:flex; justify-content:center; align-items:flex-end; padding:4px 0; }
  .coinGroup { display:flex; gap:6%; justify-content:center; align-items:flex-end; }
  .coinCol { display:flex; flex-direction:column-reverse; align-items:center; gap:3px; flex:0 0 46%; }
  .coinImg { width:100%; max-width:54px; object-fit:contain; user-select:none; pointer-events:none; }
  .emptyCoins { color:#bbb; font-size:.85rem; border:2px dashed #ccc; border-radius:10px; padding:5px 8px; }

  /* mode1: +/- ボタン */
  .coinCtrlBtn {
    border:0; border-radius:10px; font-weight:950; cursor:pointer;
    width:100%; padding:6px 0; font-size:1.5rem; line-height:1;
    background:#e8e8e8; color:var(--dark);
    transition:background .1s;
  }
  .coinCtrlBtn:active { background:#ccc; }

  /* mode2: スピナー */
  .spinner { display:flex; flex-direction:column; align-items:center; gap:2px; padding:4px 0; }
  .spinBtn {
    border:0; border-radius:8px; font-size:1.4rem; font-weight:900;
    cursor:pointer; background:#e8e8e8; padding:4px 0; width:100%;
    line-height:1.2; transition:background .1s;
  }
  .spinBtn:active { background:#ccc; }
  .spinDigit { font-size:2.4rem; font-weight:950; line-height:1.2; color:var(--dark); }

  /* できた！ボタン */
  .doneBtn {
    display:block; width:100%; border:0; border-radius:16px; font-weight:950;
    padding:16px; font-size:22px; cursor:pointer; margin-top:12px;
    background:var(--dark); color:#fff;
    box-shadow:0 5px 12px rgba(0,0,0,.18);
    transition:transform .1s;
  }
  .doneBtn:active { transform:scale(.98); }

  /* 結果表示 */
  .resultArea { text-align:center; min-height:2.8rem; margin-top:8px; }
  .result { font-size:2rem; font-weight:700; min-height:2rem; }
  .result.ok { color:var(--ok); }
  .result.ng { color:var(--ng); }

  @keyframes victory {
    0%  { transform:scale(0.3); opacity:0; }
    50% { transform:scale(1.35); opacity:1; }
    70% { transform:scale(0.92); }
    100%{ transform:scale(1); opacity:1; }
  }
  .result.victory { animation:victory .55s cubic-bezier(.36,.07,.19,.97) both; color:#e8a000; }

  .star { position:fixed; font-size:1.8rem; pointer-events:none; z-index:99; animation:flystar .7s ease-out forwards; }
  @keyframes flystar {
    0%  { opacity:1; transform:translate(0,0) scale(1); }
    100%{ opacity:0; transform:translate(var(--dx),var(--dy)) scale(0.4); }
  }

  /* 数直線 */
  .numberLine { margin-top:12px; display:none; }
  .numberLine.visible { display:block; }
  .nlTrack {
    position:relative; height:18px; background:#e8e8e8;
    border-radius:9px; overflow:hidden; margin-bottom:4px;
  }
  .nlBar {
    position:absolute; left:0; top:0; height:100%;
    background:#4a9eff; border-radius:9px;
    width:0%; transition:width .5s ease-out;
  }
  .nlLabels {
    display:flex; justify-content:space-between;
    font-size:0.65rem; color:#888; padding:0 2px;
  }
  .nlValue { text-align:center; font-size:1rem; font-weight:700; color:#4a9eff; margin-top:2px; }

  /* 戻るボタン */
  .backBtn {
    border:0; background:none; font-size:0.85rem; color:#888;
    cursor:pointer; padding:4px 0; margin-bottom:8px;
    display:flex; align-items:center; gap:4px;
  }
  .backBtn:active { opacity:.6; }

  @media (max-width: 480px) {
    .wrap { padding:6px; }
    .panel { padding:10px 8px 16px; }
    .title { font-size:17px; }
    .problemNum { font-size:2.8rem; }
    .modeBtn { font-size:16px; padding:15px 16px; }
    .doneBtn { font-size:19px; padding:14px; }
    .spinDigit { font-size:2rem; }
  }
</style>
</head>
<body>
<div class="wrap">
  <div class="panel">
    <div class="titleRow">
      <p class="title">おかねとすうじ</p>
      <button id="soundBtn" class="soundBtn" type="button">
        <span id="soundIcon">🔊</span>
        <span id="soundLabel">おと ON</span>
      </button>
    </div>

    <!-- スタート画面 -->
    <div class="startScreen" id="startScreen">
      <p class="startMsg">モードをえらんでね</p>
      <div class="howto">
        <p class="howto-title">あそびかた</p>
        <ul class="howto-steps">
          <li>
            <span class="howto-num">①</span>
            <span><strong>すうじ→おかね</strong>：数字を見て、コインを正しい枚数に合わせよう</span>
          </li>
          <li>
            <span class="howto-num">②</span>
            <span><strong>おかね→すうじ</strong>：コインを数えて、▲▼で数字を合わせよう</span>
          </li>
          <li>
            <span class="howto-num">③</span>
            <span>できたら「<strong>できた！</strong>」を押そう</span>
          </li>
        </ul>
      </div>
      <div class="modeButtons">
        <button class="modeBtn m1" id="startMode1" type="button">
          すうじ → おかね
          <span class="sub">数字を見てコインをならべる</span>
        </button>
        <button class="modeBtn m2" id="startMode2" type="button">
          おかね → すうじ
          <span class="sub">コインを数えて数字を合わせる</span>
        </button>
      </div>
    </div>

    <!-- ゲーム画面（Task 3・4 で中身を追加） -->
    <div class="gameScreen" id="gameScreen">
      <button class="backBtn" id="backBtn" type="button">← もどる</button>
      <!-- mode1 / mode2 の中身は Task 3・4 で挿入 -->
    </div>

  </div>
</div>

<script src="logic.js"></script>
<script>
  // ===== サウンド =====
  let soundOn = localStorage.getItem('okaneToSuujiSoundOn') !== 'off';
  let audioCtx = null;
  function getCtx() {
    const Cls = window.AudioContext || window.webkitAudioContext;
    if (!Cls) return null;
    if (!audioCtx) audioCtx = new Cls();
    if (audioCtx.state === 'suspended') audioCtx.resume();
    return audioCtx;
  }
  function beep({ freq=880, dur=0.08, type='sine', gain=0.03, when=0 }) {
    if (!soundOn) return;
    const ctx = getCtx(); if (!ctx) return;
    const now = ctx.currentTime + when;
    const osc = ctx.createOscillator();
    const amp = ctx.createGain();
    osc.type = type; osc.frequency.setValueAtTime(freq, now);
    amp.gain.setValueAtTime(0.0001, now);
    amp.gain.exponentialRampToValueAtTime(gain, now + 0.01);
    amp.gain.exponentialRampToValueAtTime(0.0001, now + dur);
    osc.connect(amp); amp.connect(ctx.destination);
    osc.start(now); osc.stop(now + dur + 0.02);
  }
  function playTap()     { beep({ freq:440, dur:0.04, gain:0.02 }); }
  function playVictory() {
    beep({ freq:880,  dur:0.08, gain:0.035, when:0    });
    beep({ freq:1100, dur:0.08, gain:0.03,  when:0.09 });
    beep({ freq:1320, dur:0.12, gain:0.035, when:0.18 });
  }
  function playWrong() { beep({ freq:180, dur:0.12, type:'triangle', gain:0.03 }); }

  function updateSoundBtn() {
    document.getElementById('soundBtn').classList.toggle('off', !soundOn);
    document.getElementById('soundIcon').textContent = soundOn ? '🔊' : '🔇';
    document.getElementById('soundLabel').textContent = soundOn ? 'おと ON' : 'おと OFF';
  }
  document.getElementById('soundBtn').addEventListener('click', () => {
    soundOn = !soundOn;
    localStorage.setItem('okaneToSuujiSoundOn', soundOn ? 'on' : 'off');
    updateSoundBtn();
  });
  updateSoundBtn();

  // ===== 画面切替 =====
  function showStart() {
    document.getElementById('startScreen').style.display = '';
    document.getElementById('gameScreen').classList.remove('active');
  }
  function showGame() {
    document.getElementById('startScreen').style.display = 'none';
    document.getElementById('gameScreen').classList.add('active');
  }

  document.getElementById('backBtn').addEventListener('click', showStart);

  // モードボタン（Task 3・4 で startGame(mode) を実装）
  document.getElementById('startMode1').addEventListener('click', () => startGame('mode1'));
  document.getElementById('startMode2').addEventListener('click', () => startGame('mode2'));

  function startGame(mode) {
    showGame();
    // Task 3・4 で実装
    console.log('startGame:', mode);
  }
</script>
</body>
</html>
```

- [ ] **Step 2: ブラウザで開いてスタート画面を目視確認**

```bash
open ~/Workspace/okane-to-suuji/index.html
```

確認事項:
- タイトル「おかねとすうじ」表示
- 「すうじ→おかね」「おかね→すうじ」の2ボタン（オレンジ・青）
- あそびかた3ステップ表示
- 音ON/OFFボタン右上
- ボタン押下でコンソールに `startGame: mode1` が出る（DevTools で確認）

- [ ] **Step 3: コミット**

```bash
cd ~/Workspace/okane-to-suuji && git add index.html
git commit -m "feat: add start screen HTML/CSS"
```

---

## Task 3: すうじ→おかね モード（コイン操作 UI）

**Files:**
- Modify: `index.html` — gameScreen 内に mode1 HTML 追加、startGame/renderCoins 実装

**Interfaces:**
- Consumes: `splitDigits(target)`, `generateProblem(prev)`
- Produces: mode1 でコインを増減できる状態。「できた！」は押せるが判定は Task 5 で実装

- [ ] **Step 1: gameScreen 内に mode1 HTML を追加**

`index.html` の `<!-- mode1 / mode2 の中身は Task 3・4 で挿入 -->` コメントを削除し、以下に置き換える:

```html
      <!-- mode1: すうじ→おかね -->
      <div id="mode1Screen" style="display:none">
        <div class="problemNum" id="m1ProblemNum">438</div>
        <table class="place">
          <thead>
            <tr>
              <th class="h">百のくらい</th>
              <th class="t">十のくらい</th>
              <th class="o">一のくらい</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td class="h"><button class="coinCtrlBtn" data-place="h" data-delta="1">＋</button></td>
              <td class="t"><button class="coinCtrlBtn" data-place="t" data-delta="1">＋</button></td>
              <td class="o"><button class="coinCtrlBtn" data-place="o" data-delta="1">＋</button></td>
            </tr>
            <tr>
              <td class="h"><div class="coinArea" id="m1CoinsH"></div></td>
              <td class="t"><div class="coinArea" id="m1CoinsT"></div></td>
              <td class="o"><div class="coinArea" id="m1CoinsO"></div></td>
            </tr>
            <tr>
              <td class="h"><button class="coinCtrlBtn" data-place="h" data-delta="-1">−</button></td>
              <td class="t"><button class="coinCtrlBtn" data-place="t" data-delta="-1">−</button></td>
              <td class="o"><button class="coinCtrlBtn" data-place="o" data-delta="-1">−</button></td>
            </tr>
          </tbody>
        </table>
        <button class="doneBtn" id="m1DoneBtn" type="button">できた！</button>
        <div class="resultArea"><div class="result" id="m1Result"></div></div>
        <div class="numberLine" id="m1NumberLine">
          <div class="nlTrack"><div class="nlBar" id="m1NlBar"></div></div>
          <div class="nlLabels"><span>0</span><span>500</span><span>1000</span></div>
          <div class="nlValue" id="m1NlValue"></div>
        </div>
      </div>

      <!-- mode2: おかね→すうじ（Task 4 で追加） -->
      <div id="mode2Screen" style="display:none">
      </div>
```

- [ ] **Step 2: mode1 の状態管理とコイン描画を実装**

`index.html` の `<script>` 内の `startGame` 関数と、その下に以下を追加（既存の `console.log` 行を置き換える）:

```js
  // ===== 共通状態 =====
  let currentMode = null;
  let targetNum   = 0;
  let userCoins   = { h: 0, t: 0, o: 0 };
  let userDigits  = { h: 0, t: 0, o: 0 };

  const COIN_SRC = {
    h: 'imgs/money_100.png',
    t: 'imgs/money_10.png',
    o: 'imgs/money_1.png',
  };

  function renderCoinArea(el, coinSrc, count) {
    el.innerHTML = '';
    if (count === 0) {
      el.innerHTML = '<span class="emptyCoins">０まい</span>';
      return;
    }
    const group = document.createElement('div');
    group.className = 'coinGroup';
    (count <= 5 ? [count] : [5, count - 5]).forEach(n => {
      const col = document.createElement('div');
      col.className = 'coinCol';
      for (let i = 0; i < n; i++) {
        const img = document.createElement('img');
        img.className = 'coinImg'; img.src = coinSrc; img.alt = '';
        col.appendChild(img);
      }
      group.appendChild(col);
    });
    el.appendChild(group);
  }

  function renderMode1() {
    renderCoinArea(document.getElementById('m1CoinsH'), COIN_SRC.h, userCoins.h);
    renderCoinArea(document.getElementById('m1CoinsT'), COIN_SRC.t, userCoins.t);
    renderCoinArea(document.getElementById('m1CoinsO'), COIN_SRC.o, userCoins.o);
  }

  function startGame(mode) {
    showGame();
    currentMode = mode;
    targetNum = generateProblem();

    document.getElementById('mode1Screen').style.display = mode === 'mode1' ? '' : 'none';
    document.getElementById('mode2Screen').style.display = mode === 'mode2' ? '' : 'none';

    if (mode === 'mode1') {
      userCoins = { h: 0, t: 0, o: 0 };
      document.getElementById('m1ProblemNum').textContent = targetNum;
      document.getElementById('m1Result').textContent = '';
      document.getElementById('m1Result').className = 'result';
      document.getElementById('m1NumberLine').classList.remove('visible');
      renderMode1();
    }
  }

  // mode1: +/- ボタン
  document.querySelectorAll('#mode1Screen .coinCtrlBtn').forEach(btn => {
    btn.addEventListener('click', () => {
      const place = btn.dataset.place;
      const delta = parseInt(btn.dataset.delta, 10);
      const next = userCoins[place] + delta;
      if (next < 0 || next > 9) return;
      userCoins[place] = next;
      playTap();
      renderMode1();
    });
  });
```

- [ ] **Step 3: ブラウザで目視確認**

```bash
open ~/Workspace/okane-to-suuji/index.html
```

確認事項:
- 「すうじ→おかね」ボタンを押すとゲーム画面が開く
- 3桁の数字（100〜999）が大きく表示される
- ＋を押すとコインが増える（1〜5枚=1列、6枚以上=2列）
- −を押すとコインが減る
- 0枚のとき「０まい」表示
- 9枚で＋を押しても変わらない、0枚で−を押しても変わらない
- 「もどる」でスタート画面に戻る

- [ ] **Step 4: コミット**

```bash
cd ~/Workspace/okane-to-suuji && git add index.html
git commit -m "feat: add mode1 coin manipulation UI"
```

---

## Task 4: おかね→すうじ モード（スピナー UI）

**Files:**
- Modify: `index.html` — mode2Screen に HTML 追加、mode2 の状態管理・描画を実装

**Interfaces:**
- Consumes: `splitDigits(target)`, `generateProblem(prev)`
- Produces: mode2 でコイン表示とスピナー操作ができる状態。「できた！」は押せるが判定は Task 5 で実装

- [ ] **Step 1: mode2Screen の中身を追加**

`index.html` の `<div id="mode2Screen" style="display:none">` と `</div>` の間を以下に置き換える:

```html
        <table class="place">
          <thead>
            <tr>
              <th class="h">百のくらい</th>
              <th class="t">十のくらい</th>
              <th class="o">一のくらい</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td class="h"><div class="coinArea" id="m2CoinsH"></div></td>
              <td class="t"><div class="coinArea" id="m2CoinsT"></div></td>
              <td class="o"><div class="coinArea" id="m2CoinsO"></div></td>
            </tr>
            <tr>
              <td class="h">
                <div class="spinner">
                  <button class="spinBtn" data-place="h" data-delta="1">▲</button>
                  <div class="spinDigit" id="m2DigitH">0</div>
                  <button class="spinBtn" data-place="h" data-delta="-1">▼</button>
                </div>
              </td>
              <td class="t">
                <div class="spinner">
                  <button class="spinBtn" data-place="t" data-delta="1">▲</button>
                  <div class="spinDigit" id="m2DigitT">0</div>
                  <button class="spinBtn" data-place="t" data-delta="-1">▼</button>
                </div>
              </td>
              <td class="o">
                <div class="spinner">
                  <button class="spinBtn" data-place="o" data-delta="1">▲</button>
                  <div class="spinDigit" id="m2DigitO">0</div>
                  <button class="spinBtn" data-place="o" data-delta="-1">▼</button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
        <button class="doneBtn" id="m2DoneBtn" type="button">できた！</button>
        <div class="resultArea"><div class="result" id="m2Result"></div></div>
        <div class="numberLine" id="m2NumberLine">
          <div class="nlTrack"><div class="nlBar" id="m2NlBar"></div></div>
          <div class="nlLabels"><span>0</span><span>500</span><span>1000</span></div>
          <div class="nlValue" id="m2NlValue"></div>
        </div>
```

- [ ] **Step 2: mode2 の描画・スピナーイベントを実装**

`index.html` の `</script>` 直前（`updateSoundBtn()` 行の後）に以下を追加:

```js
  function renderMode2() {
    const digits = splitDigits(targetNum);
    renderCoinArea(document.getElementById('m2CoinsH'), COIN_SRC.h, digits.h);
    renderCoinArea(document.getElementById('m2CoinsT'), COIN_SRC.t, digits.t);
    renderCoinArea(document.getElementById('m2CoinsO'), COIN_SRC.o, digits.o);
    document.getElementById('m2DigitH').textContent = userDigits.h;
    document.getElementById('m2DigitT').textContent = userDigits.t;
    document.getElementById('m2DigitO').textContent = userDigits.o;
  }

  document.querySelectorAll('#mode2Screen .spinBtn').forEach(btn => {
    btn.addEventListener('click', () => {
      const place = btn.dataset.place;
      const delta = parseInt(btn.dataset.delta, 10);
      userDigits[place] = (userDigits[place] + delta + 10) % 10;
      playTap();
      document.getElementById(`m2Digit${place.toUpperCase()}`).textContent = userDigits[place];
    });
  });
```

また `startGame` 内の `if (mode === 'mode1') { ... }` ブロックの直後に以下を追加:

```js
    if (mode === 'mode2') {
      userDigits = { h: 0, t: 0, o: 0 };
      document.getElementById('m2Result').textContent = '';
      document.getElementById('m2Result').className = 'result';
      document.getElementById('m2NumberLine').classList.remove('visible');
      renderMode2();
    }
```

- [ ] **Step 3: ブラウザで目視確認**

```bash
open ~/Workspace/okane-to-suuji/index.html
```

確認事項:
- 「おかね→すうじ」ボタンを押すとゲーム画面が開く
- 百/十/一のコインが正しい枚数で表示される（問題）
- ▲▼でスピナーの数字が変わる（0→9→0 ラップアラウンド）
- コインは固定（タップしても変わらない）
- 「もどる」でスタート画面に戻り、再度押すと新しい問題が出る

- [ ] **Step 4: コミット**

```bash
cd ~/Workspace/okane-to-suuji && git add index.html
git commit -m "feat: add mode2 spinner UI"
```

---

## Task 5: 正誤判定 + サウンド + 星エフェクト

**Files:**
- Modify: `index.html` — 「できた！」ボタンに判定ロジックを接続

**Interfaces:**
- Consumes: `checkAnswer(answer, target)`, `playVictory()`, `playWrong()`
- Produces: 正解→「○ せいかい！」＋星エフェクト＋勝利音。不正解→「× もういちど」＋低音

- [ ] **Step 1: 星エフェクト関数を追加**

`</script>` 直前に追加:

```js
  function launchStars() {
    const dirs = [[-90,-80],[90,-80],[0,-100],[-120,-40],[120,-40],[-60,-110],[60,-110],[0,-120]];
    const cx = window.innerWidth / 2, cy = window.innerHeight * 0.35;
    dirs.forEach(([dx, dy], i) => {
      const s = document.createElement('div');
      s.className = 'star';
      s.textContent = ['★','✨','⭐'][i % 3];
      s.style.left = cx + 'px'; s.style.top = cy + 'px';
      s.style.setProperty('--dx', dx + 'px');
      s.style.setProperty('--dy', dy + 'px');
      s.style.animationDelay = (i * 0.04) + 's';
      document.body.appendChild(s);
      setTimeout(() => s.remove(), 900);
    });
  }
```

- [ ] **Step 2: onCorrect / onWrong / handleDone を実装**

```js
  function onCorrect(resultEl, numberLineId, barId, valueId) {
    resultEl.textContent = '○ せいかい！';
    resultEl.className = 'result victory';
    playVictory();
    launchStars();
    // 数直線アニメ（Task 6 で実装。今は呼ぶだけ）
    showNumberLine(numberLineId, barId, valueId, targetNum);
  }

  function onWrong(resultEl) {
    resultEl.textContent = '× もういちど';
    resultEl.className = 'result ng';
    playWrong();
  }

  function handleDone(mode) {
    if (mode === 'mode1') {
      const correct = checkAnswer(userCoins, targetNum);
      const resultEl = document.getElementById('m1Result');
      if (correct) onCorrect(resultEl, 'm1NumberLine', 'm1NlBar', 'm1NlValue');
      else onWrong(resultEl);
    } else {
      const correct = checkAnswer(userDigits, targetNum);
      const resultEl = document.getElementById('m2Result');
      if (correct) onCorrect(resultEl, 'm2NumberLine', 'm2NlBar', 'm2NlValue');
      else onWrong(resultEl);
    }
  }

  document.getElementById('m1DoneBtn').addEventListener('click', () => handleDone('mode1'));
  document.getElementById('m2DoneBtn').addEventListener('click', () => handleDone('mode2'));

  // showNumberLine は Task 6 で実装。スタブを置く
  function showNumberLine(nlId, barId, valueId, value) {
    // Task 6 で実装
  }
```

- [ ] **Step 3: ブラウザで正誤判定を確認**

```bash
open ~/Workspace/okane-to-suuji/index.html
```

確認事項（mode1）:
- コインを正しく並べて「できた！」→「○ せいかい！」（黄）＋星が飛ぶ＋勝利音
- わざと間違えて「できた！」→「× もういちど」（赤）＋低音
- 不正解後もコインは変わらない（自分で直せる）

確認事項（mode2）:
- 全桁を正しくセットして「できた！」→「○ せいかい！」＋星＋音
- 間違えて「できた！」→「× もういちど」

- [ ] **Step 4: コミット**

```bash
cd ~/Workspace/okane-to-suuji && git add index.html
git commit -m "feat: add answer check, sound effects, star animation"
```

---

## Task 6: 数直線アニメーション + 次問題サイクル

**Files:**
- Modify: `index.html` — `showNumberLine` 実装、正解後に自動で次問題へ

**Interfaces:**
- Consumes: `generateProblem(prev)`
- Produces: 正解後にバーが0から金額まで伸びる（0.5秒）→ 1.5秒後に次の問題へ

- [ ] **Step 1: showNumberLine を実装**

Task 5 で置いたスタブ `function showNumberLine(...) { // Task 6 で実装 }` を以下で置き換える:

```js
  function showNumberLine(nlId, barId, valueId, value) {
    const nlEl  = document.getElementById(nlId);
    const barEl = document.getElementById(barId);
    const valEl = document.getElementById(valueId);

    // バーをリセット（transition なしで 0 に）
    barEl.style.transition = 'none';
    barEl.style.width = '0%';
    nlEl.classList.add('visible');
    valEl.textContent = '';

    // 1フレーム待ってから transition 付きで伸ばす
    requestAnimationFrame(() => requestAnimationFrame(() => {
      barEl.style.transition = 'width 0.5s ease-out';
      barEl.style.width = (value / 1000 * 100).toFixed(2) + '%';
      valEl.textContent = value + 'えん';
    }));

    // 1.5秒後に次問題
    setTimeout(() => {
      nlEl.classList.remove('visible');
      barEl.style.transition = 'none';
      barEl.style.width = '0%';
      nextProblem();
    }, 1500);
  }
```

- [ ] **Step 2: nextProblem を実装**

`showNumberLine` の直後に追加:

```js
  function nextProblem() {
    const prev = targetNum;
    targetNum = generateProblem(prev);

    if (currentMode === 'mode1') {
      userCoins = { h: 0, t: 0, o: 0 };
      document.getElementById('m1ProblemNum').textContent = targetNum;
      document.getElementById('m1Result').textContent = '';
      document.getElementById('m1Result').className = 'result';
      renderMode1();
    } else {
      userDigits = { h: 0, t: 0, o: 0 };
      document.getElementById('m2Result').textContent = '';
      document.getElementById('m2Result').className = 'result';
      renderMode2();
    }
  }
```

- [ ] **Step 3: ブラウザで通しテスト**

```bash
open ~/Workspace/okane-to-suuji/index.html
```

確認事項（mode1）:
- 正解すると青いバーが 0 から正解金額の位置まで 0.5 秒で伸びる
- バー右上に「438えん」のような表示が出る
- 1.5秒後にコインが全部0にリセットされ新しい問題が出る
- 連続5問正解しても新しい問題が出続ける
- 同じ数字が連続して出ない

確認事項（mode2）:
- 同様に数直線が出て次問題へ

確認事項（両モード）:
- 不正解後も「できた！」を何度押せる（数直線は出ない）
- 「もどる」→モード選択 → 再度モード選択で新しい問題が出る

- [ ] **Step 4: 最終コミット**

```bash
cd ~/Workspace/okane-to-suuji && git add index.html
git commit -m "feat: add number line animation and auto next-problem cycle"
```

---

## 自己レビューメモ

### Spec カバレッジ確認

| 設計書要件 | 対応タスク |
|-----------|-----------|
| 声なし・指だけ | Task 3（コイン操作）・Task 4（スピナー） |
| 2モード選択 | Task 2（スタート画面） |
| 「できた！」で判定 | Task 5 |
| 不正解=「× もういちど」のみ | Task 5 |
| 正解後に横バー数直線 | Task 6 |
| 1.5秒後に次問題 | Task 6 |
| 百の位1以上・100〜999 | Task 1（logic.js） |
| コイン画像かずよみ流用 | Task 1 |
| 列ごとの色分け | Task 2（CSS --h-bg/--t-bg/--o-bg） |
| 効果音ON/OFF | Task 2（soundBtn） |
| 0枚=「０まい」 | Task 3（renderCoinArea） |
| 各位9枚上限・0下限 | Task 3（coinCtrlBtn handler） |
| スピナーラップアラウンド | Task 4（+10)%10） |
| 数直線バーの長さで表現 | Task 6 |
| 同じ問題が連続しない | Task 6（generateProblem(prev)） |

### プレースホルダーなし ✅

### 型・関数名の一貫性 ✅
- `userCoins: {h,t,o}` → `checkAnswer(userCoins, targetNum)` ← logic.js の `checkAnswer(answer, target)` と一致
- `userDigits: {h,t,o}` → `checkAnswer(userDigits, targetNum)` ← 同上
- `splitDigits(targetNum)` → `{h,t,o}` ← logic.js と一致
