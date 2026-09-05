# 熟成条件の編集と、写真つき日誌 — 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 仕込み内容を後から直せるようにし、日付つきのメモと写真を記録できるようにする。

**Architecture:** `batch.weights` を `batch.logs`（日付・重量・メモ・写真IDの配列）に置き換える。
写真の実体は localStorage ではなく IndexedDB の別ストアに Blob で置き、`logs` はIDだけを持つ。
UIは「記録する」モーダル（重量・メモ・写真を1つにまとめる）と「編集」モーダルの2つを追加し、
既存の「重量を記録」モーダルを前者で置き換える。

**Tech Stack:** 素の HTML / CSS / JavaScript（フレームワーク・ビルド・依存パッケージなし）。
localStorage、IndexedDB、Canvas、`<details>`。すべて単一の `index.html` に内包する。

**設計書:** `docs/superpowers/specs/2026-09-06-edit-and-log-design.md`

## Global Constraints

- 配色は3色のみ: 生成り `#FAF5EE` / 濃茶 `#3B2A22` / パンチェッタローズ `#B14A5A`。補助色は `#F1E8DC`（`--cream-deep`）と `#E7CBCF`（`--rose-soft`）。**新しい色を足さない**
- `alert()` `confirm()` `prompt()` は使わない。確認はアプリ内モーダル、通知は `toast()`
- 単一の `index.html` に内包。ビルドなし、外部通信なし、CDN読み込みなし
- スマホ縦画面ファースト。最大幅480px
- 日数カウントは仕込み当日を「0日目」とする
- UI文言はすべて日本語
- マスコットは `pig(size, mood)` の自前SVG。絵文字でマスコットを描かない（ボタン内の 🥓 ＋ 📖 🔔 は既存の流儀なので踏襲してよい）
- 新規IDは `uid()` で作る。`Date.now().toString(36)` 単独は使わない（同一ミリ秒で衝突するため）

## テスト方針（この計画で「テスト」と呼ぶもの）

このプロジェクトには意図的にビルド工程もパッケージ依存も無い。Jest等を入れると
`node_modules` とビルドが発生し、CLAUDE.md の「ビルド不要の静的サイト」という前提が崩れる。
そのため**実ブラウザのコンソールで実行する検証コード**をテストとして扱う。
各タスクは「先に失敗する検証を書いて実行 → 実装 → 通ることを確認」の順で進める。

**検証の実行手順（全タスク共通）:**

1. ブラウザで `index.html` を開く（`file://` で可。IndexedDB も動く）
2. DevTools のコンソールを開く
3. タスクに書かれた検証コードを貼って実行する
4. 期待される出力と照合する

検証コードは `console.assert` ではなく **明示的に `OK` / `NG` を返す式**で書く
（`console.assert` は通ったとき無言なので、実行したのか分かりにくい）。

**テスト用データを入れる/消す（必要なとき使う）:**

```js
// 旧形式（weights）のデータを1件仕込む — 移行の検証に使う
localStorage.setItem('pancetta-batches', JSON.stringify([{
  id:'testold', name:'移行テスト', startDate:'2026-08-25', d1:5, d2:7,
  saltPct:3, sugarPct:1, meatWeight:460, spice:'黒コショウ',
  weights:[{date:'2026-08-25',grams:460},{date:'2026-08-30',grams:438}],
  status:'curing', rating:0, memo:'', doneDate:null, calReg:true
}])); location.reload();
```

```js
// 全部消す（テストの後始末）
localStorage.removeItem('pancetta-batches');
indexedDB.deleteDatabase('pancetta-photos');
location.reload();
```

## ファイル構成

変更するファイルは2つだけ。

| ファイル | 責務 | 変更内容 |
|---|---|---|
| `index.html` | アプリ本体（HTML/CSS/JS を内包） | 全タスクで変更 |
| `CLAUDE.md` | プロジェクトの説明書 | Task 1（データ構造）と Task 7（機能一覧・設計ルール） |

`index.html` 内の JavaScript は、既存のコメント区切り（`/* ---------- 見出し ---------- */`）の流儀に
合わせて次の順に並べる。

```
保存・読込                  … 既存 + migrate()（Task 1）
写真の保存箱（IndexedDB）    … 新規（Task 2）
豚さん（SVG）               … 既存
日付ユーティリティ           … 既存 + uid(), addLog()（Task 1）
カレンダー                  … 既存 + 注意書き（Task 5）
配合計算・新規登録           … 既存（Task 1 で logs 化）
描画                       … 既存 + logRows(), hydratePhotos()（Task 3/4/6）
お知らせカード              … 既存
記録（重量・メモ・写真）      … modal-weight を置き換え（Task 3/4）
編集                       … 新規（Task 5）
完成処理                    … 既存
共通（削除・タブ・トースト）   … 既存 + 汎用削除確認（Task 3）
```

---

## Task 1: `weights` を `logs` に移行する

見た目は一切変えない。データの形だけを入れ替え、既存の重量記録が失われないことを確かめる。

**Files:**
- Modify: `index.html`（データ構造コメント、`load()`、`addBatch()`、`render()`、`openWeight()`、`saveWeight()`）
- Modify: `CLAUDE.md`（「データ構造」の節）

**Interfaces:**
- Consumes: なし（最初のタスク）
- Produces:
  - `uid() -> string` — 衝突しない識別子
  - `addLog(b, log) -> void` — `b.logs` に日付昇順で挿入する
  - `migrate() -> boolean` — 旧形式を変換した場合 `true`
  - `batch.logs` の各要素 `{ id:string, date:string, grams:number|null, memo:string, photos:string[] }`

- [ ] **Step 1: 失敗する検証を書いて実行する**

ブラウザで `index.html` を開き、コンソールで実行する。

```js
localStorage.setItem('pancetta-batches', JSON.stringify([{
  id:'testold', name:'移行テスト', startDate:'2026-08-25', d1:5, d2:7,
  saltPct:3, sugarPct:1, meatWeight:460, spice:'黒コショウ',
  weights:[{date:'2026-08-25',grams:460},{date:'2026-08-30',grams:438}],
  status:'curing', rating:0, memo:'', doneDate:null, calReg:true
}])); location.reload();
```

再読み込み後、コンソールで実行する。

```js
(()=>{const b=batches[0];
 return [
  'logs がある:'      + (Array.isArray(b.logs) ? 'OK' : 'NG'),
  'weights が消えた:' + (b.weights===undefined ? 'OK' : 'NG'),
  '件数2:'            + (b.logs && b.logs.length===2 ? 'OK' : 'NG'),
  '1件目460g:'        + (b.logs && b.logs[0].grams===460 ? 'OK' : 'NG'),
  'memo空文字:'       + (b.logs && b.logs[0].memo==='' ? 'OK' : 'NG'),
  'photos空配列:'     + (b.logs && Array.isArray(b.logs[0].photos) ? 'OK' : 'NG'),
  'idがある:'         + (b.logs && b.logs[0].id ? 'OK' : 'NG')
 ].join('\n');})()
```

期待: **すべて NG**（`logs がある:NG` ほか。`b.logs` が無いため）

- [ ] **Step 2: `uid()` と `addLog()` を追加する**

`index.html` の日付ユーティリティの並び（`function todayStr(){...}` の直前）に足す。

```js
/* ---------- 識別子 ----------
   Date.now() だけだと、同じミリ秒に複数の写真を保存したときIDがぶつかる。
   乱数4桁を足して避ける。 */
function uid(){ return Date.now().toString(36) + Math.random().toString(36).slice(2,6); }

/* 記録を日付の昇順で差し込む。同じ日付のものは追加した順に並ぶ
   （Array.sort は同値の順序を保つため） */
function addLog(b, log){
  b.logs.push(log);
  b.logs.sort((x,y)=> x.date < y.date ? -1 : x.date > y.date ? 1 : 0);
}
```

- [ ] **Step 3: `migrate()` を追加し、`load()` から呼ぶ**

既存の `load()` を丸ごと次で置き換える。

```js
function load(){
  try{
    const r = localStorage.getItem(KEY);
    if(r) batches = JSON.parse(r);
  }catch(e){ /* 初回はデータが無いだけなので無視 */ }
  if(migrate()) save();
  render();
}

/* ---------- 旧データの読み替え ----------
   weights:[{date,grams}] を logs:[{id,date,grams,memo,photos}] にする。
   一度変換すれば logs が存在するので、次回以降は何もしない。 */
function migrate(){
  let changed = false;
  batches.forEach(b=>{
    if(!Array.isArray(b.logs)){
      b.logs = (b.weights||[]).map(w=>({
        id: uid(), date: w.date, grams: w.grams, memo:'', photos:[]
      }));
      changed = true;
    }
    if('weights' in b){ delete b.weights; changed = true; }
  });
  return changed;
}
```

