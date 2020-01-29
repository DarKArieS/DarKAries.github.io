---
title: 'ML Study Jam1:使用 BigQueryML預測計程車費率' 
date: 2019-05-13 11:37:56
tags: 
- 'Google Could Platform'
- 'SQL'
categories: 'Google Could Platform'
---

[ML Study Jam](https://events.withgoogle.com/ml-study-jam-basic-tw/)

[Qwiklabs連結](https://google.qwiklabs.com/focuses/1797?parent=catalog)

前陣子~~從入門到放棄~~自學機器學習時就有耳聞 Google 推出了許多雲端工具，例如本篇會使用的 BigQuery 等等。

以前自學的時候看到類似的雲端工具時，總會覺得不知該從何處下手才好，最後便下意識地跳過:sweat_smile:

而 Google 最近為了推廣自家的 GCP 雲端平台，發起了 ML Study Jam ，提供免費的一個月線上學習平台 Qwiklabs 訂閱。這次主打的 Qwiklabs 項目會手把手教你如何使用 GCP 上的各項功能，不會涉及到太多機器學習技術探討。 Qwiklabs 會提供一個有時間及功能限制的 GCP 帳號供你練習。在教學中會有一些小關卡，當你在該帳號上完成時就可以過關。

這邊不會介紹 Qwiklabs 與 GCP 介面的基礎使用方式，還沒試過的同學可以先[來這裡看看](https://google.qwiklabs.com/quests/23)，第一個介紹 Qwiklabs 與 GCP 的 Hands-on lab 是免費的。

<!-- more -->

大部分圖片來自 Qwiklabs 。

# Overview

[BigQuery](https://cloud.google.com/bigquery/)是 Google 推出的無伺服器企業資料倉儲服務，同時內建 [BigQuery MachineLearning](https://cloud.google.com/bigquery/docs/bigqueryml-analyst-start) 功能，可以直接在雲端上將資料用於 training 、 evaluation 、 prediction 等等。

這篇文章所參考的 Lab 將會使用 BigQuery 中的公開數據集—— New york city taxi cab 2015 來建立並評估模型，並用來預測計程車費。

# Explore dataset

在 GCP Console 中打開 BigQuery 頁面，長得像這個樣子：
左側選單找不到的同學可以從上方的搜尋欄輸入 BigQuery 。

<img src="https://cdn.qwiklabs.com/vd3QNHGB4BAyHjAvOHw9qD0iqCPNaHAzY657z%2FGWLtY%3D" width="80%">

根據教學文件， BigQuery 使用相容 SQL 2011 standard 的 SQL 語法來操作數據，詳細的訊息可以參考[文件](https://cloud.google.com/bigquery/docs/reference/standard-sql/)。對於之前沒用過 SQL 的人(我)來說，這個部分我覺得並不是太好上手，花了好幾天的時間研究。

在 Query editor 中輸入 SQL 指令後按下 Run 按鈕，系統會按照 SQL 指令將資料撈出來，並透過下方表格呈現結果。也可以試著用看看 Data Studio ，畫成圖表。

先來看看數據集中的數據長什麼樣子，像是每個月的 trips 總數量：
```sql
#standardSQL
SELECT
  TIMESTAMP_TRUNC(pickup_datetime,
    MONTH) month,
  COUNT(*) trips
FROM
  `bigquery-public-data.new_york.tlc_yellow_trips_2015`
GROUP BY
  1
ORDER BY
  1
```

從程式碼中可以看出數據是來自 `bigquery-public-data.new_york.tlc_yellow_trips_2015` ，並使用 [TIMESTAMP_TRUNC](https://cloud.google.com/bigquery/docs/reference/standard-sql/timestamp_functions) 將時間戳轉為月份。

要瀏覽數據集的細節需要使用擁有 BigQuery 資源的帳號(這樣才可以計費啊XD)，例如 Qwiklabs 提供的帳號。這次用的數據集細節可以在 [Dataset Preview](https://bigquery.cloud.google.com/table/nyc-tlc:yellow.trips?tab=preview) 找到

想知道 BigQuery 的其他的公開數據集可以來 [BigQuery Public-Data](https://cloud.google.com/bigquery/public-data/) 找找。

等 Query 完成後，下方顯示結果為：

<img src="https://cdn.qwiklabs.com/i3PmHY7jPV1XuAIql%2FDlkUHWFWuPJLcW1VECFP9P%2BuI%3D" width="30%">

每個月都有大約一千萬多筆的資料。

再來看看紐約的計程車司機在一天中不同的時間點上，跑得有多快：

```sql
#standardSQL
SELECT
  EXTRACT(HOUR
	FROM
		pickup_datetime) hour,
  ROUND(AVG(trip_distance / TIMESTAMP_DIFF(dropoff_datetime,
        pickup_datetime,
        SECOND))*3600, 1) speed
FROM
  `bigquery-public-data.new_york.tlc_yellow_trips_2015`
WHERE
  trip_distance > 0
  AND fare_amount/trip_distance BETWEEN 2
  AND 10
  AND dropoff_datetime > pickup_datetime
GROUP BY
  1
ORDER BY
  1
```

這段程式藉由 `trip_distance` 、 `pickup_datetime` 、 `dropoff_datetime` 這三個欄位來計算出計程車的平均速率，並使用 ROUND 函數換成時速。
最後使用 `GROUP BY 1` ，根據第一欄 hour 來分群。

結果像是這個樣子，半夜凌晨大夥兒傾向飆車。

<img src="https://cdn.qwiklabs.com/%2BF8Mj8HqPM9b8%2Fna1JYPy66xTKDcUu%2BQs1oh5Gy07A4%3D" width="10%">

# Training dataset

現在，我們想要試試以下欄位是否是好的 feature ，決定來 train 一發看看：

- Tolls Amount
- Fare Amount
- Hour of Day
- Pick up address
- Drop off address
- Number of passengers

我們必須把它們撈出來，並且做一些數據篩選，完整程式碼如下:

```sql
#standardSQL
WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    pickup_longitude AS pickuplon,
    pickup_latitude AS pickuplat,
    dropoff_longitude AS dropofflon,
    dropoff_latitude AS dropofflat,
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
  WHERE
    trip_distance > 0 AND fare_amount > 0
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.TRAIN
  )

  SELECT *
  FROM taxitrips
```

這一段使用 `WITH...AS...` 開了三個 SubQuery 。
 `params` 與 `daynames` 分別將 `TRAIN` 跟 星期 等「常數」定義好，而最後輸出的查詢則是由 `taxitrips` 負責。

`params` ：

<img src="https://i.imgur.com/mETjvTd.png" width="20%">

`daynames` ：

<img src="https://i.imgur.com/5asAu7S.png" width="15%">

`taxitrips` ：

<img src="https://cdn.qwiklabs.com/9%2FORyElhKMLalupP%2FuG%2BZqE%2FTjLX4XYCXnvsEmGLang%3D" width="70%">

在 `taxitrips` 這個查詢中，藉著 `WHERE` 來篩選資料：
`trip_distance > 0 AND fare_amount > 0`

並以以下條件來區分 Training Set 與 Validation Set :

```sql
MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.TRAIN
```

這裡使用了一個特別的 [FARM_FINGERPRINT](https://cloud.google.com/bigquery/docs/reference/standard-sql/hash_functions) Hash 函數，可以把字串或位元組轉成長整數。

這個條件式運作方式為：
日期轉為字串 -> 轉為長整數的絕對值 -> 取除1000的餘數 -> 判斷答案是否為params.TRAIN，也就是1。
根據教學文件，可以選出 1/1000 的數據來作為訓練集。

# Train model

## Create a BigQuery dataset to store models

在訓練之前，我們必須要開一個可以存放訓練好的 model 的空間。

在左邊的 BigQuery Resource 選單中，選擇最下方的 ProjectID ，使用 Qwiklabs 帳號的話看起來會像 `qwiklabs-gcp-xxx` 。

點選右下邊的 CREATE DATSET ，並取個名字，就叫 taxi 好了。

<img src="https://cdn.qwiklabs.com/QGOFCQMb3UNnOf2dByXcmcH7%2BX6xwnoQFX0Fdo7fRLU%3D" width="70%">

好了之後點 Create dataset 就可以了。

## Create and train a model

先上 Query 程式碼：

```sql
#standardSQL
CREATE or REPLACE MODEL taxi.taxifare_model
OPTIONS
  (model_type='linear_reg', labels=['total_fare']) AS

-- 以下跟上一段的程式碼相同

WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    pickup_longitude AS pickuplon,
    pickup_latitude AS pickuplat,
    dropoff_longitude AS dropofflon,
    dropoff_latitude AS dropofflat,
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
  WHERE
    trip_distance > 0 AND fare_amount > 0
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.TRAIN
  )

  SELECT *
  FROM taxitrips
```

在這段程式碼中，我們創造了一個 model ： taxi.taxifare_model ，也就是隸屬於剛創建的 taxi 空間之下。
OPTIONS 指定了 model 的訓練方式以及想預測的目標。
BigQuery ML 目前只提供三種機器學習演算法，分別是 `linear_reg` 、 `logistic_reg` 以及 `kmeans` 。
在這個範例中我們預測的目標是個連續的數值，故線性回歸 `linear_reg` 符合我們的需求。
演算法本身也有各種的參數可以調整，更詳細的內容就交給 [文檔](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-create) 來解決。

跑完後可以看到剛創建的 taxi 下面蹦出 `taxifare_model` 。

<img src="https://cdn.qwiklabs.com/3e%2B%2Bn9FtlJ4zZDbP1edDClbadjTZmcifikRDzSbd3fA%3D" width="35%">

# Evaluate model performance

一樣先上 Query 程式碼：

```sql=
#standardSQL
SELECT
  SQRT(mean_squared_error) AS rmse
FROM
  ML.EVALUATE(MODEL taxi.taxifare_model,
  (

-- 此處與上上段程式碼雷同，但使用 params.EVAL 作為篩選條件，請看33行

  WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    pickup_longitude AS pickuplon,
    pickup_latitude AS pickuplat,
    dropoff_longitude AS dropofflon,
    dropoff_latitude AS dropofflat,
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
  WHERE
    trip_distance > 0 AND fare_amount > 0
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.EVAL
  )

  SELECT *
  FROM taxitrips

  ))
```

此段使用驗證集 `params.EVAL` 、 [ML.EVALUATE](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-evaluate) 函數計算推測與正確答案之間的 Root mean square error ，數值越小代表模型越準確。

結果來到 9.5 左右。

# Prediction

那來看看預測的數值吧！把上一段程式碼換成使用 [ML.PREDICT](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-predict) 函數就可以了：

```sql=
#standardSQL
SELECT
*
FROM
  ML.PREDICT(MODEL `taxi.taxifare_model`,
   (

 WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    pickup_longitude AS pickuplon,
    pickup_latitude AS pickuplat,
    dropoff_longitude AS dropofflon,
    dropoff_latitude AS dropofflat,
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
  WHERE
    trip_distance > 0 AND fare_amount > 0
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.EVAL
  )

  SELECT *
  FROM taxitrips

));
```

輸出如下，預測的數值與真實數據看來有段差距：

<img src="https://cdn.qwiklabs.com/vOie0YpofU3oI7zMJrG9OJT%2Bom89nokwCtceenccSc0%3D" width="80%">


# Improving the model

根據實驗結果，結果看起來並不是那麼優秀。

我們再觀察一次想預測的目標之一 －－ `fare_amount` ：

```sql
SELECT
  COUNT(fare_amount) AS num_fares,
  MIN(fare_amount) AS low_fare,
  MAX(fare_amount) AS high_fare,
  AVG(fare_amount) AS avg_fare,
  STDDEV(fare_amount) AS stddev
FROM `nyc-tlc.yellow.trips`
```

<img src="https://cdn.qwiklabs.com/uX%2Fr9WrJRi7usPBEaElFe7gFgAMrm0Q3exGSbGCeV1I%3D" width="70%">

挖賽， Maximum 竟然衝到 503325 ，誰會花這麼多錢搭計程車啊?

我們必須要將這種奇怪的數據清掉！

... ...

有書則長，無書則短。最後的篩選如下：
```sql
WHERE trip_distance > 0 AND fare_amount BETWEEN 6 and 200
    AND pickup_longitude > -75 #limiting of the distance the taxis travel out
    AND pickup_longitude < -73
    AND dropoff_longitude > -75
    AND dropoff_longitude < -73
    AND pickup_latitude > 40
    AND pickup_latitude < 42
    AND dropoff_latitude > 40
    AND dropoff_latitude < 42
```

再來 train 一發！

```sql=
CREATE OR REPLACE MODEL taxi.taxifare_model_2
OPTIONS
  (model_type='linear_reg', labels=['total_fare']) AS

WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    SQRT(POW((pickup_longitude - dropoff_longitude),2) + POW(( pickup_latitude - dropoff_latitude), 2)) as dist, #Euclidean distance between pickup and drop off
    SQRT(POW((pickup_longitude - dropoff_longitude),2)) as longitude, #Euclidean distance between pickup and drop off in longitude
    SQRT(POW((pickup_latitude - dropoff_latitude), 2)) as latitude, #Euclidean distance between pickup and drop off in latitude
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
WHERE trip_distance > 0 AND fare_amount BETWEEN 6 and 200
    AND pickup_longitude > -75 #limiting of the distance the taxis travel out
    AND pickup_longitude < -73
    AND dropoff_longitude > -75
    AND dropoff_longitude < -73
    AND pickup_latitude > 40
    AND pickup_latitude < 42
    AND dropoff_latitude > 40
    AND dropoff_latitude < 42
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.TRAIN
  )

  SELECT *
  FROM taxitrips
```

驗證一下， RMSE 結果來到 5.1 左右，帥呀！

# Outro

身為一個好的機器學習工程師，要做的工作還有很多。
Qwiklabs 的教學就到此為止，一個小時的時限也過得差不多了！

以個人的觀點而言， BQML 能使用的功能似乎還是挺有限。
不過BQ的強項在於處理超大量的數據，要深入研究模型的話可以到 [Colab](https://colab.research.google.com/) 來看看。
