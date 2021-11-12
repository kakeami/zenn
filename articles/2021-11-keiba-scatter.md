---
title: "ウマ娘にハマったので35年分のレース結果をPlotlyで散布図にした"
emoji: "🏇"
type: "tech"
topics: ["python", "pandas", "競馬", "plotly", "可視化"]
published: false
---

- アニメ[ウマ娘プリティーダービー](https://anime-umamusume.jp/)にハマった
- 原作（実際の競馬）に興味が出たので，35年分の重賞[^graded]のレース結果を[netkeiba](https://www.netkeiba.com/)から取得した
- [Plotly](https://plotly.com/python/)でインタラクティブな散布図を描き，馬・タイトル・適性毎に分布を見てニヤニヤした
- （記事が長すぎるので，YouTubeの字幕をONにしてデモだけでも御覧いただければ幸甚です ）

https://youtu.be/fBI9aKwAhfM

https://youtu.be/tSBXADm7pv8

https://github.com/kakeami/keiba-eda-public

[^graded]: 厳密にはnetkeibaから35年分の中央競馬の全結果を取得し，レース名に`[G1, G2, G3, G, L]`のいずれかが付与された平地競走のみフィルタリングした上でプロットしました．

# はじめに

[ウマ娘プリティーダービー](https://umamusume.jp/)（以下，ウマ娘）とはCygamesによるスマホ向けゲームを中心とするとメディアミックスコンテンツ[^trendy]です．[テレビアニメ](https://anime-umamusume.jp/)は2018年4月から6月まで第1期，2021年1月から3月まで第2期が放送されました．私は[当時それどころではなかった](https://zenn.dev/kakeami/articles/617efe3bf9841d)こともありリアルタイムで視聴できませんでしたが，のちほど全話視聴し，見事にハマってしまいました．

[^trendy]: [2021年11月に日経トレンディおよび日経クロストレンドから発表された「2021年ヒット商品30」では2位に選出](https://xtrend.nikkei.com/atcl/contents/18/00549/00003/)されました．また，Cygamesはおろか，親会社のサイバーエージェントのゲーム事業の売上高が過去最高を更新したことにも驚きました．

ウマ娘の素晴しい点の一つは，競馬へのただならぬ愛情とリスペクト[^rocket][^note]です．競走馬の外観や体格（やときにはジョッキーの勝負服）を反映したキャラクターデザイン[^hachimitsu]，競走馬の気性やエピソードを考慮した脚本[^nice]，そして実況含め史実を忠実に再現したレース展開[^jikkyo]，と枚挙にいとまがありません．熱にあてられたプレーヤーや視聴者が興味を持ち始め，競馬ファンの拡大に貢献しました[^netkeiba-news]．2021年上半期には，ウマ娘をダウンロードした28.7%のユーザが[競馬情報アプリnetkeiba](https://dir.netkeiba.com/app/index.html)をダウンロードしていたそうです[^4gamer]．

[^rocket]: https://rocketnews24.com/2021/03/19/1470546/
[^note]: https://note.com/cmr2tf/n/n8f3e4e7574e3
[^netkeiba-news]: https://news.netkeiba.com/?pid=column_view&cid=49766
[^4gamer]: https://www.4gamer.net/games/522/G052249/20210901047/
[^nice]: ゲーム中，ナイスネイチャとトレーナーの距離が近いのは，[モデルとなった競走馬のナイスネイチャが担当厩務員に非常になついていた](https://xn--o9j0bk9l4k169rk1cxv4aci7a739c.com/post-7082)からだ，という説があります．
[^hachimitsu]: [イラストだけで競走馬を当てる企画](https://www.youtube.com/watch?v=o6YVWYkkGL0&ab_channel=%E6%9C%88%E4%BA%AD%E5%85%AB%E5%85%89%E3%81%AE%E5%85%AB%E3%81%A1%E3%82%83%E3%82%93%E3%81%AD%E3%82%8B)が成り立つほど
[^jikkyo]: https://dic.nicovideo.jp/a/%E3%82%A6%E3%83%9E%E5%A8%98%E3%83%97%E3%83%AD%E5%AE%9F%E6%B3%81%E3%83%AA%E3%83%B3%E3%82%AF

私もそんな視聴者の一人です．アニメを2周ほどしたあと，Wikipediaやレース動画（やニコ動やVTuberによる実況）を一通り眺めたのですが，それだけでは満足できなくなってきました．色々調べたところ，[netkeiba](https://www.netkeiba.com/)で1986年から現在までの中央競馬の結果が無料で公開されていることがわかったので，1986年〜2020年の35年分のデータを分析し，競走馬たちに思いを馳せることにしました．

少し話は変わりますが，ウマ娘には適性という概念が存在します．これはウマ娘がどのような環境で力を発揮しやすいかを表すもので，バ場[^ba]適性（芝，ダート），距離適性（短距離，マイル，中距離，長距離），脚質適性（逃げ，先行，差し，追込）があり，それぞれアルファベットのGからAで評価されます[^indices]．こちらも合わせて分析すると考察に深みが出ると考えたため，[神ゲー攻略，ウマ娘攻略Wiki](https://kamigame.jp/umamusume/page/110667391372886023.html)から適性情報を取得させて頂くことにしました．

[^indices]: 今回は初期値を利用しました．継承により適性を向上させ，最終的にA以上の「S」に到達することも可能らしいです．
[^ba]: 有名な余談ですが，ウマ娘の世界では四本脚で走る「馬」という概念が存在しないため，「馬」の代わりに「バ」あるいは「（馬のれっかを点二つに置き換えた架空の漢字）」が使用されています．

データの取得には[Requests](https://docs.python-requests.org/en/latest/)および[Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)，データの管理には[SQLite](https://www.sqlite.org/index.html)，データの整形には[Pandas](https://pandas.pydata.org/)，データの可視化には[Plotly](https://plotly.com/python/)を用いました．可視化にデファクトスタンダードの[matplotlib](https://matplotlib.org/)を使用しなかったのは，[前回の記事](https://zenn.dev/kakeami/articles/617efe3bf9841d)と同様，インタラクティブにデータの中身を確認したかったためです．

分析結果は下記のGitHubにて公開中です．
- [`demos/`](https://github.com/kakeami/keiba-eda-public/tree/master/demos)：YouTubeのデモ動画の元データを格納
- [`figs/scatters/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters)：作図した散布図を格納
	- [`scatter_all.html`](https://github.com/kakeami/keiba-eda-public/blob/master/figs/scatters/scatter_all.html)：全データをプロットした結果
	- [`horses/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters/horses)：各競走馬に注目した散布図を格納
	- [`indices/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters/indices)：各適性がAの競走馬に注目した散布図を格納
	- [`title/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters/titles)：各タイトルで賞金を獲得した競走馬に注目した散布図を格納
	- [`vs/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters/vs)：2頭の競走馬に着目した散布図を格納
- [`notebooks/`](https://github.com/kakeami/keiba-eda-public/tree/master/notebooks)：処理内容の一部をNotebookとして格納
	- [`scatter_scrape.ipynb`](https://github.com/kakeami/keiba-eda-public/blob/master/notebooks/scatter_scrape.ipynb)：[データ取得（ウマ娘）](https://zenn.dev/kakeami/articles/04fa22797d9850#%E3%83%87%E3%83%BC%E3%82%BF%E5%8F%96%E5%BE%97%EF%BC%88%E3%82%A6%E3%83%9E%E5%A8%98%EF%BC%89)
	- [`scatter_preprocess.ipynb`](https://github.com/kakeami/keiba-eda-public/blob/master/notebooks/scatter_preprocess.ipynb)：[前処理](https://zenn.dev/kakeami/articles/04fa22797d9850#%E5%89%8D%E5%87%A6%E7%90%86)
	- [`scatter_plot.ipynb`](https://github.com/kakeami/keiba-eda-public/blob/master/notebooks/scatter_plot.ipynb)：[Plotlyによる可視化](https://zenn.dev/kakeami/articles/04fa22797d9850#plotly%E3%81%AB%E3%82%88%E3%82%8B%E5%8F%AF%E8%A6%96%E5%8C%96)

https://github.com/kakeami/keiba-eda-public

私はまだ馬券を購入したことがなく，かつウマ娘のゲームもチュートリアルすら終わらせず断念した根性なしですので，本記事には誤った情報が含まれる可能性があります．競馬ファンおよびトレーナーの皆様におかれましては，何卒ご了承いただき，可能であればご指摘頂けますと幸いです．


# データ取得（レース結果）

## 取得元

![](/images/2021-11-keiba-scatter/netkeiba.png)
*https://db.netkeiba.com/race/199306040411/*

[netkeiba](https://www.netkeiba.com/)で無料で公開されている中央競馬のレース結果を，情報解析を目的としてスクレイピングしました．定番のサイトですので，ネット上にスクリプトが溢れていましたが，細かいチューニングをすることを考えて自分で全部書きました．

## 実装

netkeibaからのスクレイピング方法は優れた解説記事が多く存在します．ただでさえ長くなってしまったので，紙面の都合上，本記事では詳細に触れません．私はオーソドックスに[Requests](https://docs.python-requests.org/en/latest/)でHTMLを取得し，[Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)で解析しました．

:::message alert
スクレイピングにあたっては，くれぐれもサーバに負荷をかけない方法で，適法に取得するようお願いいたします．
:::

## データ管理

データの規模がそこそこあるので，[SQLite](https://www.sqlite.org/index.html)を使ってデータを管理しました．レースの概要情報を管理する`race`テーブルと，着順等の結果を管理する`result`テーブルを用意しました．以下でスキーマを示します．

### `race`テーブル

|カラム|データ型|補足|
|:----|:----|:----|
|race_id|TEXT (UNIQUE)|URLの末尾|
|race_name|TEXT|レース名|
|date|TEXT|開催日．`YYYY-MM-DD`|
|place|TEXT|開催場所|
|distance|INTEGER|距離[m]|
|dart|TEXT|ダートか否か．`{'True', 'False'}`|
|dart_cond|TEXT|ダートの状況．`dart = 'False'`の場合は`Null`|
|turf|TEXT|芝か否か．`{'True', 'False'}`|
|turf_cond|TEXT|芝の状況．`turf = 'False'`の場合は`Null`|
|steeple|TEXT|障害競走か否か．`{'True', 'False'}`|
|direction|TEXT|右回りか左回りか．`{'Right', 'Left'}`|
|weather|TEXT|天候|
|start_time|TEXT|発走時刻．`HH:MM`|

![](/images/2021-11-keiba-scatter/race_schema.png)

### `result`テーブル

|カラム|データ型|補足|
|:----|:----|:----|
|race_id|TEXT|URLの末尾|
|arrival_order|INTEGER|着順．存在しない場合は`Null`|
|frame_no|INTEGER|枠番|
|horse_no|INTEGER|馬番|
|horse_name|TEXT|馬名|
|horse_id|TEXT|馬のID|
|horse_sex|TEXT|馬の性別[^gelding]|
|horse_age|INTEGER|馬の年齢|
|horse_weight|REAL|馬体重[kg]|
|horse_weight_change|REAL|馬体重の増減[kg]|
|jockey_name|TEXT|騎手名|
|jockey_weight|REAL|斤量[kg]|
|seconds_total|REAL|タイム[s]|
|seconds_3f|REAL|上りタイム[s]|
|odds|REAL|単勝オッズ|
|popularity_order|INTEGER|人気順|
|trainer_name|TEXT|調教師名|
|trainer_affiliation|TEXT|調教師の所属．`{'East', 'West'}`|
|owner_name|TEXT|馬主名|
|prize|REAL|賞金[万円]|

![](/images/2021-11-keiba-scatter/result_schema.png)

[^gelding]: 「牝」（メス）「牡」（オス）の他に「セ」（騸馬，去勢された牡馬）という概念があり，驚きました．

## 苦労したこと

### `pd.read_html()`で抜けないページがあった

`result`テーブルに該当する情報をPandasの[`read_html()`](https://pandas.pydata.org/docs/reference/api/pandas.read_html.html)で抜けると**非常に**楽だったのですが，1986年頃の古い結果ページに対しては正しく動作しませんでした．原因を調べるのが面倒でしたし，`table`タグ以外は結局解析する必要があったため，諦めて全て書き下しました．

### 着順に数字以外が入ることがあった

[降着・失格](https://www.jra.go.jp/judge/)，[競走中止](https://jra.jp/faq/pop02/2_8.html)，[出走取消](https://jra.jp/faq/pop02/2_7.html)等が原因で着順がつかず，「着順」に数字以外が入ることがある[^shikkaku]ことを知りませんでした．

![](/images/2021-11-keiba-scatter/cancel.png)
*出走取消の例：https://db.netkeiba.com/race/198601010207*

[^shikkaku]: 35年分の中央競馬のデータのうち，何らかの理由で着順に数字以外が入っていたレース数は合計で13143（約11%≒13143/119898）に上りました．結構ありますね，何か処理をミスっているかもしれません．

### 獲得賞金が1000万円以上だと`,`が入っていた

これもありがちですが，獲得賞金が1000万円を超えると`,`が含まれる（例：`4,000`）ため，そのままINTEGERにキャストしようとするとエラーが出ました．

# データ取得（ウマ娘）

## 取得元

[神ゲー攻略，ウマ娘攻略Wiki](https://kamigame.jp/umamusume/page/110667391372886023.html)から適性情報をスクレイピングしました．

![](/images/2021-11-keiba-scatter/kamigame.png)
*https://kamigame.jp/umamusume/page/110667391372886023.html*

## 実装

具体的な処理内容は下記のNotebookをご確認ください．実行環境に関しては本記事のAppendixをご参照ください．

https://github.com/kakeami/keiba-eda-public/blob/master/notebooks/scatter_scrape.ipynb

## 苦労したこと

### 新衣装等で複数回登場するウマ娘がいた

エアグルーヴとエアグルーヴ（花嫁）等，表中に複数回登場するウマ娘が数人いました[^date]．念の為検証したところ，衣装によって適性は変わらないことが確認できたため，安心しました．

[^date]: 2021年11月9日時点．

- エアグルーヴとエアグルーヴ（花嫁）
- エルコンドルパサーとエルコンドルパサー（新衣装）
- グラスワンダーとグラスワンダー（新衣装）
- ゴールドシチーとゴールドシチー（新）
- シンボリルドルフとシンボリルドルフ（新）
- スペシャルウィークとスペシャルウィーク（水着）
- スーパークリークとスーパークリーク（ハロウィン）
- トウカイテイオーとトウカイテイオー（新衣装）
- マチカネフクキタルとマチカネフクキタル（フルアーマー）
- マヤノトップガンとマヤノトップガン（花嫁）
- マルゼンスキーとマルゼンスキー（水着）
- メジロマックイーンとメジロマックイーン（新衣装）
- ライスシャワーとライスシャワー（ハロウィン）

# 前処理

散布図のプロットにあたり，下記の前処理を施しました：

- `race`テーブルに`grade`カラムと`title`カラムを追加
- プロット用の軽量なデータフレームの作成

本章ではそれぞれについて概説します．ソースコードは下記のNotebookをご確認ください．実行環境に関しては本記事のAppendixをご参照ください．

https://github.com/kakeami/keiba-eda-public/blob/master/notebooks/scatter_preprocess.ipynb

## `race`テーブルの更新

下記のように`race.race_name`からグレード情報とタイトル情報を抜き出し，それぞれ`grade`カラムと`title`カラムとして追加しました．

![](/images/2021-11-keiba-scatter/race_name.png)

### `grade`カラムの追加

競馬のレースには**グレード**と呼ばれる序列が存在しており，上から順に

- G1
- G2
- G3
- リステッド
- オープン特別
- 3勝クラス
- 2勝クラス
- 1勝クラス
- 新馬・未勝利

に分類されています[^grade][^steeple]．グレードが高くなるほど出馬条件が厳しくなり，G1レースに出馬できる競走馬はほんの一握りです．実際，1986年から2020年までの35年間で，中央競馬の[平地競走](https://www.jra.go.jp/kouza/yougo/w343.html)（障害競走以外のレース）に出場経験のある競走馬は149610頭いましたが，そのうちG3以上のレースに出場したのは：

- G1：4981頭（3.3293%）
- G2：6966頭（4.6561%）
- G3：11356頭（7.5904%）

でした[^boundary]．そもそも**中央競馬**の平地競走に出場できない競走馬も星の数ほど存在します[^urara]ので，G1ホースともなるとエリート中のエリートであることが再確認できました．

[^boundary]: 1986年から2020年までのレース結果のみから判断しているため，運悪く全盛期がこの期間にかぶらなかった競走馬（例えば2020年にメイクデビューして，2021年以降に重賞に出始めた馬）に関しては不利な集計になっています．そのあたりの境界条件を考慮すると，正確にはもう少し大きな数値になると思いますが，今回は目をつぶりました．
[^grade]: https://www.jra.go.jp/keiba/rules/class.html
[^steeple]: [平地競走](https://www.jra.go.jp/kouza/yougo/w343.html)に関する区分です．[障害競走](https://www.jra.go.jp/kouza/yougo/w256.html)の格付けは少し異なり，J.G1，J.G2，J.G3となります．https://www.jra.go.jp/kouza/yougo/w323.html
[^urara]: 高知競馬場を中心に活躍したハルウララとか．

こうした背景から，グレードの考慮は必須と判断しました．下記のような関数で`race_name`から`grade`を抜き出しました．

```python:scatter_preprocess.ipynb
def get_grade(race_name):
    """race_nameからグレード情報を抽出"""
    r = re.search('(\((J.G|G)\d+\)|\((L|G)\))', race_name)
    if r:
        grade = r.group().replace('(', '').replace(')', '')
    else:
        grade = None
    return grade
```

### `title`カラムの追加

グレードを含め，`race_name`には回数や西暦など様々な修飾語がついています．例えば有馬記念等，タイトルを軸に過去と現在の優勝馬を比較してみたかったので，`race_name`から`title`を抜き出す関数を作成しました．

```python:scatter_preprocess.ipynb
def get_title(race_name):
    """race_nameからタイトル情報を抽出"""
    # 空白文字を除外
    title = re.sub('\s$', '', race_name)
    # 第n回
    title = re.sub('第\d+回', '', title)
    # グレード：(Gn)，(J.Gn)
    title = re.sub('\((J.G|G)\d+\)', '', title)
    # リステッド競争：(L), 重賞：(G)
    title = re.sub('\((L|G)\)', '', title)
    # (n)
    title = re.sub('\(\d+\)', '', title)
    # 第n戦
    title = re.sub('第\d+戦', '', title)
    # 末尾の数字
    title = re.sub('\d$', '', title)
    # 西暦
    title = re.sub('19[8-9][0-9]|20[0-2][0-9]', '', title)
    # 略式の西暦
    title = re.sub('’([8-9][0-9]|[0-2][0-9])', '', title)
    return title
```

泥臭い正規表現を見ていただければわかるように，この作業が一番地味で一番辛かったです．

- 西暦が入っているもの（例：`2001ファイナルS`）
- 略西暦が入っているもの（例：`’88フェアウェルS`）[^muzirushi]
- ({回数})が入っているもの（例：`ドイツ騎手招待(1)`）
- 第n戦が入っているもの（例：`オールスターJ第1戦`）[^allstar]
- 末尾に回数が入っているもの（例：`ヤングJSFR中山1`）[^jsfr]

は一通りカバーしました．

[^muzirushi]: しかも無印の`フェアウェルS`も存在しました．1995年以降はレース名に略西暦がつかなくなったようです．
[^allstar]: オールスターJ第{n}戦に関しては，nに応じて出馬条件が異なるようだったので別タイトルとして集計するか悩みましたが，細かくなりすぎても扱いきれないため，同一タイトルとしました．
[^jsfr]: ヤングJSFR中山{n}に関してはnによって出馬条件が異なるようでしたが，オールスターJ第n戦と同様に同一タイトルとしました．

## 可視化用のDataFrameの準備

当初は全レースをPlotlyで散布図にしようとしていたのですが，処理が重すぎて端末がフリーズしてしまったので，グレードのついたレースのみに絞ることにしました．取り扱いやすいよう，`race`テーブルとジョインし，`all_res.csv`として中間出力しました．大まかな処理としては：

1. `race`テーブルと`result`テーブルをジョインし，`grade`および`steeple`でフィルタ
2. レース全体および上り[^agari]3ハロンの平均の速さ（`speed_total`，`speed_3f`）を計算して追加
3. ウマ娘基準の距離区分を追加

でした．以下それぞれについて解説します．

[^agari]: 読みは「あがり」です．ざっと調べたところ「上がり」という振り仮名が正しそうでしたが，netkeibaの列名「上り」を尊重して本記事では「上り」を採用しています（途中で気づいたけど書き直す気力がなかった）．

### `race`テーブルと`result`テーブルの結合

下記のクエリで任意のグレードの平地競走のレース結果を抽出しました．

```python: scatter_preprocess.ipynb
def query_results_and_races_by_grade(grade):
    """グレードでレース結果を集計するクエリ"""
    q = f'''
        SELECT *
        FROM (
            SELECT * FROM race 
            WHERE grade = '{grade}'
            AND steeple = 'False'
        ) AS race_g
        INNER JOIN result
        ON race_g.race_id = result.race_id;
    '''
    return q
```

抽出対象としたのは，`race_name`から取得可能な平地競走のグレード全て（`['G1', 'G2', 'G3', 'G', 'L']`）です．詳細は後述しますが，`G`については正確な定義が見当たらず苦労しました．`L`はリステッド競争を表し，重賞に次ぐ重要なレースと位置づけられています[^listed]．

[^listed]: https://jra.jp/kouza/yougo/w573.html

### レース全体および上り3ハロンの平均の速さを追加

散布図の縦軸・横軸として平均の速さ[km/h][^speed]を用いるため，上記のDataFrameに`speed_total`と`speed_3f`を追加しました．

```python: scatter_preprocess.ipynb
def add_average_speed_to_df(df):
    """平均の速さを計算して追加"""
    df_new = df.copy()
    df_new = df_new[~df_new['seconds_total'].isna()].\
        reset_index(drop=True)
    df_new = df_new[~df_new['seconds_3f'].isna()].\
        reset_index(drop=True)
    df_new['speed_total'] = \
        df_new['distance'] / df_new['seconds_total'] * 60 * 60 / 1000
    df_new['speed_3f'] = \
        600 / df_new['seconds_3f'] * 60 * 60 / 1000
    return df_new
```

[^speed]: デモ動画中では何気なく「速度」という表現を使っていましたが，ベクトルではなくスカラーなので「速さ」という表現が正しいです．ごめんなさい，デモを撮り直す気力はありませんでした．ご了承ください．

`speed_total`はレース全体の平均の速さを表します．`race.distance`[m]を`result.seconds_total`[s]で割り，単位変換することで時速を算出しました．`speed_total`が大きいほど，レース全体として速く走ったことを表します．

`speed_3f`は上り3ハロンの平均の速さを表します．上り3ハロンとは，レース終盤の残り600mを指し，日本競馬では特に重要な区間と見なされています[^3f]．600[m]を`result.seconds_3f`[s]で割り，単位変換することで時速を算出しました．`speed_3f`が大きいほど，ラストスパートが速かったことを表します．

横軸を`seed_total`，縦軸を`speed_3f`としてプロットすることで，競走馬の脚質適性が分布に現れることを期待していました．

[^3f]: [Wikipedia](https://ja.wikipedia.org/wiki/%E4%B8%8A%E3%81%8C%E3%82%8A_(%E7%AB%B6%E9%A6%AC))によると海外では計測しない国が多いとのことなので，日本に生まれてよかったです．

### ウマ娘基準の距離区分を追加

距離区分ごとに散布図を作図するため，上記のDataFrameに`distance_class`を追加しました．国際基準である[SMILE](https://ja.wikipedia.org/wiki/%E8%B7%9D%E9%9B%A2_(%E7%AB%B6%E9%A6%AC)#SMILE_%E5%8C%BA%E5%88%86)[^smile]を用いるか悩みましたが，ウマ娘の距離適性と比較分析するため，下記の[ウマ娘における距離区分](https://altema.jp/umamusume/kyoritekisei)を採用しました．

- 短距離: 1600m未満
- マイル: 1600m - 2000m未満
- 中距離: 2000m - 2500m未満
- 長距離: 2500m以上

[^smile]: S（Sprint: 1000m - 1300m），M（Mile: 1301m - 1899m），I（Intermediate: 1900m - 2100m），L（Long: 2101m - 2700m），E（Extended: 2701m以上）

```py: scatter_preprocess.ipynb
def get_distance_class(distance):
    """ウマ娘における距離区分を返す
    https://altema.jp/umamusume/kyoritekisei
    """
    if distance < 1600:
        return 'short'
    elif distance < 2000:
        return 'mile'
    elif distance < 2500:
        return 'intermediate'
    elif distance >= 2500:
        return 'long'
    else:
        return None
	

def add_distance_class_to_df(df):
    """距離区分をdfに追加"""
    df_new = df.copy()
    df_new['distance_class'] = \
        df_new['distance'].apply(
            lambda x: get_distance_class(x))
    return df_new
```

## 苦労したこと

### グレード`G`が何を表すかわからなかった

調査した結果，末尾に`(G)`を冠する`race_name`は下記3種類に分類できることがわかりました．

1. もともとG3だったが，天候等の影響で一時的にグレードが外れた重賞
	- [東京新聞杯](https://ja.wikipedia.org/wiki/%E6%9D%B1%E4%BA%AC%E6%96%B0%E8%81%9E%E6%9D%AF)（1995年）
	- [共同通信杯4歳S](https://ja.wikipedia.org/wiki/%E5%85%B1%E5%90%8C%E9%80%9A%E4%BF%A1%E6%9D%AF)（1998年）
2. G3の新規格付けをもらう前の新設重賞および重賞
	- [レパードステークス](https://ja.wikipedia.org/wiki/%E3%83%AC%E3%83%91%E3%83%BC%E3%83%89%E3%82%B9)（2009年，2010年）
	- [アルテミスステークス](https://ja.wikipedia.org/wiki/%E3%82%A2%E3%83%AB%E3%83%86%E3%83%9F%E3%82%B9%E3%82%B9%E3%83%86%E3%83%BC%E3%82%AF%E3%82%B9)（2012年，2013年）
	- [いちょうステークス](https://ja.wikipedia.org/wiki/%E3%82%B5%E3%82%A6%E3%82%B8%E3%82%A2%E3%83%A9%E3%83%93%E3%82%A2%E3%83%AD%E3%82%A4%E3%83%A4%E3%83%AB%E3%82%AB%E3%83%83%E3%83%97)（2014年，のちのサウジアラビアRC）
	- [サウジアラビアRC](https://ja.wikipedia.org/wiki/%E3%82%B5%E3%82%A6%E3%82%B8%E3%82%A2%E3%83%A9%E3%83%93%E3%82%A2%E3%83%AD%E3%82%A4%E3%83%A4%E3%83%AB%E3%82%AB%E3%83%83%E3%83%97)（2015年）
	- [ターコイズステークス](https://ja.wikipedia.org/wiki/%E3%82%BF%E3%83%BC%E3%82%B3%E3%82%A4%E3%82%BA%E3%82%B9%E3%83%86%E3%83%BC%E3%82%AF%E3%82%B9)（2015年，2016年）
	- [葵ステークス](https://ja.wikipedia.org/wiki/%E8%91%B5%E3%82%B9%E3%83%86%E3%83%BC%E3%82%AF%E3%82%B9)（2018年，2019年，2020年）
3. 不明
	- [セイユウ記念](https://ja.wikipedia.org/wiki/%E3%82%BB%E3%82%A4%E3%83%A6%E3%82%A6%E8%A8%98%E5%BF%B5)（1986年〜1995年）
	- [タマツバキ記念](https://ja.wikipedia.org/wiki/%E3%82%BF%E3%83%9E%E3%83%84%E3%83%90%E3%82%AD%E8%A8%98%E5%BF%B5)（1986年〜1995年）

3については，両者ともにアングロアラブの名馬セイユウとタマツバキを冠した由緒ある重賞であることまではわかったのですが，それが必要十分条件かどうか判断できませんでした．ご存知の方いらっしゃいましたら，ご教示頂けますと幸いです．

### `pd.DataFrame.to_sql(method='multi')`が動かなかった

`race`テーブルの更新にあたっては

1. `pd.DataFrame`として読み出し
2. `title`および`grade`カラムを追加
3. `pd.DataFrame`から`to_sql()`で`race`テーブルを上書き保存

というかなり行儀の悪いことをやっていたのですが，最後の`to_sql()`が遅すぎて困りました．`method='multi'`を指定すると高速化できるらしいのですが，[バグのためエラーが発生してしまいました](https://github.com/pandas-dev/pandas/issues/29921)．結局，[SQLAlchemy](https://www.sqlalchemy.org/)経由で書き込むことにしました．

```py: scatter_preprocess.ipynb
import sqlalchemy as sa

engine = sa.create_engine(
    f'sqlite:///{path_db}', echo=False)
df_race.to_sql(
    TN_RACE, engine, if_exists='replace',
    method='multi', chunksize=5000)
```

# Plotlyによる可視化

## 作ったもの

前章で作成した可視化用データを元に，次のような散布図を作成しました．まずはデモをご覧ください．字幕で解説を入れておりますので，「字幕ON」の設定をおすすめします．

https://youtu.be/fBI9aKwAhfM

https://youtu.be/tSBXADm7pv8

まず**距離区分**に応じて，散布図を4つ描画しました．これにより各競走馬の**距離適性**，および距離区分における全体傾向を明らかにすることが狙いです．

各散布図の横軸として**レース全体の平均の速さ**，縦軸として**上り3ハロンの平均の速さ**を設定しました．直感的には，右側に位置する競走馬ほど速く到着しており，上側に位置する競走馬ほどラストスパートが速かったことを示します．これにより各競走馬の**脚質適性**を見ることが狙いです．

サンプルの色は**獲得賞金**を表します．獲得賞金が高いほど明るい黄色になります．どのような距離適性・脚質適性の競走馬が稼いでいるか直感的に眺めることが狙いでした．もちろん，日本競馬は高速化しているため，35年前のタイムと現在のタイムを一概に比較することはできません．また，同じ距離区分内でもレースごとに距離が異なるため，各サンプル同士をフェアに比較することはできません[^comparison]．あくまでも全体傾向をぼんやりと眺める用途で作図しました．

[^comparison]: たとえ距離が同じであっても，そのレースによって対抗馬や体調や天候は異なります．原理的にフェアな比較は不可能という前提で見て頂けると幸いです．

## Plotlyとは

[Plotly](https://plotly.com/graphing-libraries/)とは，インタラクティブなグラフや地図を描画するためのライブラリ群です．Python，R，Julia，Javascript，F#，MATLAB等様々な言語へのインターフェースが提供されています．

Pythonのデファクトである[matplotlib](https://matplotlib.org/)との大きな違いは，手軽にインタラクティブなグラフを作成できる点です．私は仕事柄，入出力データについて社内外で議論することが多いのですが，分布を俯瞰して見ながら，特定のサンプルに焦点をあてる[^outlier]ことが可能なPlotlyは非常に便利です[^excel]．

[^outlier]: この外れ値なんだっけ？みたいなときにすごく便利です．

[^excel]: エクセルでも同じことができるのですが，データの規模が大きくなってくると動作がもたついてくること，選択的にデータを秘匿することができないこと，またマクロを組まなければ自動化が難しいこと，（そして私のエクセル力が極端に低いこと，）からPlotlyを使うことが多いです．

[公式ドキュメント](https://plotly.com/python/)で実装例が豊富なのでやりたいことはたいてい見つかりますし，近年[Plotly Express](https://plotly.com/python/plotly-express/)という高レベルインターフェースが追加されたため，導入のハードルは低くなっています．

Python用Plotlyは下記のコマンドでインストールできます．

```sh
pip install plotly
```

## 実装

下記のNotebookをご参照ください．実行環境に関しては本記事のAppendixをご参照ください．

https://github.com/kakeami/keiba-eda-public/blob/master/notebooks/scatter_plot.ipynb

大きく2種類の散布図を作成しました．それぞれについて概説します．

- 全データの散布図（例：[`scatter_all.html`](https://github.com/kakeami/keiba-eda-public/blob/master/figs/scatters/scatter_all.html)）
- 全データの上に，注目したいデータを重ねた散布図（例：[`scatter_トウカイテイオー.html`](https://github.com/kakeami/keiba-eda-public/blob/master/figs/scatters/horses/scatter_%E3%83%88%E3%82%A6%E3%82%AB%E3%82%A4%E3%83%86%E3%82%A4%E3%82%AA%E3%83%BC.html)）

### 全データの散布図（例：[`scatter_all.html`](https://github.com/kakeami/keiba-eda-public/blob/master/figs/scatters/scatter_all.html)）

[`scatter_all.html`](https://github.com/kakeami/keiba-eda-public/blob/master/figs/scatters/scatter_all.html)を作成するために，下記の`subplot_scatter_by_distance_class()`を定義しました．

```python: scatter_plot.ipynb
import plotly.express as px
from plotly.subplots import make_subplots
import plotly.graph_objects as go


def subplots_scatter_by_distance_class(
        df, color_col='prize', color_title='獲得賞金', asc=True):
    """距離区分ごとにsubplotでscatterを描画"""
    fig = make_subplots(
        rows=2, cols=2, subplot_titles=SUBPLOT_TITLES)
    x_min, x_max = get_min_and_max_of_col(df, 'speed_total')
    y_min, y_max = get_min_and_max_of_col(df, 'speed_3f')
    for i, dc in enumerate(DISTANCE_CLASSES):
        df_tmp = make_df_for_plot(df, dc, color_col, asc)
        add_line_y_equal_x_to_fig(fig, i)
        add_scatter_trace_to_fig(
            fig, x=df_tmp['speed_total'], y=df_tmp['speed_3f'],
            color=df_tmp[color_col], text=df_tmp['hover_text'],
            name=dc, i=i)
    update_colorbar_of_fig(fig, color_title)
    update_axis_ranges_of_fig(
        fig, x_min=x_min, x_max=x_max, 
        y_min=y_min, y_max=y_max)
    update_axis_titles_of_fig(fig)
    return fig
```

Plotlyには，subplotを実現する方法がいくつかあります．おそらく一番手軽なのは[Plotly Expressの`facet`オプション](https://plotly.com/python/facet-plots/)を使う方法だと思いますが，細かい制御ができず[^facet]諦めました．代わりに，今回は[`make_subplots`](https://plotly.com/python-api-reference/generated/plotly.subplots.make_subplots.html)メソッドを利用しました．引数として`rows`で行数，`cols`で列数，`subplot_titles`で各サブプロット名を定義することができます．

[^facet]: subplotの順序を制御するのが難しかったり，subplotを跨いだcolorbarの表示が崩れたりしました．

上記で作成した`fig`オブジェクトに，独自に作成した`add_scatter_trace_to_fig()`を使って，合計4つの`DISTANCE_CLASSES`ごとに`trace`オブジェクトを追加しました[^fig_add_trace]．

[^fig_add_trace]: 最初に`fig`オブジェクトを作って，それに`add_trace`でグラフを追加していくのは，[Plotlyによる作図のよくあるパターンの一つ](https://ai-research-collection.com/add_traceupdate_layout/)です．Plotly Expressはその辺すら簡略化してくれますが．

```python: scatter_plot.ipynb
def add_scatter_trace_to_fig(
        fig, x, y, color, text, name, i,
        opacity=1., symbol='circle', size=10,
        hover=True, linecolor='White'):
    """figに対しscatterを追加"""
    fig.add_trace(
        go.Scatter(
            x=x,
            y=y,
            mode='markers',
            marker_symbol=symbol,
            marker_size=size,
            opacity=opacity,
            hoverinfo='text' if hover else 'skip',
            marker={
                'color': color,
                'coloraxis':'coloraxis',
                'line':{
                    'color': linecolor,
                    'width': 1},
            },
            text=text,
            hovertemplate='%{text}' if hover else None,
            name=name,
        ),
    i//2+1, i%2+1)
```

`fig.add_trace()`内では，所望のグラフオブジェクトを指定して`fig`に追加できます．ここでは`go.Scatter()`で散布図のオブジェクトを指定しました．

`marker_symbol`（マーカーの形），`marker_size`（マーカーの大きさ），`opacity`（マーカーの透明度），`hover`（ホバー時にテキスト等を表示するかどうか）をオプション引数としたのは，次節で使用するためです．

ホバー時に表示するテキストは`hovertemplate`で制御できます．詳細は後述します．

ちなみに，最後の`i//2+1, i%2+1`は散布図の行番号と列番号を表します．

### 注目したいデータを重ねた散布図（例：[`scatter_トウカイテイオー.html`](https://github.com/kakeami/keiba-eda-public/blob/master/figs/scatters/horses/scatter_%E3%83%88%E3%82%A6%E3%82%AB%E3%82%A4%E3%83%86%E3%82%A4%E3%82%AA%E3%83%BC.html)）

冒頭のデモでお見せしたトウカイテイオーのように，全体の散布図を背景として示しつつ，注目したいデータを重ねて描画した散布図を作成しました．

- [`figs/scatters/horses/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters/horses)：各競走馬に注目した散布図
- [`figs/scatters/indices/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters/indices)：各適性がAの競走馬に注目した散布図
- [`figs/scatters/title/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters/titles)：各タイトルで賞金を獲得した競走馬に注目した散布図

これらは全て，前記の`subplots_scatter_by_distance_class()`を少しだけ変更した`subplots_two_scatters_by_distance_class()`により作図しました．

```python: scatter_plot.ipynb
def subplots_two_scatters_by_distance_class(
        df, df_star, color_col='prize', color_title='獲得賞金', asc=True):
    """距離区分ごとにsubplotでscatterを描画
    ただしdf_starの結果は☆でプロット"""
    fig = make_subplots(
        rows=2, cols=2, subplot_titles=SUBPLOT_TITLES)
    x_min, x_max = get_min_and_max_of_col(df, 'speed_total')
    y_min, y_max = get_min_and_max_of_col(df, 'speed_3f')
    for i, dc in enumerate(DISTANCE_CLASSES):
        df_tmp = make_df_for_plot(df, dc, color_col, asc)
        df_star_tmp = make_df_for_plot(df_star, dc, color_col, asc)
        add_line_y_equal_x_to_fig(fig, i)
        # 背景表示用
        add_scatter_trace_to_fig(
            fig, x=df_tmp['speed_total'], y=df_tmp['speed_3f'],
            color=df_tmp[color_col], text=df_tmp['hover_text'],
            name=dc, i=i, opacity=0.3, hover=False)
        # 注目したいデータ用
        add_scatter_trace_to_fig(
            fig, x=df_star_tmp['speed_total'], y=df_star_tmp['speed_3f'],
            color=df_star_tmp[color_col], text=df_star_tmp['hover_text'],
            name=dc, i=i, symbol='star', size=25)
    update_colorbar_of_fig(fig, color_title)
    update_axis_ranges_of_fig(
        fig, x_min=x_min, x_max=x_max, 
        y_min=y_min, y_max=y_max)
    update_axis_titles_of_fig(fig)
    return fig
```

主な違いは下記です：

- `df`に加え，重ねて表示したいデータを格納した`df_star`を渡す
- `DISTANCE_CLASSES`毎に**2回ずつ**`add_scatter_trace_to_fig()`を呼び出す
    1. `df`を半透明（`opacity=0.3`）で，ホバー時に反応しないように（`hover=False`）追加
    2. `df_star`を星型マーカー（`symbol='star'`）で，大きめに（`size=25`）で追加

この`df_star`を競走馬ごと，タイトルごと，適性毎に作成することで，様々なパターンの散布図を作成しました．

ちなみに[`figs/scatters/vs/`](https://github.com/kakeami/keiba-eda-public/tree/master/figs/scatters/vs)では**2頭の競走馬**に注目した散布図（一頭目を星型マーカー，二頭目を三角型マーカーで表示）を作成していますが，基本的な考え方は同じです．

## 苦労したこと

### 見栄えを整えるために獲得賞金でソートした

高額賞金を獲得した競走馬は一握りでしたので，何も考えずに散布図を描画すると画面が真っ黒になってしまいました．そこで，`prize`で昇順に並び替えてからプロットすることで，明るい黄色のマーカーが前面に描画されるよう工夫しました．

### `hovertemplate`にHTMLを直打ちした

Plotlyでは，マーカー上にカーソルをホバーしたときに表示する情報を制御できます．Plotly Expressは[`hover_data`, `hover_name`等のオプションで手軽に指定できる](https://plotly.com/python/hover-text-and-formatting/#customizing-hover-text-with-plotly-express)機能を提供しているのですが，今回はPlotly Expressを採用しなかったため，より低レベルなインターフェースである[`hovertemplate`](https://plotly.com/python/hover-text-and-formatting/#customizing-hover-text-with-a-hovertemplate)を使いました．

```python: scatter_plot.ipynb
def add_scatter_trace_to_fig(
        fig, x, y, color, text, name, i,
        opacity=1., symbol='circle', size=10,
        hover=True, linecolor='White'):
    """figに対しscatterを追加（一部省略）"""
    fig.add_trace(
        go.Scatter(
            # 略
            text=text,
            hovertemplate='%{text}' if hover else None,
            # 略
        ),
    i//2+1, i%2+1)
```

`hovertemplate`ではグラフオブジェクト内の変数を自由に展開できます．今回は`text`をそのまま展開していますが，`x`（X軸に対応する値），`y`（Y軸に対応する値），`color`（カラースケールに対応する値）等ももちろん呼び出せます．HTMLタグも有効です．

今回の実装では，下記のような`hover_text`カラムをDataFrameに追加しておき，`add_scatter_trace_to_fig()`に`text`として渡しました．

```python: scatter_plot.ipynb
def make_hover_text(
        horse_name, horse_age,
        race_name, turf, dart, date, distance,
        seconds_total, seconds_3f,
        speed_total, speed_3f,
        arrival_order, prize):

    if turf:
        race_type = '芝'
    elif dart:
        race_type = 'ダート'
    else:
        race_type = ''

    text = f'''<b>{horse_name}</b> ({horse_age}歳) <br><br>
    レース：{race_name} ({date}, {race_type}, {distance}m)<br>
    タイム： {seconds_total} 秒 (上り: {seconds_3f} 秒) <br>
    平均の速さ: {speed_total:.4} km/h （上り: {speed_3f:.4} km/h） <br>
    着順：{arrival_order}<br>
    賞金：{prize}万円
    '''
    return text


def add_hover_text_to_df(df):
    """hovertemplate用のhover_textカラムを追加"""
    df_new = df.copy()
    df_new['hover_text'] = \
        df_new[
            ['horse_name', 'horse_age', 
             'race_name', 'turf', 'dart', 'date', 'distance', 
             'seconds_total', 'seconds_3f',
             'speed_total', 'speed_3f',
             'arrival_order', 'prize']].apply(
        lambda x: make_hover_text(*x), axis=1)
    return df_new
```

### サイズを圧縮するために不要なサンプルを削除した

何も考えずに出力すると，各散布図のファイルサイズは31MB程度になってしまいました．少しでも圧縮するため，マーカーの位置が重複する場合は最も獲得賞金が大きいもの（獲得賞金が同じ場合は開催日が新しいもの）以外を削除することにしました．

```python: scatter_plot.ipynb
def drop_dup_for_plot(
        df, cols_dup=['speed_total', 'speed_3f'],
        cols_sort=['prize', 'date'], asc=False):
    """cols_dupが重複したデータを削除
    その際， cols_sortでasc順にソートしたあとで重複を削除されることに注意．
    デフォルト設定ではprizeおよびyearが大きいものが残る形になる．
    """
    df_new = df.copy()
    df_new = df_new.sort_values(cols_sort, ascending=asc)
    df_new = df_new.drop_duplicates(subset=cols_dup)
    return df_new
```

結果的にファイルサイズは15MB程度[^size]まで小さくなりました．これでも十分巨大ですが…．

[^size]: タイムの精度が小数点以下1桁で，走行距離のバリエーションも多くないので，もっと圧縮できると思っていましたが…．

# 考察

## 全体像

## ウマ娘

### 対象

### トウカイテイオー

### ツインターボ

### その他

その他のデータは下記にアップロードしてあります．

## ウマ娘の比較

### 対象

### トウカイテイオー vs メジロマックイーン

### その他

## レース

### 対象

### 有馬記念

### その他

## 適性

### 対象

### 逃げ"A"

### 追込み"A"

### その他

# 今後の課題

## 上りタイムクラスターの原因特定

なんとなく開催年にヒントがありそうですが，現時点では原因を特定できていません．単純に高速化で片付けて良い問題なのかどうか…．

## 統計的仮説検定

上記の課題とも絡みますが，競馬界で囁かれている定説について，数理統計的な観点での裏付けに挑戦したいです．

- 日本競馬の高速化
- 先行脚質のある馬は外枠で不利
- 「芦毛は走らない」[^ashige]

きちんと調査すればどこかで誰かが検証している気がしますが，統計的仮説検定の練習としてやってみたいです．

[^ashige]: タマモクロスやオグリキャップが登場する前は，「芦毛の馬は走らない」が定説だったそうです．[シンデレラグレイ](https://ynjn.jp/app/title/1180)最高！

## ダッシュボード化

そもそもこういったデータは[クソデカHTMLとしてGitHubで公開する](https://github.com/kakeami/keiba-eda-public)のではなく，ダッシュボード化して，誰でも任意の切り口で集計・分析できるようにしてあると親切です．それこそ[Plotly Dash](https://dash.plotly.com/)をサーバー上に構築しても良いですし，[Google Data Studio](https://developers.google.com/datastudio?hl=ja)等でサーバーレスに実現しても良いでしょう．

ただ，これを突き詰めていくと[netkeiba スーパープレミアムコース](https://regist.netkeiba.com/?pid=premium)と競合しそう[^omoiagari]で…．本家に迷惑をかけるのは本意でないので，対応を検討中です．

[^omoiagari]: 思い上がりも甚だしいですが．

# おわりに

[ジョージア工科大学の合格通知](https://zenn.dev/kakeami/articles/617efe3bf9841d)を頂いてからおよそ1週間後に，軽度の[頚椎椎間板ヘルニア](http://www.neurospine.jp/original24.html)を診断され，趣味だったジョギングを止めなければならなくなりました．かなり落ち込んでいたのですが，ウマ娘や競走馬の走りを見て，大きく勇気づけられました．少しでもその魅力が伝わるよう，今回は稚拙ながら記事を書かせて頂きました．

分析はデータがないことには始まりません．35年以上前から，良質なデータを取得し続けてくださった競馬関係者の皆様に感謝申し上げます．いつも面白いレースをありがとうございます．また，ウマ娘関係者の皆様，このような素晴らしいコンテンツを発見するきっかけを与えてくださり，ありがとうございます．今後もウマ娘を応援し続けます！

なお，本記事の企画構想，データ取得・加工・可視化の作業，そして執筆完了まで合計でxx時間xx分xx秒[^time-track]かかりました．

[^time-track]: スクレイピング等の自動処理中など，特に手を動かしていない時間は含まれません．

# Appendix：実行環境の構築

本記事の処理の一部をNotebookにまとめ，下記で公開しています．

https://github.com/kakeami/keiba-eda-public/tree/master/notebooks

Python等の環境は[notebooks/Dockerfile](https://github.com/kakeami/keiba-eda-public/blob/master/notebooks/Dockerfile)に記載の通り[`jupyter/datascience-notebook:2021-11-04`](https://hub.docker.com/r/jupyter/datascience-notebook/)イメージに準拠しています．

```Dockerfile: Dockerfile
FROM jupyter/datascience-notebook:2021-11-04
MAINTAINER Kakeami

RUN pip install jupyterlab && \
    jupyter serverextension enable --py jupyterlab
```

Jupyter labの環境を立ち上げる際は

```bash
sudo docker-compose up -d
```

した上で`localhost:9999`[^port]に接続し，パスワード`twinturbo`を入力してください．ただし，[`docker-compose.yml`](https://github.com/kakeami/keiba-eda-public/blob/master/docker-compose.yml)の

[^port]: ローカルのJupyterと衝突しないよう9999番にフォワーディングしています．こちらも[`docker-compose.yml`](https://github.com/kakeami/keiba-eda-public/blob/master/docker-compose.yml)から適宜変更してください．

```yml: docker-compose.yml
version: '3'
services:
  jupyterlab:
    build:
      context: ./notebooks/
    volumes:
      - "./:/home/jovyan/work"
    user: root
    ports:
      - "9999:8888"
    environment:
      NB_UID: 1000
      NB_GID: 1000
      GRANT_SUDO: "yes"
    command: start.sh jupyter lab --NotebookApp.password="sha1:d382f585c867:09d7af4613fb433cbebf2606a5ede52c705f5d45"
```
最終行を適宜修正し，必ずパスワードを変更してください．