- [ ] **Step 4: `addBatch()` が `logs` を作るようにする**

`addBatch()` 内の次の1行を、

```js
    weights: w>0 ? [{date, grams:w}] : [],
```

こう置き換える。

```js
    logs: w>0 ? [{id:uid(), date, grams:w, memo:'', photos:[]}] : [],
```

- [ ] **Step 5: 重量の読み手を `logs` に切り替える**

`render()` の熟成中カード内、重量減少率の計算を置き換える。既存:

```js
      let lossHtml='–';
      if(b.weights.length>=1 && b.weights[0].grams>0){
        const first=b.weights[0].grams, last=b.weights[b.weights.length-1].grams;
        lossHtml = first===last ? '0%' : '-'+(((first-last)/first)*100).toFixed(1)+'%';
      }
```

置き換え後（重量を書かなかった日＝`grams:null` を除いて計算する）:

```js
      let lossHtml='–';
      const gl = b.logs.filter(l=>l.grams>0);
      if(gl.length>=1){
        const first=gl[0].grams, last=gl[gl.length-1].grams;
        lossHtml = first===last ? '0%' : '-'+(((first-last)/first)*100).toFixed(1)+'%';
      }
```

`render()` のアーカイブ内も同様に置き換える。既存:

```js
      let loss='';
      if(b.weights.length>=2){
        const f=b.weights[0].grams,l=b.weights[b.weights.length-1].grams;
        loss='・重量 -'+(((f-l)/f)*100).toFixed(1)+'%';
      }
```

置き換え後:

```js
      let loss='';
      const gl = b.logs.filter(l=>l.grams>0);
      if(gl.length>=2){
        const f=gl[0].grams, l=gl[gl.length-1].grams;
        loss='・重量 -'+(((f-l)/f)*100).toFixed(1)+'%';
      }
```

- [ ] **Step 6: 既存の重量モーダルを `logs` で動かす**

`openWeight()` 内の履歴描画を置き換える。既存:

```js
  const first = b.weights.length? b.weights[0].grams : 0;
  log.innerHTML = b.weights.map(w=>{
```

置き換え後:

```js
  const gl = b.logs.filter(l=>l.grams>0);
  const first = gl.length ? gl[0].grams : 0;
  log.innerHTML = gl.map(w=>{
```

`saveWeight()` 内の追加処理を置き換える。既存:

```js
  b.weights.push({date:todayStr(), grams:g});
```

置き換え後:

```js
  addLog(b, {id:uid(), date:todayStr(), grams:g, memo:'', photos:[]});
```

- [ ] **Step 7: データ構造のコメントを直す**

`index.html` 冒頭のコメント内、

```
             meatWeight, spice, weights:[{date,grams}],
```

を次に置き換える。

```
             meatWeight, spice, logs:[{id,date,grams,memo,photos}],
```

- [ ] **Step 8: 検証を実行して通ることを確認する**

ブラウザを再読み込みし、Step 1 の後半の検証コードを再実行する。

期待: **すべて OK**

```
logs がある:OK
weights が消えた:OK
件数2:OK
1件目460g:OK
memo空文字:OK
photos空配列:OK
idがある:OK
```

続けて、画面が壊れていないことを確認する。

- 熟成中タブに「移行テスト」のカードが出ている
- カードの「重量変化」が **-4.8%** と出ている
- 「重量を記録」を開くと履歴に `2026/08/25 460g 基準` と `2026/08/30 438g -4.8%` が並ぶ
- 重量に `430` を入れて「記録する」→ 履歴が3行になり、カードが **-6.5%** になる

さらに、2度目の起動で無駄な書き込みが起きないことを確認する。

```js
(()=>{ const before = localStorage.getItem('pancetta-batches');
       return migrate()===false && localStorage.getItem('pancetta-batches')===before
            ? '2回目は変換しない:OK' : '2回目は変換しない:NG'; })()
```

期待: `2回目は変換しない:OK`

- [ ] **Step 9: `CLAUDE.md` のデータ構造を更新する**

「## データ構造」の節にある `weights` の行を置き換える。既存:

```
  weights,     // [{date:"YYYY-MM-DD", grams:数値}, ...] 重量記録の履歴
```

置き換え後:

```
  logs,        // 記録の履歴（下記）
```

同じ節の末尾、「全 batch を配列にして…」の段落の直前に足す。

````
```js
log = {
  id,      // 文字列（uid()）。記録1件の識別子
  date,    // "YYYY-MM-DD"
  grams,   // 数値 or null（重量を書かなかった日）
  memo,    // 文字列（空文字可）
  photos   // 写真IDの配列（0〜4件）。画像の実体は IndexedDB 側にある
}
```

`logs` は日付の昇順。旧 `weights` は起動時に自動で `logs` へ読み替えられる（`migrate()`）。
````

- [ ] **Step 10: コミット**

```bash
git add index.html CLAUDE.md
git commit -m "Replace the weights array with a log that can carry more than a number

A weight is only one of the things worth writing down on a given day, but
the record had no room for anything else. Widen each entry to hold a memo
and photo ids as well, and read the old shape once on startup so nothing
recorded so far is lost.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 2: 写真の保存箱（IndexedDB）と縮小処理

UIはまだ足さない。コンソールから呼べる関数だけを用意し、動くことを確かめる。

**Files:**
- Modify: `index.html`（`/* ---------- 保存・読込 ---------- */` の直後に新しい節を追加）

**Interfaces:**
- Consumes: `uid()`（Task 1）
- Produces:
  - `photoReady() -> Promise<boolean>` — 写真機能が使えるか
  - `photoPut(rec) -> Promise<string>` — `{id, batchId, blob, created}` を保存しIDを返す
  - `photoGet(id) -> Promise<{id,batchId,blob,created}|null>`
  - `photoDel(ids: string[]) -> Promise<void>`
  - `photoDelBatch(batchId) -> Promise<void>` — その仕込みの写真を全部消す
  - `shrink(file) -> Promise<Blob>` — 長辺1000px・JPEG品質75%に縮めた画像
  - `photoOK: boolean` — 起動時に判定した「写真機能が使えるか」

- [ ] **Step 1: 失敗する検証を書いて実行する**

コンソールで実行する。

```js
typeof photoPut === 'function' ? 'photoPut あり:OK' : 'photoPut あり:NG'
```

期待: `photoPut あり:NG`

- [ ] **Step 2: IndexedDB の読み書きを実装する**

`index.html` の `function save(){...}` の直後、`/* ---------- 豚さん（SVGイラスト） ---------- */` の直前に足す。

```js
/* ---------- 写真の保存箱（IndexedDB） ----------
   画像そのものは localStorage に入れない。localStorage の上限は約5MBで、
   あふれると仕込みデータの保存まで巻き添えで失敗するため、別の箱に分ける。
   IndexedDB は Blob をそのまま置けるので、文字列化による1.3倍の膨張も無い。 */
const PDB_NAME = 'pancetta-photos';
const PDB_STORE = 'photos';
let pdb = null;        // 開いた接続を使い回す
let photoOK = false;   // 写真機能を出してよいか（起動時に判定）

function photoOpen(){
  return new Promise(resolve=>{
    if(pdb) return resolve(pdb);
    if(!window.indexedDB) return resolve(null);
    let req;
    try{ req = indexedDB.open(PDB_NAME, 1); }
    catch(e){ return resolve(null); }
    req.onupgradeneeded = ()=>{
      const db = req.result;
      if(!db.objectStoreNames.contains(PDB_STORE)){
        const st = db.createObjectStore(PDB_STORE, {keyPath:'id'});
        st.createIndex('batchId', 'batchId', {unique:false});
      }
    };
    req.onsuccess = ()=>{ pdb = req.result; resolve(pdb); };
    req.onerror   = ()=> resolve(null);
    req.onblocked = ()=> resolve(null);
  });
}

function photoReady(){ return photoOpen().then(db=>!!db); }

