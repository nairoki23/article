---
title: DiscordのWebhookをもっと便利に使う方法
tags: discord Webhook Python
author: nairoki
slide: false
---
# はじめに
皆さんメリクリ。ないろきです。
Discordのwebhookの機能として私が一般的によく使われているな〜と感じる機能は、メッセージを送信できる機能とそれに埋め込みを入れる機能、fileを送信する機能ぐらいです。しかし、先日 [Discord公式apiドキュメント](https://discord.com/developers/docs/resources/webhook#execute-webhook)を確認してみたところ、様々なwebhookの機能が掲載されていました。
そこで、この記事ではDiscordをスクリプトと連携させていろんなことをしたい。でもBotを常時稼働させたくない！というときなどに使えるWebhookの便利な機能を紹介していきたいと思います。

# 対象読書
- Discordのwebhookでどんなことができるかもっと知りたい人
- わざわざDiscordBotを作らずに便利なスクリプトを組みたい人

# 環境
- Python 3.12.0
- requests 2.31.0
- Ubuntu 23.10

# 実際に使える機能の紹介
## message送信
webhookの使い方としてMessage送信のために使うのはおそらく最も一般的だと思います。しかし皆さんはWebhookからフォームを作ってメッセージを送信したり、フォーム内にメッセージを送信できることをご存知でしょうか？
### スレッド作成
```python
import requests
from json import dumps

WEBHOOK_URL = "<WebhookのURLを入れる>"
contents = {"content":"テストメッセージ","thread_name":"テストスレッド"}
headers = {'Content-Type': 'application/json'}

response = requests.post(WEBHOOK_URL, dumps(contents), headers=headers)
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3589252/e2417f76-41b0-8c68-3f94-6779021c66d3.png)

`"thread_name":"<作りたいスレッドの名前>"`を入れるだけです。
便利そうですね。
thread_nameと書いてありますが、[Discord公式apiドキュメント](https://discord.com/developers/docs/resources/webhook#execute-webhook)のthread_nameに関する記述をGoogle 翻訳すると
> Webhook チャネルはフォーラムまたはメディア チャネルである必要があります。

となっているため名前に反してスレッドは作成できないので注意です。

### スレッド内にメッセージを送信
```python
import requests
from json import dumps

WEBHOOK_URL  = "<WebhookのURLを入れる>"
contents = {"content":"フォーラムに書き込めるよ〜"}
headers      = {'Content-Type': 'application/json'}
params={"thread_id":<フォーラムのid>}
response     = requests.post(WEBHOOK_URL, dumps(contents), headers=headers,params=params)
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3589252/768ea4d8-4f39-c2bd-1a92-2937f34db375.png)

こちらはクエリパラメーターに`?thread_id=<フォーラムのid>`を入れるとできます。
この機能についても、Discord公式ドキュメントに記述はないですが、少なくとも記事執筆時点ではスレッドに対応していませんでした。

## message編集
webhookで送信したmessageを編集することもできます。
```python
import requests
from json import dumps

WEBHOOK_URL  = "<WebhookのURLを入れる>"
message_id= <編集したいmessageのidを入れる>
contents = {"content":"メッセージ変えたよ〜"}
headers      = {'Content-Type': 'application/json'}
response     = requests.patch(WEBHOOK_URL+"/messages/"+str(message_id), dumps(contents), headers=headers)
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3589252/63d840ec-2089-c22f-36b0-815227b5ebe7.png)
を
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3589252/eefca278-e960-9a8a-298d-c3a958852d92.png)
にすることができます。
このリクエストはwebhookのURLの後ろに `/messages/<メッセージのid>` をつけた上でpatchメソッドで送ってください。
また、編集先の内容をmessageを送信するときと同様にリクエストの中に入れてください。

## message削除
```python
import requests
from json import dumps

WEBHOOK_URL  = "<WebhookのURLを入れる>"
message_id=<messageのid>
headers      = {'Content-Type': 'application/json'}
response     = requests.delete(WEBHOOK_URL+"/messages/"+str(message_id),headers=headers)
```

webhook自身で送信したmessageを削除する機能もあります。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3589252/15b9c499-f757-4415-29da-aaf8af6d792c.png)
が
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3589252/8fd38316-47f3-14bb-b2c7-a87c567c35a2.png)

こんなふうに消えます。
このリクエストはwebhookのURLの後ろに `/messages/<メッセージのid>`をつけた上でdeleteメソッドで送ってください。

# 最後に
この記事ではWebhookの便利な使い方を紹介してきました。
こんな感じで公式のドキュメントを見てみればさまざまな機能が見つかることもあります。
みなさんももし時間があったら公式ドキュメントを見てみるのはいかがでしょうか。
# 参考にした記事
https://qiita.com/iroiro_bot/items/48e8a8a9754aacaf7ec9
https://discord.com/developers/docs/resources/webhook#execute-webhook
