---
title: "カケアミの数理 — Voronoi分割とBFS貪欲法でマンガのハッチングを自動生成"
emoji: "✏️"
type: "tech"
topics: ["漫画", "計算幾何", "voronoi", "typescript", "アルゴリズム"]
published: false
---

:::message
- 漫画のハッチング技法「カケアミ」を、Poisson-Disk Sampling・Voronoi分割・BFS貪欲角度割当で自動生成するツールを作った
- 隣接タイルの角度差を最大化する問題を、実射影直線 $\text{RP}^1$ 上のグラフ彩色として定式化した
- 2×2要因計画の実験で、角度割当アルゴリズムが品質の**94%**を説明することを確認した（$\eta^2 = 0.94$）
:::

**ツール本体**: https://kakeami.github.io/kakeami-generator/
**実験レポート**: https://kakeami.github.io/kakeami-generator/experiments/

## カケアミとは

カケアミは、漫画で使われるハッチング技法の一つです。短い線を複数方向に重ねて描くことで、影や質感を表現します。

スクリーントーン（点描のシールを切り貼りする技法）が普及する以前は、影の表現手段としてカケアミが広く使われていました。現在ではスクリーントーンやデジタルトーンで代替可能なため、利用頻度は落ち着いていますが、手描きの温かみや独特の味わいを求めて、今でもカケアミを使う作家は存在します。

カケアミの流儀は多様ですが、本稿では**隣接するタイルが互いに異なる角度のハッチング線を持つ**パターンを対象とします。重ねる方向の数に応じて、1掛け・2掛け・3掛け・4掛けと呼ばれます。

![1掛けから4掛けまでのカケアミ](/images/2026-03-kakeami-generator/fig_kake_comparison.png)
*1掛けから4掛けまでの比較。掛け数 $k$ が増えるほど、各タイルに重ねるハッチング方向が増える*

自分で描いてみると、カケアミは意外と難しいことに気づきます。タイルの配置に適度なランダム性が必要で、かつ近隣タイルと角度が被らないように調整しなければなりません。手描きの場合はこれを直感的にこなすわけですが、計算機で自動生成するにはどうすればよいでしょうか？

調べてみると、カケアミの学術的定義は（私が調べた限り）存在しません。自動生成手法も評価手法も見当たりませんでした[^prior]。そこで、数学のタイル敷き詰め問題やグラフ彩色の知見を応用すれば自動生成できるのではないかと考え、このツールを作りました。

[^prior]: NPR（Non-Photorealistic Rendering）分野にハッチング生成の研究はありますが、カケアミ特有の「隣接タイルの角度差最大化」を扱ったものは見当たりませんでした。CLIP STUDIOの「手書き風カケアミペン」のような商用ツールは存在しますが、アルゴリズムの詳細は非公開です。

## 問題設定: なぜ「角度割当」が難しいのか

「各タイルにランダムな角度を割り当てればよいのでは？」と思うかもしれません。ところが、単純なランダム割当では隣接タイル同士が近い角度になることがあり、パターンにムラが生じます。

![ランダム割当 vs BFS貪欲割当](/images/2026-03-kakeami-generator/fig_random_vs_bfs.png)
*左: ランダム角度割当。隣接タイルが同じ向きになる箇所がある。右: BFS貪欲角度割当。隣接タイル間の角度差が大きく、均一な印象*

この問題を整理すると、**グラフ彩色の連続版**として定式化できます。

- **頂点**: 各タイル $i$ に角度 $\varphi_i \in [0, \pi)$ を割り当てる
- **辺**: 隣接するタイル対 $(i, j) \in E$
- **目的**: 隣接タイル間の角度差（後述する $\text{RP}^1$ 距離）を最大化する

具体的には、各ノードの割当時に**隣接する配置済みノードとの最小距離を最大化する**（maxmin戦略）目的関数を用います。

$$
\varphi_i^* = \arg\max_{\varphi \in [0, \pi)} \min_{j \in N_{\text{placed}}(i)} d_{\text{RP}^1}(\varphi, \varphi_j)
$$

ここで $N_{\text{placed}}(i)$ はノード $i$ の隣接ノードのうち、すでに角度が割り当てられたものの集合です。

## 要素技術の解説

全体のパイプラインは4段階です。

![アルゴリズムパイプライン](/images/2026-03-kakeami-generator/algorithm_pipeline.png)
*Poisson-Disk Sampling → Voronoi隣接グラフ構築 → BFS貪欲角度割当 → レンダリング*