function photoPut(rec){
  return photoOpen().then(db=>{
    if(!db) return Promise.reject(new Error('写真の保存箱を開けません'));
    return new Promise((res,rej)=>{
      const tx = db.transaction(PDB_STORE,'readwrite');
      tx.objectStore(PDB_STORE).put(rec);
      tx.oncomplete = ()=>res(rec.id);
      tx.onerror    = ()=>rej(tx.error || new Error('保存に失敗'));
      tx.onabort    = ()=>rej(tx.error || new Error('保存が中断'));
    });
  });
}

function photoGet(id){
  return photoOpen().then(db=>{
    if(!db) return null;
    return new Promise(res=>{
      const req = db.transaction(PDB_STORE,'readonly').objectStore(PDB_STORE).get(id);
      req.onsuccess = ()=>res(req.result || null);
      req.onerror   = ()=>res(null);
    });
  });
}

function photoDel(ids){
  if(!ids || !ids.length) return Promise.resolve();
  return photoOpen().then(db=>{
    if(!db) return;
    return new Promise(res=>{
      const tx = db.transaction(PDB_STORE,'readwrite');
      const st = tx.objectStore(PDB_STORE);
      ids.forEach(id=>st.delete(id));
      tx.oncomplete = ()=>res();
      tx.onerror    = ()=>res();   /* 消し損ねても操作は続行させる */
    });
  });
}

/* 仕込みを削除したとき、その仕込みの写真をまとめて消す。
   消し残すと二度と参照されない画像が容量を食い続けるため。 */
function photoDelBatch(batchId){
  return photoOpen().then(db=>{
    if(!db) return;
    return new Promise(res=>{
      const tx  = db.transaction(PDB_STORE,'readwrite');
      const req = tx.objectStore(PDB_STORE).index('batchId').openCursor(IDBKeyRange.only(batchId));
      req.onsuccess = ()=>{ const c = req.result; if(c){ c.delete(); c.continue(); } };
      tx.oncomplete = ()=>res();
      tx.onerror    = ()=>res();
    });
  });
}

/* 端末の写真は1枚4MB前後ある。そのままでは保存箱がすぐ膨らむので、
   長辺1000px・JPEG品質75%に縮めてから保存する（おおよそ100KB前後になる）。
   元が1000px以下なら引き伸ばさない。 */
function shrink(file){
  return new Promise((res,rej)=>{
    const url = URL.createObjectURL(file);
    const img = new Image();
    img.onload = ()=>{
      URL.revokeObjectURL(url);
      const MAX = 1000;
      const scale = Math.min(1, MAX/Math.max(img.width, img.height));
      const w = Math.max(1, Math.round(img.width  * scale));
      const h = Math.max(1, Math.round(img.height * scale));
      const cv = document.createElement('canvas');
      cv.width = w; cv.height = h;
      cv.getContext('2d').drawImage(img, 0, 0, w, h);
      cv.toBlob(b=> b ? res(b) : rej(new Error('画像を変換できません')), 'image/jpeg', 0.75);
    };
    img.onerror = ()=>{ URL.revokeObjectURL(url); rej(new Error('画像を読めません')); };
    img.src = url;
  });
}
```

- [ ] **Step 3: 起動時に `photoOK` を決める**

`index.html` 末尾の初期化（`load();` を呼んでいる箇所）を探し、その直後に足す。

```js
/* 写真機能が使えるかを最初に判定する。使えない端末では写真欄を出さない
   （重量とメモは今までどおり動く） */
photoReady().then(ok=>{ photoOK = ok; render(); });
```

- [ ] **Step 4: 検証を実行して通ることを確認する**

ブラウザを再読み込みし、コンソールで実行する。

```js
(async()=>{
  const out = [];
  out.push('使える:' + (await photoReady() ? 'OK':'NG'));

  // 1x1 の赤い画像を作って縮小 → 保存 → 取り出し
  const cv = document.createElement('canvas'); cv.width=2400; cv.height=1200;
  const cx = cv.getContext('2d'); cx.fillStyle='#B14A5A'; cx.fillRect(0,0,2400,1200);
  const src = await new Promise(r=>cv.toBlob(r,'image/png'));
  out.push('元サイズ:' + Math.round(src.size/1024) + 'KB');

  const small = await shrink(src);
  out.push('縮小後:' + Math.round(small.size/1024) + 'KB');
  const im = await createImageBitmap(small);
  out.push('長辺1000:' + (Math.max(im.width,im.height)===1000 ? 'OK':'NG(' + im.width + 'x' + im.height + ')'));
  out.push('JPEGになった:' + (small.type==='image/jpeg' ? 'OK':'NG'));

  const id = uid();
  await photoPut({id, batchId:'zzz', blob:small, created:Date.now()});
  const got = await photoGet(id);
  out.push('取り出せる:' + (got && got.blob.size===small.size ? 'OK':'NG'));

  await photoDelBatch('zzz');
  out.push('一括削除:' + (await photoGet(id) === null ? 'OK':'NG'));

  return out.join('\n');
})()
```

期待: 次のようになる（KBの数値は環境で多少ぶれてよい。**縮小後が元より小さいこと**が要点）

```
使える:OK
元サイズ:39KB
縮小後:31KB
長辺1000:OK
JPEGになった:OK
取り出せる:OK
一括削除:OK
```

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "Put photo bytes in IndexedDB, well away from the batch records

localStorage holds about 5MB, and a photo would eat that in a few dozen
shots. The bad part is not that photos stop saving - it is that the weight
records stop saving with them. Give the images their own store, shrink
them to a long edge of 1000px on the way in, and leave localStorage
holding nothing heavier than an id.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 3: 記録モーダル（重量・メモ・1件削除）

「重量を記録」を「記録する」に置き換える。写真欄はまだ出さない（Task 4 で足す）。

**Files:**
- Modify: `index.html`（CSS、`modal-weight` のHTML、カードのボタン、記録まわりのJS、削除確認の汎用化）

**Interfaces:**
- Consumes: `uid()`, `addLog()`, `esc()`, `toast()`, `daysElapsed()`, `todayStr()`
- Produces:
  - `openLog(id)` — 記録モーダルを開く
  - `saveLog()` — 入力を検証して1件追加する
  - `logRows(b, withDel) -> string` — 記録一覧の `<li>` 群。`withDel` が `true` のとき削除ボタンを付ける
  - `askDel(title, name, note, fn)` — 汎用の削除確認モーダルを開く
  - `draftPhotos: string[]` — 記録モーダルで追加待ちの写真ID（Task 4 で使う。ここでは常に空）

- [ ] **Step 1: 失敗する検証を書いて実行する**

コンソールで実行する。

```js
[ 'openLog:'  + (typeof openLog==='function' ? 'OK':'NG'),
  'logRows:'  + (typeof logRows==='function' ? 'OK':'NG'),
  'モーダル:' + (document.getElementById('modal-log') ? 'OK':'NG')
].join('\n')
```

期待: **すべて NG**

- [ ] **Step 2: CSS を足す**

`index.html` の `.weight-log` の3行（`.weight-log{...}` `.weight-log li{...}` `.weight-log .loss{...}`）を
丸ごと次で置き換える。

```css
  .log-list{font-size:13px;margin-top:14px;border-top:1px solid var(--cream-deep);}
  .log-list li{display:flex;gap:8px;align-items:flex-start;padding:8px 2px;
    border-bottom:1px dashed var(--cream-deep);list-style:none;}
  .log-list .d{color:var(--brown-soft);flex:0 0 42px;font-size:12px;padding-top:1px;}
  .log-list .t{flex:1;min-width:0;line-height:1.5;word-break:break-word;}
  .log-list .g{font-weight:700;}
  .log-list .loss{color:var(--rose);font-weight:700;margin-left:6px;}
  .log-list .memo{white-space:pre-wrap;margin-top:2px;}
  .log-list .x{border:none;background:none;color:var(--brown-soft);font-size:15px;
    cursor:pointer;padding:2px 4px;flex:0 0 auto;line-height:1;}
```

- [ ] **Step 3: モーダルのHTMLを置き換える**

`<div class="modal-bg" id="modal-weight">` から対応する `</div>` までの11行を、丸ごと次で置き換える。

```html
<div class="modal-bg" id="modal-log">
  <div class="modal">
    <button class="close-x" onclick="closeModal('modal-log')">✕</button>
    <h3>今日の記録</h3>
    <p style="font-size:12px;color:var(--brown-soft);" id="ml-name"></p>
    <div class="row2">
      <div><label>日付</label><input type="date" id="ml-date"></div>
      <div><label>重量（g）※任意</label>
        <input type="number" id="ml-grams" inputmode="decimal" placeholder="例：420"></div>
    </div>
    <label>メモ ※任意</label>
    <textarea id="ml-memo" rows="2" placeholder="例：ラップでくるんで冷蔵庫で熟成"></textarea>
    <div id="ml-photo-area"></div>
    <div class="btn-row"><button class="btn primary" onclick="saveLog()">記録する</button></div>
    <ul class="log-list" id="ml-list"></ul>
  </div>
