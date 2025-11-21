---
title: Minio×Hono×Reactをやろうとした経験談を書いておこうと思ふ。
tags: minio
author: nairoki
slide: false
---
この記事は[木更津高専 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/nit_kisarazu)参加記事です．
前の前の記事:[今年もプロ研が学園祭Webサイト制作をした話](https://qiita.com/naotiki/items/89e18320b8ec20dd91cb) by [naotiki](https://qiita.com/naotiki)
前の記事:[祇園祭Webサイト2024のサーバ周りについて](https://zenn.dev/nitkc_proken/articles/d1af2d9aeb029b) by [NXVZBGBFBEN](https://zenn.dev/nxvzbgbfben)
次の記事:[Vikeを試してみた](https://qiita.com/naotiki/items/96a7786db3806261f862) by [naotiki](https://qiita.com/naotiki)

# はじめに
皆さんメリクリ。ないろきです。
先日私達で作成した、祇園祭webサイトでは画像などを保存するオブジェクトストレージとしてMinIOを使用しました。
その際に、MinioとHonoを組み合わせている記事が少なくとも軽く調べた範囲では見つからなかったので、大まかな経験とやり方を書いた記事をここに残しておこうと思います(まぁ愚直に実装しただけですが。)
祇園祭webに自体についての情報は[naotiki](https://qiita.com/naotiki)さんの[今年もプロ研が学園祭Webサイト制作をした話](https://qiita.com/naotiki/items/89e18320b8ec20dd91cb)を参照してください。

# 使ったjsライブラリ・フレームワーク
- Hono
- react(remix)
- sharp
- S3Client
- Zod

# やったこと
とりあえずこの記事では適当なエンドポイントを作成し、jsonに含まれるbase64エンコードされた画像をMiniOに登録することを目指します。
## DockerでMiniOの環境構築
試すためにはとにかくMiniOの環境が必要なのでサクッとDockerで作成しましょう
```yaml:docker-compose.yaml
services:
  minio:
    image: minio/minio:RELEASE.2024-08-29T01-40-52Z
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: root
      MINIO_ROOT_PASSWORD: password
    command: server /data --console-address :9001
    volumes:
      -  "minio_bucket:/data"

  minio_mc:
    image: minio/mc:RELEASE.2024-08-26T10-49-58Z
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
      until (mc alias set myminio http://minio:9000 root password) do echo '...waiting...' && sleep 1; done;
      mc mb myminio/gionsai;
      mc mb myminio/content;
      mc anonymous set download myminio/gionsai;
      mc anonymous set download myminio/content;
      mc admin user add myminio hono_client firefire;
      mc admin policy create myminio hono /policy/hono.json;
      mc admin policy attach myminio hono --user hono_client;
      echo '終了';
      exit 0;
      "
    volumes:
      - "./minio/policy:/policy"

volumes:
  minio_bucket:
```
minio自体を立ち上げたあと、mcを立ち上げて`gionsai`と`content`バケットを作成し、権限情報を読ませてexitで終了しています。
ここのところは、mcコマンドを各自都合の良いように書いてください。

## Honoのindex.tsを用意
```ts:index.js
import { serve } from "@hono/node-server";
import { Hono } from "hono";
import { env } from "~/lib/env";
import { minioRoute } from "./minio";

const app = new Hono().basePath("/api");
const route = app
	.get("/", (c) => {
		return c.text("Hello Hono!");
	})
	.route("/minio", minioRoute);

const port = Number(env.PORT) || 3000;
console.log(`Server is running on port ${port}`);

serve({
	fetch: app.fetch,
	port,
});

export type ApiType = typeof route;
```
必要最低限な状態のHonoのエンドポイントです。
```ts
.route("/minio", minioRoute);
```
この部分でminioのRouterを登録しています。


## Honoにエンドポイントを追加
まずはコードの全体像です。
```ts:minio.ts
import { Buffer } from "node:buffer";
import { PutObjectCommand, S3Client } from "@aws-sdk/client-s3";
import { zValidator } from "@hono/zod-validator";
//MinIOのExampleです
import { Hono } from "hono";
import sharp from "sharp";
import { z } from "zod";
import { env } from "~/lib/env";

const schema = z.object({
	text: z.string(),
	image: z.string(),
});
const minio = new S3Client({
	region: "ap-northeast-1",
	endpoint: env.MINIO_ENDPOINT || "",
	forcePathStyle: true, // MinIO利用時には必要そう。
	credentials: {
		accessKeyId: env.MINIO_USER || "",
		secretAccessKey: env.MINIO_PASSWORD || "",
	},
});
const db = [];

export const minioRoute = new Hono().post(
	"/",
	zValidator("json", schema.pick({text:true,image:true})),
	async (c) => {
		try{
			const data = c.req.valid("json");
			const fileData = data.image.replace(/^data:\w+\/\w+;base64,/, "");
			const decodedBuffer = Buffer.from(fileData, "base64");
			const image_file = await sharp(decodedBuffer).toFormat("webp").toBuffer();
			db.push("test")
			await minio.send(new PutObjectCommand({
					Bucket: "content",
					Key: `${(db.length + 1).toString()}.webp`,
					Body: image_file,
				}),
			);

		return c.json({ message: "さくせす" }, 201);
		}catch(err){console.log(err)
		return c.json({message:"異常！"},500);
		}
	},
);
```
上から順番にかいつまんで説明していきます
```ts
const minio = new S3Client({
	region: "ap-northeast-1",
	endpoint: env.MINIO_ENDPOINT || "",
	forcePathStyle: true, // MinIO利用時には必要そう。
	credentials: {
		accessKeyId: env.MINIO_USER || "",
		secretAccessKey: env.MINIO_PASSWORD || "",
	},
});
```
上記のコードでS3Clientを利用しMiniOと接続するためのインスタンスを作成してます。
```ts
export const minioRoute = new Hono().post(
	"/",
	zValidator("json", schema.pick({text:true,image:true})),
	async (c) => {
		try{
			const data = c.req.valid("json");
			const fileData = data.image.replace(/^data:\w+\/\w+;base64,/, "");
			const decodedBuffer = Buffer.from(fileData, "base64");
			const image_file = await sharp(decodedBuffer).toFormat("webp").toBuffer();
			db.push("test")
			await minio.send(new PutObjectCommand({
					Bucket: "content",
					Key: `${(db.length + 1).toString()}.webp`,
					Body: image_file,
				}),
			);

		return c.json({ message: "成功" }, 201);
		}catch(err){console.log(err)
		return c.json({message:"異常発生"},500);
		}
	},
);
```
Routerの部分です。
POSTで受け取ったjsonからZodを活用しつつkeyが`image`なvalueを取り出しbase64エンコードされた文字列を、Buffer形式に変えて、Sharp(ライブラリ)を使いwebpに変更し、minio.sendの部分でcountentバケットに適当なkeyでファイルを登録しています。
あくまでもこれは雑なサンプルなので適宜使いやすいように書き換えてください。

## 画像をHonoに投げるための簡易的なページの作成
```tsx
import type React from "react";
import { useState } from "react";

export default function Page() {
	const [file, setFile] = useState<string | null>(null);

	const onChangeFile = (e: React.ChangeEvent<HTMLInputElement>) => {
		const files = e.target.files;
		if (files?.[0]) {
			const fr = new FileReader();
			fr.onload = (e) => {
				if (e.target) {
					setFile(e.target.result);
					alert("Fileのbase64化成功");
				}
			};
			fr.readAsDataURL(files[0]);
		}
	};
	const onClickSubmit = async () => {
		if (!file) {
			alert("Fileが未選択");
			return;
		}
		try {
			const res = await fetch("http://localhost:3000/minio", {
				headers: { "Content-Type": "application/json" },
				method: "POST",
				body: JSON.stringify({ image: file, text: "text" }),
			});
			const json = await res.json();
			console.log(json);
		} catch (err) {
			console.error(err);
		}
	};
	return (
		<>
			<h1 className={"text-3xl text-center mt-4"}>MinioのExample</h1>
			<input name="file" type="file" accept="image/*" onChange={onChangeFile} />
			<button
				className={
					"bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
				}
				disabled={!file}
				onClick={onClickSubmit}
			>
				Submit
			</button>
		</>
	);
}

```
ファイルを登録するためのtailwindで最低限見た目を整えたreactのコードです。



# 最後に
私事にはなりますが、記事を書いたのが久しぶりで、また記事執筆に当てる時間があまりなかったので適当な出来の記事になってます。
ツッコミどころや質問があれば随時コメント下さい。
時間がある際にもっとわかりやすいように書き換えようと思ってます。

# 参考にした記事
https://qiita.com/terukizm/items/6850aa316de425e3e3b4