### Poisson-Disk Sampling

タイルの中心点をどこに配置するかが最初の問題です。格子状に配置すると人工的な周期パターンが見えてしまい、完全にランダムに配置するとタイルが重なったり隙間が空いたりします。

![Poisson-Disk配置パターン](/images/2026-03-kakeami-generator/pattern-stipple.png)
*Poisson-Disk Samplingによるタイル配置。最小距離が保証されつつ、適度にランダム*

Poisson-Disk Samplingは、**任意の2点間の距離が最小距離 $r$ 以上**であることを保証するサンプリング手法です。Bridsonのアルゴリズム[^bridson]により、$O(n)$ の計算量で生成できます。

[^bridson]: Bridson, R. "Fast Poisson Disk Sampling in Arbitrary Dimensions" (SIGGRAPH 2007 Sketches)

$$
\forall i \neq j: \| \mathbf{p}_i - \mathbf{p}_j \| \geq r
$$

本ツールでは `tileSize` パラメータで最小距離 $r$ を制御しています。

### Voronoi分割と隣接グラフ

タイル中心が決まったら、隣接関係を定義する必要があります。ここでVoronoi分割（とその双対であるDelaunay三角形分割）が登場します。

![Voronoiパターン](/images/2026-03-kakeami-generator/pattern-voronoi.png)
*Voronoiセルの可視化。各タイル中心（母点）を囲む領域がVoronoiセル*

母点集合 $P = \{\mathbf{p}_1, \ldots, \mathbf{p}_n\}$ に対し、Voronoiセル $V_i$ は次のように定義されます。

$$
V_i = \{ \mathbf{x} \in \mathbb{R}^2 \mid \| \mathbf{x} - \mathbf{p}_i \| \leq \| \mathbf{x} - \mathbf{p}_j \| \quad \forall j \neq i \}
$$

Voronoiセルが辺を共有する2つのタイルを「隣接」とみなします。これはDelaunay三角形分割の辺に対応しています。

![Delaunay三角形分割とVoronoi図](/images/2026-03-kakeami-generator/fig_delaunay_voronoi.png)
*Delaunay三角形分割（青）とVoronoiセル境界（赤）の双対関係*

実装では `d3-delaunay` ライブラリを用いて $O(n \log n)$ でDelaunay三角形分割を構築し、そこから隣接リストを導出しています。

### $\text{RP}^1$ 距離

カケアミのハッチング線は**向きを持たない直線**です。つまり、角度 $0°$ と $180°$ は同じ方向を表します。この性質から、角度の空間は通常の円 $S^1$（$[0, 2\pi)$）ではなく、**実射影直線** $\text{RP}^1$（$[0, \pi)$ で両端を同一視）上に存在します。

![RP1距離の可視化](/images/2026-03-kakeami-generator/fig_rp1_circle.png)
*$\text{RP}^1$ 上の距離。$\varphi_1 = 10°$ と $\varphi_2 = 170°$ の通常の差は $160°$ だが、$\text{RP}^1$ 距離は $20°$（$0°$ と $180°$ が同一視されるため）。最大距離は $\pi / 2 = 90°$（直交）*

$\text{RP}^1$ 上のFubini-Study距離は次のように定義されます。

$$
d_{\text{RP}^1}(\varphi_1, \varphi_2) = \min(|\varphi_1 - \varphi_2| \bmod \pi, \; \pi - |\varphi_1 - \varphi_2| \bmod \pi)
$$

この距離の値域は $[0, \pi/2]$ で、最大値 $\pi/2$ は2つの角度が**直交**していることを意味します。

実装は非常にシンプルです。

```typescript
/** RP¹ Fubini-Study distance. Range [0, π/2]. */
export function dRp1(t1: number, t2: number): number {
  const diff = ((Math.abs(t1 - t2) % PI) + PI) % PI;
  return Math.min(diff, PI - diff);
}
```

### BFS貪欲角度割当

隣接グラフと距離関数が揃ったところで、いよいよ角度割当です。

1. **最大次数のノードからBFS開始**: 制約が最も厳しいノードを最初に処理する
2. **各未訪問ノードに対し、360個の候補角度から最適を選択**: maxmin戦略で、配置済み隣接ノードとの最小 $\text{RP}^1$ 距離を最大化する
3. **孤立ノードにはランダム角度を割当**

![BFS貪欲角度割当のステップ](/images/2026-03-kakeami-generator/fig_bfs_steps.png)
*BFS貪欲角度割当の進行。ノードの色はHSVカラーマップで角度に対応。BFS順序で外側に広がりながら、各ノードに最適な角度を割り当てていく*