</div>
```

- [ ] **Step 4: カードのボタンを差し替える**

`render()` の熟成中カード内、既存のボタン行:

```js
        <div class="btn-row">
          <button class="btn" onclick="openWeight('${b.id}')">重量を記録</button>
          <button class="btn primary" onclick="openDone('${b.id}')">完成にする</button>
          <button class="btn quiet" onclick="delBatch('${b.id}')">削除</button>
        </div>
```

を次で置き換える（「編集」は Task 5 で足すので、ここではまだ入れない）。

```js
        <div class="btn-row">
          <button class="btn" onclick="openLog('${b.id}')">記録する</button>
          <button class="btn primary" onclick="openDone('${b.id}')">完成にする</button>
          <button class="btn quiet" onclick="delBatch('${b.id}')">削除</button>
        </div>
```

- [ ] **Step 5: 記録モーダルのJSを書く**

`/* ---------- 重量記録 ---------- */` の見出しから `saveWeight()` の終わりまで（`openWeight` と
`saveWeight` の2関数）を、丸ごと次で置き換える。

```js
/* ---------- 記録（重量・メモ・写真） ----------
   重量とメモを別々のボタンにすると、書く内容によって押し分ける判断が毎回発生する。
   「記録する」1つに集め、書きたい欄だけ埋めてもらう。 */
let draftPhotos = [];    // 保存前に追加された写真ID（Task 4 で使う）

function openLog(id){
  modalTarget = id;
  const b = batches.find(x=>x.id===id);
  document.getElementById('ml-name').textContent =
    b.name + ' ・ ' + daysElapsed(b.startDate) + '日目';
  document.getElementById('ml-date').value  = todayStr();
  document.getElementById('ml-grams').value = '';
  document.getElementById('ml-memo').value  = '';
  draftPhotos = [];
  renderLogList();
  document.getElementById('modal-log').classList.add('show');
}

function renderLogList(){
  const b = batches.find(x=>x.id===modalTarget);
  document.getElementById('ml-list').innerHTML = logRows(b, true);
}

/* 記録一覧の中身。新しいものが上。
   減少率の基準は「重量を書いた記録のうち最も古いもの」。 */
function logRows(b, withDel){
  if(!b.logs.length) return '<li><span class="t">まだ記録がありません</span></li>';
  const gl = b.logs.filter(l=>l.grams>0);
  const base = gl.length ? gl[0].grams : 0;
  return b.logs.slice().reverse().map(l=>{
    let g = '';
    if(l.grams>0){
      const loss = (base>0 && l.grams!==base)
        ? '<span class="loss">-'+(((base-l.grams)/base)*100).toFixed(1)+'%</span>'
        : '<span class="loss">基準</span>';
      g = '<span class="g">'+l.grams+'g</span>'+loss;
    }
    const m  = l.memo ? '<div class="memo">'+esc(l.memo)+'</div>' : '';
    const x  = withDel ? '<button class="x" onclick="askDelLog(\''+l.id+'\')">✕</button>' : '';
    return '<li><span class="d">'+fmt(parseD(l.date))+'</span>'
         + '<span class="t">'+g+m+'</span>'+x+'</li>';
  }).join('');
}

function saveLog(){
  const b    = batches.find(x=>x.id===modalTarget);
  const date = document.getElementById('ml-date').value;
  const gRaw = document.getElementById('ml-grams').value.trim();
  const memo = document.getElementById('ml-memo').value.trim();
  if(!date){ toast('日付を入力してください'); return; }
  let grams = null;
  if(gRaw !== ''){
    const g = parseFloat(gRaw);
    if(!(g>0)){ toast('重量は正の数で入力してください'); return; }
    grams = g;
  }
  if(grams===null && !memo && draftPhotos.length===0){
    toast('重量・メモ・写真のどれかを入力してください'); return;
  }
  addLog(b, {id:uid(), date, grams, memo, photos:draftPhotos.slice()});
  draftPhotos = [];
  save(); render();
  closeModal('modal-log');
  toast('記録しました');
}

/* 記録1件を消す。写真も道連れにする（残しても参照する先が無いため） */
function askDelLog(logId){
  const b = batches.find(x=>x.id===modalTarget);
  const l = b.logs.find(x=>x.id===logId);
  if(!l) return;
  askDel('この記録を削除しますか？', fmt(parseD(l.date)) + ' の記録',
         '写真も一緒に削除されます。元に戻せません。', ()=>{
    photoDel(l.photos);
    b.logs = b.logs.filter(x=>x.id!==logId);
    save(); render(); renderLogList();
    toast('記録を削除しました');
  });
}
```

- [ ] **Step 6: 削除確認モーダルを汎用にする**

削除確認モーダルのHTMLを、見出しと注意文を差し替えられる形にする。既存:

```html
    <h3>この仕込みを削除しますか？</h3>
    <p style="font-size:12px;color:var(--brown-soft);margin-top:4px;" id="dl-name"></p>
    <p style="font-size:12px;color:var(--brown-soft);">削除すると記録は元に戻せません。</p>
```

置き換え後:

```html
    <h3 id="dl-title">この仕込みを削除しますか？</h3>
    <p style="font-size:12px;color:var(--brown-soft);margin-top:4px;" id="dl-name"></p>
    <p style="font-size:12px;color:var(--brown-soft);" id="dl-note">削除すると記録は元に戻せません。</p>
```

JS側、既存の `delBatch()` と `confirmDel()`（`let delTarget = null;` を含む）を丸ごと置き換える。

```js
/* 削除確認モーダル。仕込みの削除と記録の削除で使い回す */
let delAction = null;
function askDel(title, name, note, fn){
  document.getElementById('dl-title').textContent = title;
  document.getElementById('dl-name').textContent  = name;
  document.getElementById('dl-note').textContent  = note;
  delAction = fn;
  document.getElementById('modal-del').classList.add('show');
}
function confirmDel(){
  const fn = delAction;
  delAction = null;
  closeModal('modal-del');
  if(fn) fn();
}
function delBatch(id){
  const b = batches.find(x=>x.id===id);
  if(!b) return;
  askDel('この仕込みを削除しますか？', b.name, '写真を含め、記録は元に戻せません。', ()=>{
    photoDelBatch(id);
    batches = batches.filter(x=>x.id!==id);
    save(); render();
    toast('削除しました');
  });
}
```

- [ ] **Step 7: 検証を実行して通ることを確認する**

ブラウザを再読み込みし、Step 1 の検証コードを再実行する。

期待: **すべて OK**

続けて、入力の検証をコンソールで確かめる。

```js
(()=>{ openLog(batches[0].id);
  const r = [];
  // 全部空 → 保存されない
  const n0 = batches[0].logs.length;
  saveLog();
  r.push('空では保存しない:' + (batches[0].logs.length===n0 ? 'OK':'NG'));
  // 重量に0 → 保存されない
  document.getElementById('ml-grams').value = '0';
  saveLog();
  r.push('0gは弾く:' + (batches[0].logs.length===n0 ? 'OK':'NG'));
  // メモだけ → 保存される
  document.getElementById('ml-grams').value = '';
  document.getElementById('ml-memo').value  = 'ラップでくるんで冷蔵庫で熟成';
  saveLog();
  const last = batches[0].logs[batches[0].logs.length-1];
  r.push('メモだけで保存:' + (batches[0].logs.length===n0+1 ? 'OK':'NG'));
  r.push('重量はnull:'     + (last.grams===null ? 'OK':'NG'));
  return r.join('\n');
})()
```

期待:

```
空では保存しない:OK
0gは弾く:OK
メモだけで保存:OK
重量はnull:OK
```

日付順に差し込まれることを確かめる。

```js
(()=>{ openLog(batches[0].id);
  document.getElementById('ml-date').value = '2026-08-27';
  document.getElementById('ml-memo').value = '間に入る記録';
  saveLog();
  const d = batches[0].logs.map(l=>l.date);
  return '日付昇順:' + (d.join(',')===d.slice().sort().join(',') ? 'OK':'NG(' + d.join(',') + ')');
})()
```

期待: `日付昇順:OK`

画面でも確認する。

- 熟成中カードのボタンが「記録する / 完成にする / 削除」になっている
- 「記録する」を開くと、上部に `移行テスト ・ N日目`、履歴が新しい順に並ぶ
- 重量を書いた行だけ `438g -4.8%` のように出て、メモだけの行には重量が出ない
- 履歴の「✕」→「この記録を削除しますか？」が出て、「削除する」で消え、履歴が1行減る
- 「削除」（カード）→「この仕込みを削除しますか？」の見出しに戻っている

- [ ] **Step 8: コミット**

```bash
git add index.html
git commit -m "Fold the weight modal into one record that takes a note too

