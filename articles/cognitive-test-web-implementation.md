---
title: "認知科学テスト5種をWebアプリ化した — PVT・DSST・Nバック課題の実装ノート"
emoji: "🔬"
type: "tech"
topics: ["個人開発", "JavaScript", "認知科学", "React", "WebAPI"]
published: true
---

## はじめに

「自分の脳の調子を毎日数値で知りたい」。

そう思って認知科学の文献を漁り、5種類の認知テストをWebアプリとして実装した。[CortexLab](https://picoli.site/z-cortex-0002a) という名前でデプロイして、自分で3ヶ月使っている。

この記事では、5つのテストそれぞれの認知科学的な背景と、Web実装する上でのポイントをまとめる。

## 実装した5種類のテスト

| # | テスト名 | 測定対象 | 所要時間 |
|---|---------|---------|---------|
| 1 | PVT（精神運動覚醒テスト） | 覚醒度・反応速度 | 約2分 |
| 2 | DSST（数字符号置換テスト） | 処理速度 | 90秒 |
| 3 | ワーキングメモリテスト | 短期記憶容量 | 約2分 |
| 4 | パターン認識テスト | 論理的推論・ひらめき | 約1分 |
| 5 | マルチタスクテスト | 注意分割能力 | 約1分 |

全部で約5分。毎朝起きてから30分以内に実施する設計にしている。

---

## 1. PVT（Psychomotor Vigilance Task）

### 認知科学的背景

NASAが宇宙飛行士の覚醒度モニタリングに使用しているテスト。睡眠研究でもゴールドスタンダードとされている。

測定原理はシンプルで、ランダムなタイミングで現れる刺激に対する反応時間を測る。**覚醒度が低い（眠い・疲れている）と反応が遅くなる**。これは意志の力では隠せない。

正常範囲は200〜250ms。300ms以上は「ラプス」と呼ばれ、注意力の一時的な断絶を示す。

### 実装のポイント

```javascript
// 刺激間隔は2〜10秒のランダム
// 一様分布だと短い間隔が多くなりすぎるので注意
function getRandomISI() {
  // 2000〜10000msの一様分布
  return 2000 + Math.random() * 8000;
}
```

**ハマったこと**: `requestAnimationFrame` がタブ非アクティブ時に停止する問題。Page Visibility APIでテスト中のタブ切り替えを検知し、該当の試行を無効化する必要があった。

```javascript
document.addEventListener('visibilitychange', () => {
  if (document.hidden && testInProgress) {
    invalidateCurrentTrial();
  }
});
```

**反応入力**: `click` ではなく `mousedown` / `touchstart` を使う。`click` は `mouseup` 時に発火するため、指を離す時間が含まれてしまう。

---

## 2. DSST（Digit Symbol Substitution Test）

### 認知科学的背景

DSSTは1930年代にWechsler知能検査の一部として開発された。現在でも認知機能スクリーニングの標準的なテストとして臨床で広く使われている。

画面上部に「記号→数字」の対応表が表示される。画面下部に記号が次々と表示され、対応する数字をできるだけ速く入力する。90秒間の正答数がスコアになる。

処理速度・注意力・作業記憶を同時に測定できるのが特徴。

### 実装のポイント

対応表の記号セットをどう設計するかが重要だった。

```javascript
// 記号セット：視覚的に区別しやすく、文化に依存しない図形を使用
const SYMBOLS = ['◯', '△', '□', '◇', '☆', '⬡', '⬢', '⊕', '⊗'];

// 対応表はテストごとにランダムに生成
// → 覚えてしまう（学習効果）のを防ぐ
function generateMapping() {
  const digits = [1, 2, 3, 4, 5, 6, 7, 8, 9];
  const shuffled = [...digits].sort(() => Math.random() - 0.5);
  return Object.fromEntries(
    SYMBOLS.map((sym, i) => [sym, shuffled[i]])
  );
}
```

**毎回ランダムにする理由**: 同じ対応表だと、回数を重ねるうちに対応を暗記してしまい、純粋な処理速度ではなく記憶力のテストになってしまう。論文でも「毎回新しい対応表を使うべき」と推奨されている。

**入力UI**: モバイルではテンキーレイアウトのカスタムキーパッドを実装。ネイティブキーボードだと表示に時間がかかり、テストの流れが中断される。

```jsx
// カスタムテンキー（React）
function Numpad({ onInput }) {
  const keys = [[1, 2, 3], [4, 5, 6], [7, 8, 9]];

  return (
    <div className="grid grid-cols-3 gap-2">
      {keys.flat().map(n => (
        <button
          key={n}
          onTouchStart={(e) => {
            e.preventDefault();
            onInput(n);
          }}
          className="h-14 text-xl font-bold rounded-lg
                     bg-gray-100 active:bg-blue-200"
        >
          {n}
        </button>
      ))}
    </div>
  );
}
```

ここでも `onTouchStart` を使っているのは、`onClick` だとタッチデバイスで遅延が発生するため。

---

## 3. ワーキングメモリテスト（Grid Pattern Memory）

### 認知科学的背景

ワーキングメモリは「脳のRAM」に例えられる。情報を一時的に保持しながら操作する能力。Corsi Block-Tapping TaskやNバック課題が有名だが、今回は**グリッドパターン記憶テスト**を採用した。

理由は2つ：
1. Nバック課題は連続的で疲労が大きい（毎日やるには辛い）
2. グリッドパターンは視覚的にわかりやすく、難易度の段階的調整が容易

### 実装のポイント

難易度の段階的調整（適応的テスト）がこのテストの肝。

```javascript
// 適応的難易度調整
class AdaptiveGridTest {
  constructor() {
    this.gridSize = 3;      // 3x3からスタート
    this.activeCount = 3;   // 最初は3マスが光る
    this.level = 1;
    this.consecutiveCorrect = 0;
    this.consecutiveWrong = 0;
  }

  generatePattern() {
    const totalCells = this.gridSize * this.gridSize;
    const indices = Array.from({ length: totalCells }, (_, i) => i);

    // Fisher-Yatesシャッフルで偏りなくランダム選択
    for (let i = indices.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [indices[i], indices[j]] = [indices[j], indices[i]];
    }

    return new Set(indices.slice(0, this.activeCount));
  }

  adjustDifficulty(correct) {
    if (correct) {
      this.consecutiveCorrect++;
      this.consecutiveWrong = 0;

      // 2回連続正解で難易度アップ
      if (this.consecutiveCorrect >= 2) {
        this.level++;
        this.activeCount = Math.min(this.activeCount + 1, this.gridSize ** 2 - 1);

        // activeCountが上限に達したらグリッドサイズを拡大
        if (this.activeCount >= this.gridSize ** 2 * 0.7) {
          this.gridSize = Math.min(this.gridSize + 1, 6); // 最大6x6
        }
        this.consecutiveCorrect = 0;
      }
    } else {
      this.consecutiveWrong++;
      this.consecutiveCorrect = 0;

      // 2回連続不正解で難易度ダウン
      if (this.consecutiveWrong >= 2) {
        this.level = Math.max(1, this.level - 1);
        this.activeCount = Math.max(2, this.activeCount - 1);
        this.consecutiveWrong = 0;
      }
    }
  }
}
```

**パターン表示時間**: レベルに応じて表示時間を調整する。研究では「項目あたり約800ms」が短期記憶の符号化に必要とされている。

```javascript
function getDisplayDuration(activeCount) {
  // 基礎表示時間 + 項目あたりの時間
  return 500 + activeCount * 600; // ms
}
```

**正答判定**: 完全一致ではなく、部分一致のスコアリングも記録している。「8マス中7マス正解」の情報があると、ワーキングメモリの上限をより正確に推定できる。

---

## 4. パターン認識テスト

### 認知科学的背景

Ravenの漸進的マトリックスに着想を得たテスト。3x3のグリッドに図形パターンが並び、空欄に入る図形を選択肢から選ぶ。流動性知能（新しい問題を解く能力）の指標。

### 実装のポイント

問題の自動生成がチャレンジだった。

```javascript
// パターンルールの種類
const RULES = {
  ROTATION: 'rotation',       // 回転（45°, 90°, 180°）
  COLOR_SHIFT: 'colorShift',  // 色の変化
  SIZE_CHANGE: 'sizeChange',  // サイズ変化
  SHAPE_MORPH: 'shapeMorph',  // 形状変化
  COUNT_CHANGE: 'countChange', // 個数変化
};

// 難易度ごとにルールの組み合わせ数を変える
function generatePuzzle(difficulty) {
  const ruleCount = Math.min(difficulty, 3); // 最大3ルール同時
  const selectedRules = pickRandom(Object.values(RULES), ruleCount);

  // 各行・各列にルールを適用してグリッドを生成
  const grid = createGridFromRules(selectedRules);

  // 正解を一つ、ダミーを3つ生成
  const answer = grid[2][2]; // 右下が空欄
  const distractors = generateDistractors(answer, selectedRules);

  return { grid, answer, options: shuffle([answer, ...distractors]) };
}
```

**誤答選択肢の生成**が難しい。ランダムだと明らかに違う選択肢になってしまい、テストとして機能しない。「ルールの一部だけ満たしている」ダミーを生成する必要がある。

```javascript
function generateDistractors(answer, rules) {
  return rules.map(rule => {
    // 1つのルールだけ破った図形を生成
    return applyAllRulesExcept(answer, rules, rule);
  });
}
```

---

## 5. マルチタスクテスト

### 認知科学的背景

注意の分割能力を測るテスト。2つの独立したタスクを同時にこなす。

- **タスクA**: 画面左に数字が表示 → 奇数/偶数を判定
- **タスクB**: 画面右に文字が表示 → 母音/子音を判定

両方が同時に表示され、できるだけ速く両方に回答する。

### 実装のポイント

2つのタスクの入力を同時に受け付ける必要がある。PCではキーボード左手（Q/A）と右手（P/L）に割り当てた。

```javascript
// キーバインド
const BINDINGS = {
  // タスクA（奇偶判定）
  'q': 'odd',
  'a': 'even',
  // タスクB（母音子音判定）
  'p': 'vowel',
  'l': 'consonant',
};

// 両方のタスクへの回答を独立に計測
function handleKeydown(e) {
  const key = e.key.toLowerCase();
  const action = BINDINGS[key];
  if (!action) return;

  const now = performance.now();

  if (['odd', 'even'].includes(action) && !taskA.responded) {
    taskA.responded = true;
    taskA.reactionTime = now - stimulusOnset;
    taskA.correct = checkTaskA(action);
  }

  if (['vowel', 'consonant'].includes(action) && !taskB.responded) {
    taskB.responded = true;
    taskB.reactionTime = now - stimulusOnset;
    taskB.correct = checkTaskB(action);
  }

  // 両方回答したら次の試行へ
  if (taskA.responded && taskB.responded) {
    nextTrial();
  }
}
```

**モバイル対応**が一番の課題だった。2つのボタン群を同時に操作する必要があるため、`touch-action: none` でスクロールを無効化し、マルチタッチに対応した。

---

## スコアリングとトラッキング

5つのテストの結果を統合して、その日の「総合スコア」を算出する。

```javascript
function calculateCompositeScore(results) {
  // 各テストを0-100にスケーリング
  const normalized = {
    pvt: normalizePVT(results.pvt),          // 低いほど良い → 反転
    dsst: normalizeDSST(results.dsst),        // 高いほど良い
    memory: normalizeMemory(results.memory),   // 高いほど良い
    pattern: normalizePattern(results.pattern), // 高いほど良い
    multitask: normalizeMultitask(results.multitask), // 高いほど良い
  };

  // 重み付け平均（PVTは覚醒度の基本指標なので重めに）
  const weights = { pvt: 0.25, dsst: 0.2, memory: 0.2, pattern: 0.15, multitask: 0.2 };

  return Object.entries(weights).reduce(
    (sum, [key, weight]) => sum + normalized[key] * weight, 0
  );
}

// PVTの正規化（200ms以下=100, 400ms以上=0）
function normalizePVT(avgReactionTime) {
  return Math.max(0, Math.min(100,
    100 - ((avgReactionTime - 200) / 200) * 100
  ));
}
```

さらに、睡眠時間・カフェイン摂取量・運動の有無を毎日記録し、スコアとの相関を分析できるようにした。

3ヶ月使った結果、**睡眠7時間以上の日はスコアが安定して高い**、**前日に30分歩いた翌日はワーキングメモリが良い**、**コーヒー3杯以上の翌日はPVTが落ちる**ということがデータで見えた。

## まとめ

認知科学テストのWeb実装で重要だったこと：

1. **タイミング精度**: `performance.now()` + `requestAnimationFrame` が必須
2. **入力レイテンシ**: `mousedown` / `touchstart` を使い、`click` は避ける
3. **適応的難易度**: ワーキングメモリテストは個人の能力に合わせて動的に調整
4. **問題の自動生成**: パターン認識テストの誤答選択肢は「一部だけルールを破る」設計
5. **モバイル対応**: タッチ遅延、マルチタッチ、キーボード表示問題への対処

認知科学テストは仕様自体はシンプルだが、「正確に測定する」というハードルが想像以上に高かった。ブラウザのタイミングAPIの制約を理解していないと、データとしての信頼性が担保できない。

CortexLabでは上記5種類のテストを毎日5分で受けられる。自分の脳のパフォーマンスが何に影響されているか、データで可視化してみたい人は試してみてほしい。

https://picoli.site/z-cortex-0002b