BFS順序を採用する理由は、ノードの角度を決定する時点で**隣接ノードの多くがすでに配置済み**である確率が高いためです。ランダム順やDFS順では、隣接ノードがまだ未配置の状態で決定を迫られることが多く、最適化の品質が下がります。

### 品質指標

生成されたカケアミパターンの品質を定量的に評価するため、6種類の指標を定義しました。

#### 角度系指標

| 指標 | 値域 | 望ましい方向 | 意味 |
|------|------|------------|------|
| $E_{\text{contrast}}$ | [0, 1] | 高い | 隣接タイル間の平均角度コントラスト |
| $E_{\text{LdG}}$ | [0, 1] | 高い | Landau-de Gennesエネルギー |
| $S$ | [0, 1] | 低い | スカラー秩序パラメータ（等方性） |
| $H_{\text{angle}}$ | [0, 1] | 高い | 角度分布のシャノンエントロピー |

それぞれの定義は次の通りです。

$$
E_{\text{contrast}} = \frac{2}{\pi|E|} \sum_{(i,j) \in E} d_{\text{RP}^1}(\varphi_i, \varphi_j)
$$

$$
E_{\text{LdG}} = \frac{1}{|E|} \sum_{(i,j) \in E} \sin^2(\varphi_i - \varphi_j)
$$

$$
S = \left| \frac{1}{n} \sum_{i=1}^{n} e^{2i\varphi_i} \right|
$$

$$
H_{\text{angle}} = -\frac{1}{\ln 12} \sum_{b=1}^{12} p_b \ln p_b
$$

#### 空間系指標

| 指標 | 値域 | 望ましい方向 | 意味 |
|------|------|------------|------|
| $C_{\text{cov}}$ | [0, 1] | 高い | 領域の被覆率 |
| $U_{\text{vor}}$ | [0, ∞) | 低い | Voronoiセル面積の変動係数（CV） |

$E_{\text{contrast}}$ は $\text{RP}^1$ 距離の平均を $[0, 1]$ に正規化したもので、最も直感的な品質指標です。$S$（秩序パラメータ）は、角度が一方向に偏っていると $1$ に近づき、等方的に分散していると $0$ に近づきます。$H_{\text{angle}}$ は角度分布のシャノンエントロピーで、多様な角度が使われているほど高くなります。

:::details $E_{\text{LdG}}$ の由来: Landau-de Gennes理論

$E_{\text{LdG}} = \frac{1}{|E|} \sum_{(i,j) \in E} \sin^2(\varphi_i - \varphi_j)$ は、ネマティック液晶の物理で用いられるLandau-de Gennesエネルギーに着想を得ています。

液晶分子は棒状で向きを持たない（$0°$ と $180°$ が同一）ため、その配向場は $\text{RP}^1$ 上の値を取ります。Landau-de Gennes理論では、隣接する液晶分子の配向差に対して $\sin^2$ のエネルギーペナルティを課します。$\sin^2(\Delta\varphi)$ は $\Delta\varphi = 0$ で $0$、$\Delta\varphi = \pi/2$（直交）で最大値 $1$ となります。

カケアミでは隣接タイルが直交に近いほど見栄えが良いため、$E_{\text{LdG}}$ が高いほど良い、と解釈できます。$E_{\text{contrast}}$ とは単調関係にありますが、$\sin^2$ の非線形性により、直交からのずれに対してより敏感に反応します。

メッシュ生成における方向場設計（cross-field design）の文脈では、Ginzburg-Landau理論に基づく類似のエネルギー汎関数が広く使われています。

:::

## 実験: 何が品質を決めるのか

「Poisson-Disk配置」と「BFS貪欲角度割当」のどちらが品質に効くのかを定量的に検証するため、2×2の要因計画で実験を行いました。

| | BFS角度割当 | Random角度割当 |
|---|---|---|
| **Poisson配置** | poissonBfs（**提案手法**） | poissonRandom |
| **Random配置** | randomBfs | randomRandom |

追加で格子+市松模様（gridCheckerboard）を診断用リファレンスとして含め、計5条件 × 100シード = 500試行を実施しました（$k = 1$）。

### 主要結果