Two buttons would have meant deciding which one to press every time, and
opening the sheet twice on a day worth writing about. One sheet takes the
date, the weight, and the note, and any of them may be left blank. The
history is now a single line of time instead of two.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 4: 写真の追加・表示・削除

**Files:**
- Modify: `index.html`（CSS、写真ビューアのHTML、記録モーダルの写真欄、`logRows()` にサムネイル）

**Interfaces:**
- Consumes: `photoOK`, `photoPut()`, `photoGet()`, `photoDel()`, `shrink()`, `uid()`, `logRows()`, `draftPhotos`
- Produces:
  - `renderPhotoArea()` — 記録モーダルの写真欄を描き直す
  - `pickPhotos(input)` — 選ばれたファイルを縮小して保存する
  - `hydratePhotos()` — `data-photo` を持つ `<img>` に画像を流し込む
  - `viewPhoto(photoId)` / `delPhoto()` — 全画面表示と1枚削除
  - `photoThumbs(log) -> string` — サムネイルのHTML

- [ ] **Step 1: 失敗する検証を書いて実行する**

コンソールで実行する。

```js
[ 'pickPhotos:'  + (typeof pickPhotos==='function' ? 'OK':'NG'),
  'ビューア:'     + (document.getElementById('modal-photo') ? 'OK':'NG'),
  '写真欄:'       + (document.getElementById('ml-photo-area') &&
                     document.getElementById('ml-photo-area').innerHTML.trim() ? 'OK':'NG')
].join('\n')
```

（3つ目は先に `openLog(batches[0].id)` を実行してから確認する）

期待: **すべて NG**

- [ ] **Step 2: CSS を足す**

`.log-list .x{...}` の直後に足す。

```css
  /* 写真 */
  .thumbs{display:flex;gap:6px;margin-top:6px;flex-wrap:wrap;}
  .thumb{width:52px;height:52px;border-radius:8px;object-fit:cover;
    border:1px solid var(--cream-deep);background:var(--rose-soft);cursor:pointer;}
  .log-list .thumb{width:44px;height:44px;}
  .thumb-add{width:52px;height:52px;border-radius:8px;border:1.5px dashed var(--cream-deep);
    background:#fff;color:var(--brown-soft);font:inherit;font-size:11px;cursor:pointer;
    display:flex;align-items:center;justify-content:center;flex-direction:column;line-height:1.3;}
  .photo-view{align-items:center;background:rgba(59,42,34,.88);}
  .photo-stage{width:100%;max-width:480px;padding:16px;}
  .photo-stage img{width:100%;border-radius:12px;background:var(--cream);display:block;}
  .photo-stage .btn{background:var(--cream);}
```

- [ ] **Step 3: 写真ビューアのHTMLを足す**

`<div class="toast" id="toast"></div>` の直前に足す。

```html
<!-- 写真の全画面表示 -->
<div class="modal-bg photo-view" id="modal-photo">
  <div class="photo-stage">
    <img id="pv-img" alt="記録した写真">
    <div class="btn-row">
      <button class="btn" onclick="closeModal('modal-photo')">閉じる</button>
      <button class="btn quiet" onclick="delPhoto()">この写真を削除</button>
    </div>
  </div>
</div>
```

- [ ] **Step 4: 写真のJSを書く**

`saveLog()` の直後に足す。

```js
/* ---------- 写真 ----------
   IndexedDB から取り出した Blob は、表示のたびに一時URLを作る必要がある。
   作りっぱなしだとメモリに残るので、描き直す前に必ず破棄する。 */
let objUrls = [];
let viewingPhoto = null;   // 全画面表示中の写真ID

function freeUrls(){ objUrls.forEach(u=>URL.revokeObjectURL(u)); objUrls = []; }

/* 一覧・履歴に置くサムネイルの器。画像は hydratePhotos() が後から流し込む */
function photoThumbs(l){
  if(!l.photos || !l.photos.length) return '';
  return '<div class="thumbs">'
    + l.photos.map(p=>'<img class="thumb" data-photo="'+p+'" alt="記録した写真" '
        + 'onclick="viewPhoto(\''+p+'\')">').join('')
    + '</div>';
}

/* data-photo を持つ img に、IndexedDB の画像を流し込む。
   innerHTML を書き換えた後に必ず呼ぶ */
function hydratePhotos(){
  document.querySelectorAll('img[data-photo]:not([src])').forEach(el=>{
    photoGet(el.dataset.photo).then(rec=>{
      if(!rec) return;
      const u = URL.createObjectURL(rec.blob);
      objUrls.push(u);
      el.src = u;
    });
  });
}

/* 記録モーダルの写真欄。写真機能が使えない端末では何も出さない */
function renderPhotoArea(){
  const box = document.getElementById('ml-photo-area');
  if(!photoOK){ box.innerHTML = ''; return; }
  box.innerHTML =
    '<label>写真 ※任意</label>'
    + '<div class="thumbs" id="ml-thumbs">'
    +   draftPhotos.map(p=>'<img class="thumb" data-photo="'+p+'" alt="追加した写真" '
          + 'onclick="viewPhoto(\''+p+'\')">').join('')
    +   (draftPhotos.length < 4
          ? '<button class="thumb-add" onclick="document.getElementById(\'ml-file\').click()">＋<br>追加</button>'
          : '')
    + '</div>'
    + '<input type="file" id="ml-file" accept="image/*" multiple hidden onchange="pickPhotos(this)">';
  hydratePhotos();
}

function pickPhotos(input){
  const files = Array.from(input.files || []);
  input.value = '';                       /* 同じ写真を続けて選べるようにする */
  if(!files.length) return;
  const room = 4 - draftPhotos.length;
  if(files.length > room) toast('写真は1件につき4枚までです');
  const use = files.slice(0, room);
  const batchId = modalTarget;
  Promise.all(use.map(f=>
    shrink(f).then(blob=>{
      const id = uid();
      return photoPut({id, batchId, blob, created:Date.now()}).then(()=>id);
    })
  )).then(ids=>{
    draftPhotos = draftPhotos.concat(ids);
    renderPhotoArea();
  }).catch(()=>{
    toast('写真を保存できませんでした');
    renderPhotoArea();
  });
}

function viewPhoto(photoId){
  viewingPhoto = photoId;
  const img = document.getElementById('pv-img');
  img.removeAttribute('src');
  photoGet(photoId).then(rec=>{
    if(!rec){ toast('写真が見つかりません'); return; }
    const u = URL.createObjectURL(rec.blob);
    objUrls.push(u);
    img.src = u;
  });
  document.getElementById('modal-photo').classList.add('show');
}

/* 全画面表示からの1枚削除。誤タップしにくい位置なので確認は挟まない */
function delPhoto(){
  const id = viewingPhoto;
  if(!id) return;
  photoDel([id]);
  batches.forEach(b=> b.logs.forEach(l=>{
    if(l.photos) l.photos = l.photos.filter(p=>p!==id);
  }));
  draftPhotos = draftPhotos.filter(p=>p!==id);
  viewingPhoto = null;
  save(); render();
  closeModal('modal-photo');
  if(document.getElementById('modal-log').classList.contains('show')){
    renderLogList(); renderPhotoArea();
  }
  toast('写真を削除しました');
}
```

- [ ] **Step 5: 既存の描画に写真をつなぐ**

`logRows()` の中で、メモの次にサムネイルを差し込む。既存:

```js
    const m  = l.memo ? '<div class="memo">'+esc(l.memo)+'</div>' : '';
    const x  = withDel ? '<button class="x" onclick="askDelLog(\''+l.id+'\')">✕</button>' : '';
    return '<li><span class="d">'+fmt(parseD(l.date))+'</span>'
         + '<span class="t">'+g+m+'</span>'+x+'</li>';
```

置き換え後:

```js
    const m  = l.memo ? '<div class="memo">'+esc(l.memo)+'</div>' : '';
    const ph = photoThumbs(l);
    const x  = withDel ? '<button class="x" onclick="askDelLog(\''+l.id+'\')">✕</button>' : '';
    return '<li><span class="d">'+fmt(parseD(l.date))+'</span>'
         + '<span class="t">'+g+m+ph+'</span>'+x+'</li>';
```

`renderLogList()` の末尾で画像を流し込む。既存:

```js
function renderLogList(){
  const b = batches.find(x=>x.id===modalTarget);
  document.getElementById('ml-list').innerHTML = logRows(b, true);
}
```

置き換え後:

```js
function renderLogList(){
  const b = batches.find(x=>x.id===modalTarget);
  document.getElementById('ml-list').innerHTML = logRows(b, true);
  hydratePhotos();
}
```

`openLog()` の `renderLogList();` の直前に1行足す。

```js
  freeUrls();
  renderPhotoArea();
  renderLogList();
```

`render()` の先頭（`renderNotice();` の直前）に1行足す。描き直しで捨てられる一時URLを解放するため。

```js
function render(){
  freeUrls();
  renderNotice();
```

- [ ] **Step 6: 検証を実行して通ることを確認する**

ブラウザを再読み込みし、コンソールで実行する。

```js
(async()=>{
  const r = [];
  openLog(batches[0].id);
  r.push('写真欄が出る:' + (document.getElementById('ml-thumbs') ? 'OK':'NG'));

  // 画像ファイルを作って pickPhotos に流し込む（ファイル選択の代わり）
  const mk = async (color)=>{
    const cv=document.createElement('canvas'); cv.width=1600; cv.height=900;
    const cx=cv.getContext('2d'); cx.fillStyle=color; cx.fillRect(0,0,1600,900);
    const b=await new Promise(r2=>cv.toBlob(r2,'image/png'));
    return new File([b], 'test.png', {type:'image/png'});
  };
  const five = [];
  for(const c of ['#B14A5A','#3B2A22','#FAF5EE','#E7CBCF','#F1E8DC']) five.push(await mk(c));

  await new Promise(res=>{ pickPhotos({files:five, value:''}); setTimeout(res, 900); });
  r.push('4枚で打ち止め:' + (draftPhotos.length===4 ? 'OK':'NG(' + draftPhotos.length + ')'));
  r.push('＋ボタンが消える:' + (!document.querySelector('.thumb-add') ? 'OK':'NG'));

  document.getElementById('ml-memo').value = '写真つきの記録';
  saveLog();
  const last = batches[0].logs[batches[0].logs.length-1];
  r.push('記録に4枚ついた:' + (last.photos.length===4 ? 'OK':'NG'));
  r.push('draftが空に戻る:' + (draftPhotos.length===0 ? 'OK':'NG'));
  r.push('実体がある:' + (await photoGet(last.photos[0]) ? 'OK':'NG'));

  // 1枚消す
  viewingPhoto = last.photos[0];
  const gone = last.photos[0];
  delPhoto();
  await new Promise(res=>setTimeout(res,300));
  const after = batches[0].logs[batches[0].logs.length-1];
  r.push('記録から外れた:' + (after.photos.length===3 ? 'OK':'NG'));
  r.push('実体も消えた:'   + (await photoGet(gone)===null ? 'OK':'NG'));
  return r.join('\n');
})()
```

期待:

```
写真欄が出る:OK
4枚で打ち止め:OK
＋ボタンが消える:OK
記録に4枚ついた:OK
draftが空に戻る:OK
実体がある:OK
記録から外れた:OK
実体も消えた:OK
```

仕込みを削除したとき写真も消えることを確かめる。

```js
(async()=>{
  const b = batches[0];
  const ids = b.logs.flatMap(l=>l.photos||[]);
  await photoDelBatch(b.id);
  const left = [];
  for(const id of ids){ if(await photoGet(id)) left.push(id); }
  return '仕込み削除で写真も消える:' + (left.length===0 ? 'OK':'NG(残り' + left.length + ')');
})()
```

期待: `仕込み削除で写真も消える:OK`

画面でも確認する。

- 「記録する」に「写真 ※任意」と「＋ 追加」が出ている
- 「＋ 追加」で写真を選ぶとサムネイルが並ぶ
- サムネイルをタップすると全画面で表示される（背景は濃茶の半透明）
- 全画面の「この写真を削除」で消え、サムネイルが1枚減る
- 記録を保存すると、履歴の行にサムネイルが並ぶ

- [ ] **Step 7: コミット**

```bash
git add index.html
git commit -m "Let a record carry photos, up to four of them

A note says the meat looked dry; a photo shows how dry. Shrink on the way
in, keep the bytes in IndexedDB and only the ids in the record, and hand
back every object URL before redrawing so the tab does not leak a
morning's worth of images.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 5: 編集モーダルとカレンダー連動

**Files:**
- Modify: `index.html`（CSS 1行、編集モーダルのHTML、カードのボタン、編集のJS、カレンダーモーダルの注意書き）

**Interfaces:**
- Consumes: `batches`, `save()`, `render()`, `toast()`, `esc()`, `addLog()`
- Produces:
  - `openEdit(id)` — 編集モーダルを開く
  - `saveEdit()` — 検証して保存し、必要なら `calReg` を戻す
  - `editBaseNote()` — 仕込み重量の注記を出し分ける

- [ ] **Step 1: 失敗する検証を書いて実行する**

コンソールで実行する。

```js
[ 'openEdit:'   + (typeof openEdit==='function' ? 'OK':'NG'),
  'モーダル:'    + (document.getElementById('modal-edit') ? 'OK':'NG')
].join('\n')
```

期待: **すべて NG**

- [ ] **Step 2: CSS を足す**

`.log-list .x{...}` の直後（写真のCSSの手前）に足す。

```css
  .edit-note{font-size:11.5px;color:var(--rose);background:var(--rose-soft);
    border-radius:8px;padding:7px 10px;margin-top:6px;line-height:1.5;}
```

- [ ] **Step 3: 編集モーダルのHTMLを足す**

`<div class="modal-bg" id="modal-log">` のブロックの直前に足す。

```html
<!-- 編集モーダル -->
<div class="modal-bg" id="modal-edit">
  <div class="modal">
    <button class="close-x" onclick="closeModal('modal-edit')">✕</button>
    <h3>仕込みを編集</h3>
    <p style="font-size:12px;color:var(--brown-soft);" id="me-name"></p>
    <label>仕込み名</label>
    <input type="text" id="me-title">
    <label>仕込み開始日</label>
    <input type="date" id="me-date">
    <div class="row2">
      <div><label>塩漬け（日）</label><input type="number" id="me-d1" inputmode="numeric"></div>
      <div><label>スパイス熟成（日）</label><input type="number" id="me-d2" inputmode="numeric"></div>
    </div>
    <label>スパイスメモ</label>
    <input type="text" id="me-spice" placeholder="例：黒コショウ＋チリパウダー">
    <div class="row2">
      <div><label>塩（%）</label><input type="number" id="me-salt" inputmode="decimal" step="0.1"></div>
      <div><label>砂糖（%）</label><input type="number" id="me-sugar" inputmode="decimal" step="0.1"></div>
    </div>
    <label>仕込み重量（g）</label>
    <input type="number" id="me-weight" inputmode="decimal" oninput="editBaseNote()">
    <p class="edit-note" id="me-note" hidden></p>
    <div class="btn-row">
      <button class="btn" onclick="closeModal('modal-edit')">やめる</button>
      <button class="btn primary" onclick="saveEdit()">保存する</button>
    </div>
  </div>
</div>
```

- [ ] **Step 4: カードに「編集」ボタンを足す**

`render()` の熟成中カード内、ボタン行を次で置き換える。既存:

```js
        <div class="btn-row">
          <button class="btn" onclick="openLog('${b.id}')">記録する</button>
          <button class="btn primary" onclick="openDone('${b.id}')">完成にする</button>
          <button class="btn quiet" onclick="delBatch('${b.id}')">削除</button>
        </div>
```

置き換え後:

```js
        <div class="btn-row">
          <button class="btn" onclick="openLog('${b.id}')">記録する</button>
          <button class="btn primary" onclick="openDone('${b.id}')">完成にする</button>
        </div>
        <div class="btn-row tight">
          <button class="btn" onclick="openEdit('${b.id}')">編集</button>
          <button class="btn quiet" onclick="delBatch('${b.id}')">削除</button>
        </div>
