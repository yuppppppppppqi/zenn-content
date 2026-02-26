---
title: "requestAnimationFrameで脳の反応速度を測る — PVT実装で学んだミリ秒精度の罠"
emoji: "🧠"
type: "tech"
topics: ["JavaScript", "requestAnimationFrame", "個人開発", "認知科学", "パフォーマンス"]
published: true
---

## はじめに

「脳の反応速度をブラウザで測りたい」。

認知科学の分野に **PVT（Psychomotor Vigilance Task / 精神運動覚醒テスト）** というテストがある。画面にランダムなタイミングで刺激が出て、それに反応するまでの時間を測る。NASAが宇宙飛行士の覚醒度チェックに使っているやつだ。

やることはシンプル。画面が光る → ボタンを押す → 反応時間を記録する。

これをWebアプリとして実装したら、ミリ秒精度のタイミング計測でハマりまくった。この記事では、PVT実装を通じて学んだブラウザのタイミングAPIの罠と、その対処法をまとめる。

実装したのは [CortexLab](https://picoli.site/z-cortex-0001a) という認知科学テストアプリ。5種類の認知テストで脳のパフォーマンスを数値化できる。

## PVTの仕様

PVTのルールはこうだ。

1. 画面に何も表示されていない状態で待つ（2〜10秒のランダム間隔）
2. 刺激が表示される（カウンターが動き始める）
3. できるだけ速くボタンを押す
4. 反応時間が記録される
5. これを10回繰り返す

測定するのは：
- **平均反応時間**（ms）
- **最速 / 最遅**
- **ラプス数**（500ms以上の遅延 = 注意散漫の指標）

臨床研究では平均反応時間200〜250msが正常範囲。300msを超えると睡眠不足や注意力低下のサインとされている。

## 最初の実装：setTimeoutの罠

最初は素朴にこう書いた。

```javascript
// ❌ 最初の実装（精度が低い）
function startTrial() {
  const delay = 2000 + Math.random() * 8000; // 2〜10秒

  setTimeout(() => {
    const startTime = Date.now();
    showStimulus();

    button.addEventListener('click', () => {
      const reactionTime = Date.now() - startTime;
      recordResult(reactionTime);
    }, { once: true });
  }, delay);
}
```

問題は2つある。

### 問題1: setTimeoutの精度

`setTimeout` の精度はブラウザやOS負荷に依存する。HTML仕様上は最小4msだが、実際にはもっとブレる。

```javascript
// 検証コード
const results = [];
for (let i = 0; i < 100; i++) {
  const start = performance.now();
  await new Promise(r => setTimeout(r, 0));
  results.push(performance.now() - start);
}
console.log('平均:', results.reduce((a, b) => a + b) / results.length);
// → Chrome: 約1.2ms, Firefox: 約1.5ms
// → タブが非アクティブ: 約1000ms（！）
```

PVTは反応時間が200ms前後のテストだ。10msの誤差は5%の測定エラーに相当する。`setTimeout` のブレは許容できない。

### 問題2: Date.nowの精度

`Date.now()` はミリ秒精度だが、実際の分解能はそれ以下の場合がある。またSpectre対策でブラウザが意図的に精度を落としている。

```javascript
// Date.now()の分解能チェック
const timestamps = new Set();
for (let i = 0; i < 1000; i++) {
  timestamps.add(Date.now());
}
console.log('ユニーク数:', timestamps.size);
// → 期待: 1000に近い, 実際: ブラウザによって100〜500程度
```

## 改善：performance.now + rAFベース

`setTimeout` の代わりに `requestAnimationFrame` ベースのタイマーに切り替えた。

```javascript
// ✅ 改善版
function waitForDelay(delayMs) {
  return new Promise(resolve => {
    const start = performance.now();

    function check() {
      if (performance.now() - start >= delayMs) {
        resolve();
      } else {
        requestAnimationFrame(check);
      }
    }
    requestAnimationFrame(check);
  });
}

async function startTrial() {
  const delay = 2000 + Math.random() * 8000;
  await waitForDelay(delay);

  const startTime = performance.now();
  showStimulus();

  button.addEventListener('click', () => {
    const reactionTime = performance.now() - startTime;
    recordResult(Math.round(reactionTime));
  }, { once: true });
}
```

`performance.now()` は `Date.now()` よりも高精度（マイクロ秒単位）で、`requestAnimationFrame` は次のフレーム描画に同期して実行されるため、画面更新と測定のタイミングが一致する。

## 次の罠：タブの非アクティブ問題

`requestAnimationFrame` にも致命的な問題があった。**タブが非アクティブになると実行頻度が激減する**。

```javascript
// rAFの実行頻度テスト
let count = 0;
const start = performance.now();

function tick() {
  count++;
  if (performance.now() - start < 5000) {
    requestAnimationFrame(tick);
  } else {
    console.log(`5秒間のrAF実行回数: ${count}`);
    // アクティブタブ: 約300回（60fps）
    // 非アクティブタブ: 約5回（1fps以下）
  }
}
requestAnimationFrame(tick);
```

PVTのテスト中にユーザーがタブを切り替えると、刺激の表示タイミングがずれる。もっと悪いことに、反応時間の計測自体が不正確になる。

### 対策：Page Visibility API

```javascript
function startTest() {
  // テスト開始時にVisibility監視
  const handleVisibility = () => {
    if (document.hidden) {
      pauseTest();
      showWarning('タブに戻ってきてください');
    } else {
      // 非アクティブから復帰した場合、現在の試行を無効化して再開
      invalidateCurrentTrial();
      resumeTest();
    }
  };

  document.addEventListener('visibilitychange', handleVisibility);
}

function invalidateCurrentTrial() {
  // 非アクティブ中の試行データは捨てる
  currentTrialValid = false;
}
```

さらに、テスト画面をフルスクリーンにして意図しないタブ切り替えを抑制した。

```javascript
async function enterFullscreen() {
  try {
    await document.documentElement.requestFullscreen();
  } catch {
    // フルスクリーン非対応の場合は警告だけ出す
    showWarning('フルスクリーンで正確な計測ができます');
  }
}
```

## フレームタイミングの問題

もう一つ厄介だったのが、**刺激の表示が次のフレーム描画まで遅延する**問題だ。

```javascript
// ❌ 表示と計測開始のタイミングがズレる
const startTime = performance.now();
stimulus.style.display = 'block'; // ← この時点ではまだ画面に描画されていない
```

DOMの変更は即座に画面に反映されない。ブラウザの描画パイプライン（Style → Layout → Paint → Composite）を経て、次のフレームで初めて画面に表示される。

```javascript
// ✅ 描画タイミングに合わせて計測開始
function showStimulusAndMeasure() {
  stimulus.style.display = 'block';

  // 強制的にレイアウトを発生させる
  stimulus.offsetHeight;

  // 次のフレーム描画時に計測開始
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      // 2フレーム後 = 確実に画面に表示されている
      stimulusShownTime = performance.now();
    });
  });
}
```

`rAF` を2回ネストしているのは、1回目の `rAF` コールバック内ではまだ描画が完了していない可能性があるため。2回目のコールバック時点で「前のフレームが確実に描画されている」ことが保証される。

## タッチイベントのレイテンシ

モバイルではもう一段階罠がある。**タッチイベントの300ms遅延**だ。

かつてモバイルブラウザはダブルタップズームのために `click` イベントを300ms遅延させていた。現在は `touch-action: manipulation` やビューポート設定で解消できるが、念のため `touchstart` でも反応を取るようにした。

```css
/* CSSでタッチ遅延を解消 */
.reaction-button {
  touch-action: manipulation;
}
```

```javascript
// touchstartとclickの両方をリスンする
function listenForResponse(callback) {
  let responded = false;

  const handler = (e) => {
    if (responded) return;
    responded = true;
    e.preventDefault();
    callback(performance.now());
  };

  button.addEventListener('touchstart', handler, { once: true, passive: false });
  button.addEventListener('mousedown', handler, { once: true });
}
```

`click` ではなく `mousedown` / `touchstart` を使っているのは、`click` は `mouseup` 後に発火するため、指を離すまでの時間が反応時間に含まれてしまうから。PVTでは「刺激を認知して運動指令を出す速度」を測りたいので、`mousedown`（指が触れた瞬間）が正しい。

## 計測精度の検証

最終的にどの程度の精度が出ているか検証した。

```javascript
// 自動テスト：既知の遅延を入れて計測精度を確認
async function calibrationTest() {
  const delays = [100, 200, 300, 500];
  const results = [];

  for (const expectedDelay of delays) {
    const measured = [];
    for (let i = 0; i < 50; i++) {
      const start = performance.now();
      await new Promise(r => setTimeout(r, expectedDelay));
      measured.push(performance.now() - start);
    }
    const avg = measured.reduce((a, b) => a + b) / measured.length;
    const stddev = Math.sqrt(
      measured.reduce((s, v) => s + (v - avg) ** 2, 0) / measured.length
    );
    results.push({ expectedDelay, avg: avg.toFixed(1), stddev: stddev.toFixed(1) });
  }
  console.table(results);
}
```

結果（Chrome 122, M1 Mac）:

| 期待値(ms) | 実測平均(ms) | 標準偏差(ms) |
|-----------|-------------|-------------|
| 100 | 101.3 | 1.2 |
| 200 | 201.1 | 1.4 |
| 300 | 301.2 | 1.1 |
| 500 | 501.0 | 1.3 |

誤差は±2ms以内。PVTの測定に十分な精度が確保できた。

## まとめ

PVT実装で学んだミリ秒精度タイミングのポイント：

1. **`setTimeout` は使わない** → `requestAnimationFrame` + `performance.now()` を使う
2. **タブの非アクティブを検知する** → Page Visibility APIで試行を無効化
3. **描画タイミングを考慮する** → `rAF` 2回ネストで確実に描画完了後を捉える
4. **モバイルのタッチ遅延に対応** → `touchstart` + `touch-action: manipulation`
5. **`click` ではなく `mousedown`** → 指を離すまでの時間を排除

ブラウザのタイミングAPIは「だいたい合っている」レベルでは使えるが、「正確に測る」となると罠だらけだった。ゲーム開発やリアルタイム系のアプリでも同じ知見が使えるはず。

CortexLabではPVT以外にもDSST、ワーキングメモリ、パターン認識、マルチタスクの計5種類のテストを実装していて、毎日のスコアトラッキングや生活習慣との相関分析もできる。興味があれば試してみてほしい。

https://picoli.site/z-cortex-0001b