| 条件 | $E_{\text{contrast}}$ | $E_{\text{LdG}}$ | $S$ (↓) | $H_{\text{angle}}$ | $C_{\text{cov}}$ | $U_{\text{vor}}$ (↓) |
|------|-----------|-------|--------|---------|-------|-----------|
| **Poisson+BFS** | 0.640±0.005 | 0.676±0.008 | 0.042±0.022 | 0.960±0.022 | **0.948±0.010** | **0.240±0.021** |
| Poisson+Random | 0.497±0.026 | 0.497±0.032 | 0.101±0.048 | 0.968±0.012 | 0.947±0.010 | 0.240±0.021 |
| Random+BFS | 0.642±0.006 | 0.679±0.009 | 0.042±0.021 | 0.959±0.022 | 0.749±0.030 | 0.558±0.073 |
| Random+Random | 0.500±0.023 | 0.500±0.027 | 0.103±0.055 | 0.967±0.015 | 0.749±0.030 | 0.558±0.073 |
| Grid+Checker | **0.696** | **0.696** | **0.000** | 0.279 | 1.000 | 0.000 |

### Two-Way ANOVAの結果

$E_{\text{contrast}}$ に対するTwo-Way ANOVA:

- **配置要因**: $F = 1.47$, $p = 0.226$, $\eta^2 = 0.0002$（寄与なし）
- **角度要因**: $F = 6512.32$, $p < 0.0001$, $\eta^2 = 0.9425$（**支配的**）
- **交互作用**: $F = 0.07$, $p = 0.787$, $\eta^2 \approx 0$（なし）

**角度割当アルゴリズム（BFS vs Random）が品質の94%を説明します。** 配置方法（Poisson vs Random）は角度品質にほぼ影響しません。

$$
\eta^2 = \frac{SS_{\text{factor}}}{SS_{\text{total}}}
$$

![E_contrastの箱ひげ図](/images/2026-03-kakeami-generator/boxplot_eContrast.png)
*$E_{\text{contrast}}$ の箱ひげ図。BFS条件（右2つ）はRandom条件（左2つ）と明確に分離している*

![H_angleの箱ひげ図](/images/2026-03-kakeami-generator/boxplot_hAngle.png)
*$H_{\text{angle}}$ の箱ひげ図。gridCheckerboardは2角度しか使わないため、エントロピーが著しく低い*

### 5条件 k=1 の視覚比較

各条件の代表画像（$E_{\text{contrast}}$ の中央値に最も近いシード）を並べてみましょう。

| poissonBfs | poissonRandom | randomBfs | randomRandom | gridCheckerboard |
|:---:|:---:|:---:|:---:|:---:|
| ![](/images/2026-03-kakeami-generator/poissonBfs_k1.png) | ![](/images/2026-03-kakeami-generator/poissonRandom_k1.png) | ![](/images/2026-03-kakeami-generator/randomBfs_k1.png) | ![](/images/2026-03-kakeami-generator/randomRandom_k1.png) | ![](/images/2026-03-kakeami-generator/gridCheckerboard_k1.png) |

Random配置の画像（randomBfs, randomRandom）は被覆率が低くて隙間が目立ちます。gridCheckerboardは数値上は優秀ですが、見た目は人工的で不自然です（$H_{\text{angle}} = 0.279$ が物語っています）。

### gridCheckerboardの教訓

Grid+Checkerboardは、$E_{\text{contrast}}$, $E_{\text{LdG}}$, $S$, $C_{\text{cov}}$, $U_{\text{vor}}$ のいずれでも最高スコアを記録しましたが、$H_{\text{angle}} = 0.279$ と著しく低い値を示しました。2角度しか使わないため、エントロピーが極端に低いのです。これは、$H_{\text{angle}}$ が「角度の多様性」を評価する上で不可欠な指標であることを示しています。単一の指標だけでは品質を捉えきれません。

![C_covの箱ひげ図](/images/2026-03-kakeami-generator/boxplot_cCov.png)
*$C_{\text{cov}}$（被覆率）の箱ひげ図。Poisson配置はRandom配置に対して被覆率で圧倒的に優位*

### 提案手法（Poisson+BFS）の総合評価

- **角度指標**: BFS条件の中でRandom+BFSとほぼ同等（差は微小）
- **空間指標**: Random配置に対して圧倒的に優位（$C_{\text{cov}}$: 0.948 vs 0.749, $U_{\text{vor}}$: 0.240 vs 0.558）
- **総合的にPoisson+BFSが最もバランスの良い条件**

角度品質はBFSが支配し、空間品質はPoissonが支配する。両者は独立に寄与しており、組み合わせることで総合的に最良のパターンが得られます。