```

- [ ] **Step 5: 編集のJSを書く**

`/* ---------- 完成処理 ---------- */` の見出しの直前に足す。

```js
/* ---------- 編集 ----------
   仕込み後に直せるものが何も無いと、日数や開始日の打ち間違いがずっと残る。
   完成済み（アーカイブ）は読み取り専用のままにする。 */
function openEdit(id){
  modalTarget = id;
  const b = batches.find(x=>x.id===id);
  document.getElementById('me-name').textContent   = b.name;
  document.getElementById('me-title').value  = b.name;
  document.getElementById('me-date').value   = b.startDate;
  document.getElementById('me-d1').value     = b.d1;
  document.getElementById('me-d2').value     = b.d2;
  document.getElementById('me-spice').value  = b.spice || '';
  document.getElementById('me-salt').value   = b.saltPct;
  document.getElementById('me-sugar').value  = b.sugarPct;
  document.getElementById('me-weight').value = b.meatWeight || '';
  editBaseNote();
  document.getElementById('modal-edit').classList.add('show');
}

/* 仕込み重量の1件目（アプリが自動で作った基準値）を探す。
   手で書き換えられた記録には触らないので、日付と値の両方が一致するものだけ返す */
function baseLog(b){
  return b.logs.find(l=>l.date===b.startDate && l.grams===b.meatWeight) || null;
}

/* 仕込み重量を変えると重量記録の1件目と食い違う。
   確認の小窓を増やす代わりに、変えている最中に注記を出す */
function editBaseNote(){
  const b    = batches.find(x=>x.id===modalTarget);
  const note = document.getElementById('me-note');
  const base = b ? baseLog(b) : null;
  const w    = parseFloat(document.getElementById('me-weight').value);
  if(base && w>0 && w!==b.meatWeight){
    note.textContent = '※ 重量記録の1件目（'+base.grams+'g・基準）も同じ値に直します';
    note.hidden = false;
  }else{
    note.hidden = true;
  }
}

function saveEdit(){
  const b = batches.find(x=>x.id===modalTarget);
  const name  = document.getElementById('me-title').value.trim() || 'パンチェッタ';
  const date  = document.getElementById('me-date').value;
  const d1    = parseInt(document.getElementById('me-d1').value, 10);
  const d2    = parseInt(document.getElementById('me-d2').value, 10);
  const spice = document.getElementById('me-spice').value.trim();
  const salt  = parseFloat(document.getElementById('me-salt').value);
  const sugar = parseFloat(document.getElementById('me-sugar').value);
  const wRaw  = document.getElementById('me-weight').value.trim();
  const w     = wRaw==='' ? 0 : parseFloat(wRaw);

  if(!date){ toast('仕込み開始日を入力してください'); return; }
  if(!(d1>=1) || !(d2>=1)){ toast('日数は1日以上で入力してください'); return; }
  if(!(salt>=0) || !(sugar>=0) || !(w>=0)){ toast('数値は0以上で入力してください'); return; }

  /* 日程が動いたか。カレンダーに登録済みの予定がズレるので後で知らせる */
  const moved = (b.startDate!==date || b.d1!==d1 || b.d2!==d2);
  const wasReg = b.calReg;

  /* 仕込み重量を変えたとき、自動で作った基準値も合わせる（先に探しておく） */
  const base = baseLog(b);

  b.name = name; b.startDate = date; b.d1 = d1; b.d2 = d2;
  b.spice = spice; b.saltPct = salt; b.sugarPct = sugar;

  if(base && w>0 && w!==b.meatWeight) base.grams = w;
  b.meatWeight = w;

  if(moved && wasReg) b.calReg = false;

  save(); render();
  closeModal('modal-edit');
  toast(moved && wasReg
    ? '日程が変わりました。カレンダーの予定を登録し直してください'
    : '変更を保存しました');
}
```

- [ ] **Step 6: カレンダーモーダルに注意書きを足す**

アプリから登録済みの予定を消す手段が無いことを伝える。`modal-cal` 内の
`※通知が鳴る時刻は…` の `<p>` の直前に足す。

```html
    <p style="font-size:11.5px;color:var(--rose);margin-top:12px;line-height:1.55;">
      ※前に登録した予定は、カレンダーアプリで手で削除してください。アプリから消すことはできません。</p>
```

- [ ] **Step 7: 検証を実行して通ることを確認する**

ブラウザを再読み込みし、Step 1 の検証コードを再実行する。

期待: **すべて OK**

続けてコンソールで実行する。

```js
(()=>{
  const r = [];
  const b = batches[0];
  b.calReg = true; b.d1 = 5; save();

  openEdit(b.id);
  // 日数を0にすると弾かれる
  document.getElementById('me-d1').value = '0';
  saveEdit();
  r.push('0日は弾く:' + (b.d1===5 ? 'OK':'NG'));

  // 塩漬けを 5 → 7
  document.getElementById('me-d1').value = '7';
  saveEdit();
  r.push('日数が変わる:'   + (b.d1===7 ? 'OK':'NG'));
  r.push('calRegが戻る:'   + (b.calReg===false ? 'OK':'NG'));

  // 日程を変えずに名前だけ変えても calReg は戻らない
  b.calReg = true; save();
  openEdit(b.id);
  document.getElementById('me-title').value = '名前だけ変更';
  saveEdit();
  r.push('名前だけなら維持:' + (b.calReg===true && b.name==='名前だけ変更' ? 'OK':'NG'));
  return r.join('\n');
})()
```

期待:

```
0日は弾く:OK
日数が変わる:OK
calRegが戻る:OK
名前だけなら維持:OK
```

仕込み重量と基準値の連動を確かめる。

```js
(()=>{
  const r = [];
  const b = batches[0];
  // 基準値がある場合：一緒に直る
  b.meatWeight = 460;
  b.startDate  = '2026-08-25';
  b.logs = [{id:uid(), date:'2026-08-25', grams:460, memo:'', photos:[]},
            {id:uid(), date:'2026-08-30', grams:438, memo:'', photos:[]}];
  save();
  openEdit(b.id);
  document.getElementById('me-weight').value = '470';
  editBaseNote();
  r.push('注記が出る:' + (!document.getElementById('me-note').hidden ? 'OK':'NG'));
  saveEdit();
  r.push('基準値も直る:'   + (b.logs[0].grams===470 ? 'OK':'NG'));
  r.push('2件目は無傷:'    + (b.logs[1].grams===438 ? 'OK':'NG'));

  // 手で書き換えた1件目（値が meatWeight と違う）には触らない
  b.meatWeight = 470;
  b.logs[0].grams = 455;
  save();
  openEdit(b.id);
  document.getElementById('me-weight').value = '480';
  editBaseNote();
  r.push('注記が出ない:' + (document.getElementById('me-note').hidden ? 'OK':'NG'));
  saveEdit();
  r.push('手入力は守られる:' + (b.logs[0].grams===455 ? 'OK':'NG'));
  return r.join('\n');
})()
```

期待:

```
注記が出る:OK
基準値も直る:OK
2件目は無傷:OK
注記が出ない:OK
手入力は守られる:OK
```

画面でも確認する。

- カードのボタンが2行になり、2行目が「編集 / 削除」
- 「編集」を開くと現在の値が入っている
- 仕込み重量を変えると、その下にローズ色の注記が出る
- 塩漬け日数を変えて保存すると、通知ボタンが「🔔 通知をカレンダーに登録」の見た目に戻り、
  画面下に「日程が変わりました。カレンダーの予定を登録し直してください」と出る
- リングの「N日目 / 全日数」と完成予定日が新しい日数で計算し直されている
- 通知モーダルにローズ色の「※前に登録した予定は…」が出ている

- [ ] **Step 8: コミット**

```bash
git add index.html
git commit -m "Let a batch be corrected after it is started

A mistyped start date used to skew the countdown for the whole cure with
no way back. Editing changes the schedule, so a calendar reminder already
saved is now pointing at the wrong morning: drop the registered flag and
say so, since the app can create those events but never delete them.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 6: アーカイブに日誌を表示する

**Files:**
- Modify: `index.html`（CSS、`render()` のアーカイブ部分）
- Modify: `CLAUDE.md`（「主要な機能」と「設計ルール」）

**Interfaces:**
- Consumes: `logRows()`, `hydratePhotos()`
- Produces: なし（最後のタスク）

- [ ] **Step 1: 失敗する検証を書いて実行する**

完成済みの仕込みを1件作ってからコンソールで実行する。

