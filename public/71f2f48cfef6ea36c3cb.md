---
title: streamlitで外部のAPIを叩く
tags:
  - Python
  - Streamlit
private: false
updated_at: '2022-12-14T15:23:42+09:00'
id: 71f2f48cfef6ea36c3cb
organization_url_name: null
slide: false
ignorePublish: false
---
pythonで使えるフレームワークの一つに、streamlitというものがあります。
URL:

https://streamlit.io/

python初心者が、気軽にいい感じのデザインのwebアプリを作るのに最適なフレームワークです。
株価の可視化アプリとか

https://streamlit-example-app-commenting-streamlit-app-qqwlsm.streamlit.app/

データを簡単に可視化できたり、誰もが分析ツールを使ったりすることができて非常に便利です。

## 自分で単機能のアプリを作った。
僕もこのフレームワークを使って、ゼミで使う二地点間の距離を計算するアプリを作りました。

https://stead08-omisegohanproject-streamlit-app-04qdaj.streamlit.app/

これは緯度経度から直線距離を導出する計算と
googlemapで二地点の車での移動距離をGoogleのAPIを叩いてデータを取得しています。

### ぶつかった壁

使っているAPIのKeyは外部に公開できないものです。
しかし、ソースコードGithubに上げていて、そのリポジトリからこのアプリの更新がされているのでAPIkeyをソースコードに直接打ち込むことができません。

### Streamlit標準機能を使う
streamlit ではprivateな情報を他のユーザに見られないようにしながら、その値をソースコードに適用できる機能があります。（無知なので同様の機能が他のフレームワークにあるのかはわかりません）

![スクリーンショット 2022-12-14 15.16.28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/687149/54548590-2e9c-882c-49ab-f8af34b1ddcc.png)

ここにAPIキーをあらかじめ保持しておき、
アプリを走らせているときに、その値を呼び出すといった感じです。
ソースコード側は、

```sample.py
#Googleマップで車での移動距離を計算
#google mapライブラリのインポート
import googlemaps
from datetime import datetime
googlemapAPIkey = st.secrets["googlemapAPIkey"]
gmaps = googlemaps.Client(key= googlemapAPIkey)
now = datetime.now()
directions_result = gmaps.directions(shop_selected_longitude_and_latitude,
                                     town_selected_longitude_and_latitude,
                                     departure_time=now)
distance_driving_km = directions_result[0]['legs'][0]['distance']['text']
```

こんな感じにst.secret["secretsで設定したキーの変数名"]をスクリプト上の変数に代入することでapiキーを秘密にしながらAPIを叩くことができます。

共有するような役立つ情報であるかは定かではありませんが、参考になれば幸いです。