## 使ってみる

![ツールのUI](/images/2026-03-kakeami-generator/fig_ui_screenshot.png)
*kakeami-generatorのデモUI。左側にカケアミのプレビュー、右側にパラメータコントロール*

https://kakeami.github.io/kakeami-generator/ にアクセスすれば、ブラウザ上ですぐに試せます。

### 主要パラメータ

| パラメータ | 説明 | デフォルト |
|-----------|------|----------|
| `regionSize` | 領域の大きさ | 12 |
| `tileSize` | タイルの最小間隔（Poisson-Disk半径） | 0.7 |
| `k` | 掛け数（1〜4） | プリセット依存 |
| `pitch` | ハッチング線の間隔 | 0.05 |
| `lineWeight` | 線の太さ | 0.5 |
| `seed` | 乱数シード（再現性のため） | ランダム |

### プリセット

1-kake / 2-kake / 3-kake / 4-kake の4つのプリセットが用意されており、ワンクリックで切り替えられます。

### エクスポート

- **SVG**: ベクター形式でダウンロード。印刷やさらなる加工に適している
- **PNG**: 4096×4096ピクセルのラスター画像としてダウンロード

### URL共有

すべてのパラメータがURLのクエリパラメータに反映されるため、URLをコピーするだけで設定を共有できます。

## 余談: Claude Codeとの共同開発

このツールは、私が**1行もソースコードを書かず**に開発しました。要件定義・研究方針の策定・分析設計のディレクションのみを行い、実装はすべてClaude Codeに委ねています。

TypeScriptでのコアアルゴリズム実装、Vitによるビルド構成、Playwrightによる515条件の実験自動化、Pythonによる統計分析、HTMLレポートの生成まで、すべてClaude Codeが生成したコードで動いています。

これは「AIがコードを書く」ことの話ではなく、**人間の役割がコーディングからディレクションに移った**ことの話です。ドメイン知識（カケアミとは何か、何が品質の良いパターンか）と研究設計（何を測り、どう比較するか）は依然として人間の仕事であり、その部分こそが成果物の質を左右しました。

## おわりに

カケアミという身近な漫画技法の中に、Voronoi分割、実射影直線 $\text{RP}^1$、Landau-de Gennes理論、グラフ彩色、シャノンエントロピーといった数理概念が自然に現れることを示しました。

この記事で紹介した実験のすべてのデータ、分析コード、生成画像は[実験レポート](https://kakeami.github.io/kakeami-generator/experiments/)として公開しています。品質指標の改善、新しい角度割当アルゴリズムの提案、あるいは実際の漫画制作への応用など、コラボレーションを歓迎します。

カケアミは、漫画文化の中で長い歴史を持ちながら、学術的にはほぼ手つかずの領域です。この小さなツールが、カケアミへの関心を再び高めるきっかけになれば幸いです。

## 参考リンク

### カケアミの文化的背景

- [Pixiv Encyclopedia「カケアミ」](https://dic.pixiv.net/a/%E3%82%AB%E3%82%B1%E3%82%A2%E3%83%9F)
- [Japanese with Anime「Kakeami」(2020)](https://www.japanesewithanime.com/2020/03/kakeami.html)
- [Emiya Manga Class「カケアミの描き方」(2018)](http://lanmy.net/2018/05/manga-kakeami/)
- 東堂, R.「マンガの基礎デッサン」(2003), Hobby Japan

### 先行ツール

- [CLIP STUDIO カケアミペン](https://assets.clip-studio.com/ja-jp/detail?id=1364005)

### NPRハッチング

- Praun, E. et al. "Real-Time Hatching" (SIGGRAPH 2001)
- Hertzmann, A. & Zorin, D. "Illustrating Smooth Surfaces" (SIGGRAPH 2000)

### タイリング・Voronoi

- Secord, A. "Weighted Voronoi Stippling" (NPAR 2002)
- Okabe, A. et al. "Spatial Tessellations" (2000), Wiley

### 方向場設計・Landau-de Gennes理論

- Beaufort, P.-A. et al. "Computing Cross Fields — A PDE Approach Based on the Ginzburg-Landau Theory" (2017), arXiv:1706.01344
- Viertel, R. & Osting, B. "An Approach to Quad Meshing Based on Harmonic Cross-Valued Maps and the Ginzburg-Landau Theory" (2019), SIAM J. Sci. Comput.
- Vaxman, A. et al. "Directional Field Synthesis, Design, and Processing" (2016), Computer Graphics Forum
