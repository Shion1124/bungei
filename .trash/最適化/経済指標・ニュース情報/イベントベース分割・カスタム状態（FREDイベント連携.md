## イベントベース分割・カスタム状態

### 【Step 2-4】イベントベース分割・カスタム状態（FREDイベント連携実装例追記）

下記から米国経済指標の過去の記録を取得：

  FRED API（経済指標：重要評価指標）：61b5b7f1da41116cc2235e2ac825f948
  https://fredaccount.stlouisfed.org/apikey

NewsAPI.org（ニュース情報、経済指標のその他のイベント等）
6fc66ae95ba6410a838423fd15fba0e9
https://newsapi.org/

#### 1. 概要

定期的に更新しなければならない：/Users/yoshihisashinzaki/Hyperliqud_bot(1)/scripts/fetch_fred_events.pyの実行
- `data/fred_major_events.csv`（主要米国経済指標の発表日リスト）を利用し、

バックテスト対象データに「イベント区間」ラベル（例：event_state）を自動付与する。

#### 2. 実装サンプル:script/ml_pipline.pyに記述

```python

import pandas as pd

  

# 1. イベントリストの読み込み

event_dates = pd.read_csv("data/fred_major_events.csv")

event_dates["date"] = pd.to_datetime(event_dates["date"])

  

# 2. バックテスト対象データ（例：df）にイベント区間ラベルを付与

# df['timestamp']はdatetime型であること

df['event_state'] = 'none'

for event_time in event_dates["date"]:

mask = (df['timestamp'] >= event_time - pd.Timedelta(hours=1)) & (df['timestamp'] <= event_time + pd.Timedelta(hours=1))

df.loc[mask, 'event_state'] = 'event'

  

# 3. 必要に応じて指標名ごとのラベルも付与可能

for _, row in event_dates.iterrows():

event_time = row["date"]

event_name = row["event_name"]

mask = (df['timestamp'] >= event_time - pd.Timedelta(hours=1)) & (df['timestamp'] <= event_time + pd.Timedelta(hours=1))

df.loc[mask, 'event_state'] = event_name # 指標名でラベル付与

```

#### 3. 活用例

- `event_state`が`event`または特定指標名の区間で、

- モデル精度・損益・特徴量分布を通常区間と比較

- イベント区間のみのバックテストや可視化

- イベント区間の長さ（前後何時間）や重複時の優先順位は要件に応じて調整

#### 4. 運用・拡張

- `data/fred_major_events.csv`を定期的に更新すれば、最新のイベント区間にも自動対応

- 他のイベント（例：FOMC声明、地政学リスク等）も同様のCSVで拡張可能

##### ● 定期更新しない場合の影響

- 新しい経済指標イベント（例：直近・今後の雇用統計やCPIなど）が反映されません。

- そのため、最新のイベント区間でラベル付与ができず、バックテストや運用時の分析・アラートに抜け漏れが生じます。

- 過去データのみで運用している場合は問題ありませんが、リアルタイムや直近データを扱う場合は必ず定期更新が必要です。

##### ● 夏時間・冬時間のズレについて

- FREDのデータは「日付（date）」のみで「時刻（hour）」情報は含まれていません。

- 米国の経済指標発表は「夏時間（DST）」と「冬時間」で日本時間換算時に1時間のズレが生じます。

- 定期更新しない場合、夏時間・冬時間の切り替わりに気づかず、イベント区間の推定時刻が1時間ずれるリスクがあります。

- 理想的には、公式カレンダーやAPIで「時刻」まで取得し、夏時間・冬時間を考慮して区間を設定するのがベストです。

#####  他イベントCSVのサンプル作成
- - fetch_fred_events.py とは別に、他イベント用のCSV（例：data/other_events.csv）を手動または自動で作成します。
- - FRED APIで取得できないイベント（FOMC声明、地政学リスク、要人発言など）は、専用のCSVファイルで管理するのが一般的です。
- fetchfetch_fred_event.py（CSVファイル取得）
- fred_major_event.csv


#####  夏時間・冬時間を考慮したイベント時刻推定例
- - FRED APIは「日付」しか返さないため、時刻（hour）は自分で推定・補正する必要があります。
- - 米国経済指標の発表時刻は「夏時間（DST）」と「標準時間」で日本時間換算時に1時間ズレます。
- 夏時間（DST）の開始:毎年 3月の第2日曜日 に始まります。この日、午前2時に時計が午前3時に進められます。
- 標準時間（Standard Time）への移行:毎年 11月の第1日曜日 に戻ります。この日、午前2時に時計が午前1時に戻されます。
- つまり、年に2回、時間の切り替えが行われます。


参考：
##### ニュースデータの自動取得・CSV化
[[金融データプロバイダー：ニュース情報の取得]]
- NewsAPI.org (金融関連ニュースにも強い)：推奨



[[ニュース情報の使い方（Optunaを使ったモデル最適化時）]]
- fetch_news_event.py（CSVファイル取得）

[[ NewsAPI.orgで取得できそうな情報と目的に対する適合性]]
- ニュース情報を取り入れたOptunaでモデル最適化の方法

#### 5. 拡張済み（ml_pipline.pyに実装済み）
- アラート通知に event_state を含める
	- 目的：アラート（例：特徴量欠損、論理エラー等）が発生した際、その時点の event_state（どのイベント区間か） を通知メッセージに含めることで、「どの経済指標イベント時に異常が多いか」を即座に把握できるようにする。

- イベント区間ごとの異常発生率の自動集計・可視化
	- 目的：event_stateごとに異常発生件数・発生率を自動集計し、可視化（棒グラフ等）で「どのイベントで異常が多いか」を直感的に把握する
		- ポイント：
			- event_stateごとに異常件数・発生率を自動集計
			- 棒グラフで「どのイベントで異常が多いか」を一目で把握
			- logs/event_anomaly_rate.pngとして保存

- Prefectフローとの連携も確認
	- main_flowの最後で、特徴量データを一時保存し、plot_event_anomaly_rateを自動実行するようにしました。

##### 主な追加・統合内容

1. update_event_data関数

　- fetch_news_events.py・fetch_fred_events.pyをサブプロセスで自動実行し、

　　Optuna最適化やバッチ処理の冒頭で最新データを取得

2. 効率的なデータ読み込み関数

　- load_recent_news・load_recent_eventsで、常に最新30日分のみをpandasで高速読み込み

3. 特徴量・event_state付与関数

　- add_news_event_state：ニュースイベント区間ラベル付与

　- add_event_state_by_name：経済指標イベント区間ラベル付与

　- add_news_features：ニュース件数などを特徴量として追加

4. main_flowの統合

　- フロー冒頭でupdate_event_data()を呼び出し、

　- 特徴量生成後に上記関数でevent_state・特徴量を自動付与


###### これで実現できること

- 常に最新のニュース・経済指標データを自動取得・反映

- 最適化対象期間だけを効率的に読み込み、パフォーマンスを維持

- event_stateやニュース特徴量を活用した高精度な最適化・分析が自動化

###### 夏時間、標準時間の指定
- fetch_fred_events.pyでJST時刻列を自動付与（夏時間補正）
- - event_time_jst列をCSVに自動出力するよう修正済みです。
- ml_pipeline.pyでevent_state付与時にJST時刻を参照
	- - add_event_state_by_name等で、JST時刻列（event_time_jst）が存在すればそれを優先してevent_stateを付与するよう修正済みです。
	- これにより、夏時間補正済みの正確なイベント時刻でevent_stateが自動付与されます。


次の作業予定：

- 全てのドキュメント化および知見管理システムへの反映



