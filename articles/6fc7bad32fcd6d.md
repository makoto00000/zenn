---
title: "Next.js 14 App RouterでSWRを使ってみる【useEffectと比較】"
emoji: "🌟"
type: "tech"
topics: [Nextjs, React, useSWR]
published: true
---

## SWRとは

Next.jsを作っているVercel社が開発しているデータフェッチのためのReact Hooksライブラリです。

Stale While Revalidateの頭文字を取っています。

細かい説明は後にして、SWRを使う場合と、使わない場合を比較していきましょう。

## 使用するAPI

今回、以下のAPIを使用します。
<https://github.com/KostaSav/hp-api>

映画「ハリー・ポッター」に登場するキャラクターの情報を取得できるAPIです。

これを使って以下のようなページのサイトを作成します。

![sight image](/images/harry_potter_collection.png)

## 環境構築

今回は`Next.js 14.1.0`で実装していきます。

また、Next.jsを始められる状態から解説しています。`Node.js`や`npm`などのセットアップ等は各自で調べておいてください。

[Next.jsドキュメント日本語翻訳サイト](https://nextjs-ja-translation-docs.vercel.app/docs/getting-started)がわかりやすいので、このサイトに従って、セットアップを進めます。

```shell
npx create-next-app@latest --typescript
```

```shell
Need to install the following packages:
create-next-app@14.1.0
Ok to proceed? (y) y
✔ What is your project named? … harry_potter_collection
✔ Would you like to use ESLint? … No / Yes → Yes
✔ Would you like to use Tailwind CSS? … No / Yes → No
✔ Would you like to use `src/` directory? … No / Yes → Yes
✔ Would you like to use App Router? (recommended) … No / Yes → Yes
✔ Would you like to customize the default import alias (@/*)? … No / Yes → Yes
✔ What import alias would you like configured? … @/* →そのままEnter
```

これで、`harry_potter_collection`というディレクトリが作成されますので、そのディレクトリに入って、コードエディタを起動してください。

ここまでで、環境構築は完了です。

## 事前準備

CSSを書いておきます。
今回は2ページしか作らないので`global.css`に記述します。

今回は重要ではないので、コピペで問題ないです。

すでに書いてあるものをすべて削除して、以下の内容に書き換えてください。

```css:/src/grobal.css
@import url('https://fonts.googleapis.com/css2?family=Cinzel+Decorative:wght@400;700;900&display=swap');

* {
  box-sizing: border-box;
  padding: 0;
  margin: 0;
  font-family: 'Cinzel Decorative', serif;
}

html,
body {
  max-width: 100vw;
  overflow-x: hidden;
}

body {
  background: #1D1E35;
  color: #B69B62;
  padding: 30px 50px;
}

a {
  color: inherit;
  text-decoration: none;
  cursor: pointer;
}

a:hover {
  color: brown;
}

h1 {
  font-size: 60px;
  text-align: center;
  font-weight: 900;
}

.characters {
  margin-top: 50px;
  color: #161616;
  font-family: "cinzel";
}

.character {
  display: flex;
  background: #E5E1E2;
  padding: 10px 50px;
  border-radius: 5px;
  margin-bottom: 50px;
  border: 3px solid #161616;
}

.image {
  margin-right: 20px;
  height: 90px;
  width: 70px;
}

.name {
  font-size: 30px;
  font-weight: 700;
  margin-bottom: 10px;
}

.birth {
  font-size: 20px;
  color: #8C3342;
}

.loading, .error {
  font-size: 30px;
  padding-top: 100px;
  text-align: center;
}

```

続いて、画像のドメインを許可する設定です。
キャラクターの画像をAPIで取得するときに、URL形式で取得するのですが、別に設定をしないと画像を表示させることができないからです。

```js: /next.config.mjs

/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    domains: ['ik.imagekit.io'],
  },
};

export default nextConfig;

```

また、`noimage.jpg`という名前で、`/public`に適当な画像を用意してください。

APIでキャラクターの画像がなかったときに変わりに表示する画像になります。

これで、事前準備は完了です。

## SWRを使わない場合

`クライアントコンポーネント`で取得する方法と、`サーバーコンポーネント`で取得する方法に分けられますが、今回はクライアントサイドでデータをfetchして表示する記述を書いてみます。

```jsx
"use client"

import Image from "next/image";
import Link from "next/link";
import { useEffect, useState } from "react";

type Character = {
  id: string,
  name: string,
  alternate_names: string[],
  species: string,
  gender: string,
  house: string,
  dateOfBirth: string | null,
  yearOfBirth: number | null,
  wizard: boolean,
  ancestry: string,
  eyeColour: string,
  hairColour: string,
  wand: {
    wood: string,
    core: string,
    length: number | null
  },
  patronus: string,
  hogwartsStudent: boolean,
  hogwartsStaff: boolean,
  actor: string,
  alternate_actors: string[],
  alive: boolean,
  image: string
}

export default function Home() {
  const [characters, setCharactors] = useState<Character[] | null>(null);
  const [error, setError] = useState<boolean>(false);
  const [loading, setLoading] = useState<boolean>(true);

  const getCharacters = async() => {
    const res = await fetch("https://hp-api.onrender.com/api/characters");
    try {
      const data = await res.json();
      setCharactors(data);

    } catch (error) {
      setError(true);

    } finally {
      setLoading(false);
    }

  }
  useEffect(() => {
    getCharacters();
  }, [])

if (loading) return (<div className={"loading"} >Loading...</div>)
if (error) return (<div className={"error"} >Data acquisition failed.</div>)
if (characters) return (
    <main>
      <h1>Characters</h1>
      <Link href={"/swr"}>SWR</Link>
      <div className={"characters"}>
        {characters.map((character) => (
          <div key={character.id} className={"character"}>
            <Image
              className={"image"}
              src={character.image === "" ? "/noimage.jpg" : character.image}
              alt={""}
              width={50}
              height={60}
            />
            <div className="data">              
              <h2 className={"name"}>{character.name}</h2>
              <p className={"birth"}>{character.dateOfBirth ? character.dateOfBirth : "unknown"}</p>
            </div>
          </div>
        ))}
      </div>
    </main>
  );
}


```

これでも正常に動作し、データを画面上に出力することはできます。

ただしuseState、useEffectを使って、さらにErrorハンドリングとLoadingの実装など、かなりのコードを書かなければなりません。

## SWRを使う場合

SWRを使えばそのような問題が解決できます。

まずは、swrをインストールします。

```shell
npm install swr
または
yarn install swr
```

そうすると`useSWR`が使えるようになります。

次に、`/src/app`に、`swr`というディレクトリを作成し、その中に`page.tsx`を作成し中身を以下のように記述します。

```diff tsx:/src/app/swr/page.tsx
"use client"

import Image from "next/image";
import Link from "next/link";
- import { useEffect, useState } from "react";
+ import useSWR from "swr";

type Character = {
  id: string,
  name: string,
  alternate_names: string[],
  species: string,
  gender: string,
  house: string,
  dateOfBirth: string | null,
  yearOfBirth: number | null,
  wizard: boolean,
  ancestry: string,
  eyeColour: string,
  hairColour: string,
  wand: {
    wood: string,
    core: string,
    length: number | null
  },
  patronus: string,
  hogwartsStudent: boolean,
  hogwartsStaff: boolean,
  actor: string,
  alternate_actors: string[],
  alive: boolean,
  image: string
}

+ const fetcher = async (key: string) => {
+   return await fetch(key).then((res) => res.json());
+ }

export default function Home() {
+ const {data, error, isLoading} = useSWR<Character[]>("https://hp-api.onrender.com/api/characters", fetcher)
-  const [characters, setCharactors] = useState<Character[] | null>(null);
-  const [error, setError] = useState<boolean>(false);
-  const [loading, setLoading] = useState<boolean>(true);

- const getCharacters = async() => {
-   const res = await fetch("https://hp-api.onrender.com/api/characters");
-   try {
-     const data = await res.json();
-     setCharactors(data);

-   } catch (error) {
-     setError(true);

-   } finally {
-     setLoading(false);
-   }

- }
- useEffect(() => {
-   getCharacters();
- }, [])

if (isLoading) return (<div className={"loading"} >Loading...</div>)
if (error) return (<div className={error} >Data acquisition failed.</div>)
+ if (data) return (
    <main>
      <h1>Characters</h1>
      <Link className={"link"} href={"/"}>TOP</Link>
      <div className={"characters"}>
+       {data.map((character) => (
          <div key={character.id} className={"character"}>
            <Image
              className={"image"}
              src={character.image === "" ? "/noimage.jpg" : character.image}
              alt={""}
              width={50}
              height={60}
            />
            <div className="data">              
              <h2 className={"name"}>{character.name}</h2>
              <p className={"birth"}>{character.dateOfBirth ? character.dateOfBirth : "unknown"}</p>
            </div>
          </div>
        ))}
      </div>
    </main>
  );
}

```

`uesEffect`,`useState`を使う必要がなくなり、コードを大幅に減らすことができていますね。

## useSWRの使い方

使いたいページで`useSWR`をインポートします。

```tsx
 import useSWR from "swr";
```

基本の使い方は以下のようになります。

```tsx
const {data, error, isLoading} = useSWR(url, fetcher)
```

この一行で、fetchで取得する`data`、エラーハンドリングの`error`、取得中を表す`issLoading`を同時に定義することができます。

try-catch文で長々と記述していたのがこれだけで済みましたね。

:::message
`data`, `error`, `isLoading`はswrで定義されているプロパティなので、独自の名前に変えることはできません。
しかし、カスタムフックにすれば、自由に定義できます。

```tsx
  const useCharacter = () => {
    const { data, error, isLoading } = useSWR(
      "https://hp-api.onrender.com/api/characters",
      fetcher
    );
    return {
      characters: data,
      isError: error,
      isLoading,
    }
  }
  const {characters, isError, isLoading} = useCharacter();
```

このように、`useCharacter`を定義しておいて、その返り値に`data`, `error`, `isLoading`を定義してしまえば、好きなプロパティ名をつけることができます。

:::

続いて、useSWRの第一引数には、エンドポイントのURLを入れます。

第二引数のfetcherは、以下のように定義しています。

```tsx
 const fetcher = async (key: string) => {
   return await fetch(key).then((res) => res.json());
 }
 ```

 `key`としてエンドポイントのURLを受け取り、レスポンスをjsonにして返す関数を定義しています。

 この中で、先程のように`error`や`isLoading`を定義しなくてもいいのは、useSWRが全部やってくれているからです。

なのでfetchする関数の定義もこのようにシンプルに書くことができるのです。

:::message
第三引数には、optionを入れることができます。
例えば、dataに初期データを入れたり、error時に再取得する設定を書いたりできます。

詳しくは[公式ドキュメント](https://swr.vercel.app/ja/docs/api)をご参照ください。
:::

## データのキャッシュ

ページの上部に、SWRを使っていないページ（<http://localhost:3000>）と、SWRを使っているページ（<http://localhost:3000/swr>）へ相互に行き来できるリンクがあるので、クリックして移動してみてください。

SWRを使っていないページは、ページ遷移の度に`useEffect`が動いて毎回データを取得しています。

一方、SWRを使っているページでは、初めて訪れるときにデータを取得し、それ以降はキャッシュされたデータを表示するだけなので、高速にレンダリングされるはずです。

これがSWRを使うメリットの一つです。

## まとめ

- SWRを使うと少ないコードでデータの取得、エラーハンドリング、Loadingを実装できる。
- 取得したデータをキャッシュしてくれるため、複数回リクエストを送らなくて済む。

ちなみに、WRが中でどんなことをしているのか、こちらの記事で詳しく解説されていましたので、最後に共有しておきます。
https://zenn.dev/yuitosato/articles/0aa4dbd13807ef