```js
(()=>{ const b = batches[0];
  b.status='done'; b.rating=4; b.doneDate='2026-09-06'; b.memo='塩3%はちょうど良い';
  save(); render(); showTab('arch');
  return 'アーカイブに日誌:' + (document.querySelector('.arch-log') ? 'OK':'NG');
})()
```

期待: `アーカイブに日誌:NG`

- [ ] **Step 2: CSS を足す**

`.arch-memo{...}` の直後に足す。

```css
  .arch-log{margin-top:10px;}
  .arch-log summary{font-size:12.5px;font-weight:700;color:var(--rose);cursor:pointer;
    list-style:none;padding:6px 0;}
  .arch-log summary::-webkit-details-marker{display:none;}
  .arch-log summary::before{content:'▸ ';}
  .arch-log[open] summary::before{content:'▾ ';}
  .arch-log .log-list{margin-top:0;}
```

- [ ] **Step 3: アーカイブカードに折りたたみを足す**

`render()` のアーカイブ内、既存:

```js
        ${b.memo? '<div class="arch-memo">'+esc(b.memo)+'</div>':''}
        <div class="btn-row"><button class="btn quiet" onclick="delBatch('${b.id}')">削除</button></div>
```

置き換え後（既定は閉じた状態。開くと記録モーダルと同じ体裁で、削除ボタンは出さない）:

```js
        ${b.memo? '<div class="arch-memo">'+esc(b.memo)+'</div>':''}
        ${b.logs.length? '<details class="arch-log"><summary>記録を見る（'+b.logs.length+'件）</summary>'
          + '<ul class="log-list">'+logRows(b,false)+'</ul></details>' : ''}
        <div class="btn-row"><button class="btn quiet" onclick="delBatch('${b.id}')">削除</button></div>
```

`render()` の末尾（アーカイブの `innerHTML` を入れ終えた後、関数を閉じる `}` の直前）に1行足す。

```js
  hydratePhotos();
}
```

- [ ] **Step 4: 検証を実行して通ることを確認する**

ブラウザを再読み込みし、アーカイブタブを開いてコンソールで実行する。

```js
(()=>{ const d = document.querySelector('.arch-log');
  const r = [];
  r.push('折りたたみがある:' + (d ? 'OK':'NG'));
  r.push('既定は閉じている:' + (d && !d.open ? 'OK':'NG'));
  r.push('削除ボタンなし:'   + (d && !d.querySelector('.log-list .x') ? 'OK':'NG'));
  if(d){ d.open = true; }
  return r.join('\n');
})()
```

期待:

```
折りたたみがある:OK
既定は閉じている:OK
削除ボタンなし:OK
```

画面でも確認する。

- アーカイブのカードに「記録を見る（N件）」があり、既定では閉じている
- 開くと日付・重量・減少率・メモ・写真サムネイルが並ぶ
- サムネイルをタップすると全画面で見られる
- 記録行に「✕」が出ていない

- [ ] **Step 5: `CLAUDE.md` を更新する**

「## 主要な機能」の3番と4番を置き換える。既存:

```
3. **重量記録** — 日付ごとの重量を追加し、初回比の減少率(%)を表示
4. **完成アーカイブ** — ★5段階評価と味メモを残し、次回の配合改善に使う
```

置き換え後:

```
3. **記録** — 日付ごとに重量・メモ・写真（最大4枚）を1つの小窓で追加する。
   重量には初回比の減少率(%)が付く。書きたい欄だけ埋めればよく、3つとも空なら保存しない。
   記録1件は後から削除できる（写真も一緒に消える）
4. **編集** — 仕込み名・開始日・塩漬け日数・スパイス熟成日数・スパイスメモ・塩%・砂糖%・
   仕込み重量を後から直せる。完成済み（アーカイブ）は読み取り専用
5. **完成アーカイブ** — ★5段階評価と味メモを残し、次回の配合改善に使う。
   日誌は「記録を見る」で折りたたんで表示する
```

続く「5. **通知**」を「6. **通知**」に直し、その説明の末尾に足す。

```
   開始日・日数を編集して日程が動いたときは、登録済みフラグを自動で解除して登録し直しを促す
   （アプリからカレンダーの予定を消すことはできないため、古い予定は手で削除してもらう）
```

「## 設計ルール（変更時も必ず守ること）」の末尾に足す。

```
- **写真の実体は IndexedDB に置く**（DB名 `pancetta-photos`）。localStorage の上限は約5MBで、
  あふれると仕込みデータの保存まで巻き添えで失敗する。`batch.logs` が持つのは写真IDだけ。
  保存前に長辺1000px・JPEG品質75%へ縮小する
- **IndexedDB から取り出した画像は一時URL（`URL.createObjectURL`）で表示する**。
  描き直す前に必ず `freeUrls()` で破棄すること
```

「## 今後の改修候補」の1行目を置き換える。既存:

```
- 記録のバックアップ（JSON書き出し / 読み込み）※ブラウザのデータ削除で消えるリスク対策
```

置き換え後:

```
- 記録のバックアップ（JSON書き出し / 読み込み）※ブラウザのデータ削除で消えるリスク対策。
  写真は IndexedDB にあるため、書き出しには画像を含める仕掛けが要る
```

- [ ] **Step 6: 全体をひととおり操作して確認する**

保存データを消してまっさらな状態から確かめる。

```js
localStorage.removeItem('pancetta-batches');
indexedDB.deleteDatabase('pancetta-photos');
location.reload();
```

順に操作する。

1. 「＋ 新規仕込み」で肉460g・塩3%・砂糖1%・塩漬け5日・スパイス7日で登録 → 通知モーダルが開く → 閉じる
2. 「記録する」で重量440g・メモ「ラップでくるんで冷蔵庫で熟成」・写真1枚 → 履歴に1行増える
3. 「編集」で塩漬けを5→7に変更 → 通知ボタンが未登録の見た目に戻り、トーストが出る
4. リングの分母と完成予定日が新しい日数になっている
5. 履歴の「✕」で記録を1件削除 → 確認モーダルが出て、消える
6. 「完成にする」で★4と味メモを入れて保存 → アーカイブに移る
7. アーカイブで「記録を見る」を開く → 記録と写真が見える
8. ブラウザを再読み込み → 記録も写真もそのまま残っている

- [ ] **Step 7: コミット**

```bash
git add index.html CLAUDE.md
git commit -m "Show the log on finished batches, folded away until asked for

The archive is for reading back what worked, and the day-by-day record is
part of that. Keep it collapsed so the ratings still scan at a glance, and
drop the delete buttons - a finished batch is a record, not a draft.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 7: GitHub Pages へ反映する

**Files:** なし（コミット済みの内容を送るだけ）

- [ ] **Step 1: 変更が全部コミットされていることを確認する**

```bash
git status --short
```

期待: 何も表示されない（`.superpowers/` は `.gitignore` 済み）

- [ ] **Step 2: 送信する**

```bash
git push origin main
```

- [ ] **Step 3: 公開先で確認する**

1〜3分待ってから <https://qp52qp21-coder.github.io/pancetta-app/> をスマホで開く。

- 「記録する」「編集」のボタンがある
- 写真を1枚追加して保存できる
- ホーム画面に追加したアプリからも同じように動く

> 公開URLと `file://` では保存箱が別なので、手元で入れたテストデータは公開先には出ない。

---

## 自己レビュー結果

**仕様書との対応:**

| 仕様書の節 | 対応するタスク |
|---|---|
| データ構造（logs） | Task 1 |
| 写真の保存先（IndexedDB） | Task 2 |
| 既存データの移し替え | Task 1 |
| 熟成中カードのボタン | Task 3（記録する）/ Task 5（編集） |
| 記録モーダル | Task 3 / Task 4（写真欄） |
| 編集モーダル・仕込み重量の整合 | Task 5 |
| アーカイブの表示 | Task 6 |
| 写真の追加・表示・削除・失敗時 | Task 4 |
| カレンダー通知との連動 | Task 5 |
| 設計ルールの遵守 | Global Constraints・各タスクの画面確認 |
| 実装しないこと | 全タスクで着手しない |

**確認済み:** 各タスクが「先に失敗する検証 → 実装 → 通ることを確認 → コミット」の順になっていること、
後のタスクで使う関数名（`logRows`、`hydratePhotos`、`photoDelBatch`、`addLog`、`uid`）が
定義したタスクの綴りと一致していること、プレースホルダ（TBD・後で実装・適切に処理する等）が
無いことを確認した。
