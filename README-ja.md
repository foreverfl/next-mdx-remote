<!-- markdownlint-disable-file MD033 MD041 -->

<!--

# next-mdx-remote

`getStaticProps`や`getServerSideProps`内でMDXをロードし、クライアント側で正しくハイドレーションできるようにする軽量ユーティリティのセットです。

-->

[![next-mdx-remote](./header.png)](.)

---

## インストール

```sh
npm install next-mdx-remote
```

Turbopackと一緒に使用する場合、[この問題](https://github.com/vercel/next.js/issues/64525)が解決されるまで、`next.config.js`に以下を追加する必要があります：

```diff
const nextConfig = {
+  transpilePackages: ['next-mdx-remote'],
}
```

## 例

```jsx
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote } from 'next-mdx-remote'

import Test from '../components/test'

const components = { Test }

export default function TestPage({ source }) {
  return (
    <div className="wrapper">
      <MDXRemote {...source} components={components} />
    </div>
  )
}

export async function getStaticProps() {
  // MDXテキスト - ローカルファイル、データベース、どこからでも取得可能
  const source = 'Some **mdx** text, with a component <Test />'
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

これら2つが同じファイルにあるのは奇妙に見えるかもしれませんが、これはNext.jsの素晴らしい点の1つです。`getStaticProps`と`TestPage`は同じファイルに現れていますが、2つの異なる場所で実行されます。最終的に、ブラウザバンドルには`getStaticProps`やサーバーでのみ使用される関数は一切含まれないので、`serialize`はブラウザバンドルから完全に削除されます。

> **重要**: `next-mdx-remote`のコードを別の「ユーティリティ」ファイルに入れることには非常に注意してください。そうすると、Next.jsのコード分割機能に問題が発生する可能性が高いです。サーバーサイドでのみ使用されるものとクライアントバンドルに残すべきものを明確に区別できる必要があります。`next-mdx-remote`のコードを外部ユーティリティファイルに入れて何かが壊れた場合は、それを削除し、上記の簡単な例から始めてから問題を報告してください。

### 追加の例

<details>
  <summary>フロントマターの解析</summary>

マークダウンは一般的にフロントマターと一緒に使用され、通常これはマークダウンの処理方法にカスタム処理を追加することを意味します。これに対処するため、`next-mdx-remote`にはフロントマターのオプション解析機能が付属しており、`serialize`に`parseFrontmatter: true`を渡すことで有効にできます。

以下がその例です：

```jsx
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote } from 'next-mdx-remote'

import Test from '../components/test'

const components = { Test }

export default function TestPage({ mdxSource }) {
  return (
    <div className="wrapper">
      <h1>{mdxSource.frontmatter.title}</h1>
      <MDXRemote {...mdxSource} components={components} />
    </div>
  )
}

export async function getStaticProps() {
  // MDXテキスト - ローカルファイル、データベース、どこからでも取得可能
  const source = `---
title: Test
---

Some **mdx** text, with a component <Test name={frontmatter.title}/>
  `

  const mdxSource = await serialize(source, { parseFrontmatter: true })
  return { props: { mdxSource } }
}
```

_フロントマターの解析には[`vfile-matter`](https://github.com/vfile/vfile-matter)が使用されます。_

</details>

<details>
  <summary>`scope`を使用してコンポーネントにカスタムデータを渡す</summary>

`<MDXRemote />`は`scope`プロップを受け取り、これによりMDX内で使用可能なすべての値が利用できるようになります。

`scope`引数の各キー/値ペアはJavaScript変数として公開されます。例えば、`{ foo: 'bar' }`のようなスコープがあった場合、`const foo = 'bar'`として解釈されます。

これは特に、`scope`引数のキー名が有効なJavaScript変数名であることを確認する必要があることを意味します。例えば、`{ 'my-variable-name': 'bar' }`を渡すと_エラー_が発生します。キー名が有効なJavaScript変数名ではないためです。

また、`scope`変数は_コンポーネントの引数_として消費される必要があり、テキストの途中でレンダリングすることはできないことに注意することが重要です。これは以下の例で示されています。

```jsx
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote } from 'next-mdx-remote'

import Test from '../components/test'

const components = { Test }
const data = { product: 'next' }

export default function TestPage({ source }) {
  return (
    <div className="wrapper">
      <MDXRemote {...source} components={components} scope={data} />
    </div>
  )
}

export async function getStaticProps() {
  // MDXテキスト - ローカルファイル、データベース、どこからでも取得可能
  const source =
    'Some **mdx** text, with a component using a scope variable <Test product={product} />'
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

</details>

<details>
  <summary>代わりに`serialize`関数に`scope`を渡す</summary>

`serialize`にカスタムデータを渡すこともでき、その値を通過させて結果から利用可能にします。`source`の結果を`<MDXRemote />`に展開することで、データが利用可能になります。

`serialize`に渡される任意のスコープ値はシリアライズ可能である必要があり、関数やコンポーネントを渡すことはできないことに注意してください。さらに、`scope`引数で名前付けられた任意のキーは有効なJavaScript変数名である必要があります。シリアライズできないカスタムスコープを渡す必要がある場合は、レンダリングされる場所で`<MDXRemote />`に直接`scope`を渡すことができます。この方法の例はこのセクションの上にあります。

```jsx
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote } from 'next-mdx-remote'

import Test from '../components/test'

const components = { Test }
const data = { product: 'next' }

export default function TestPage({ source }) {
  return (
    <div className="wrapper">
      <MDXRemote {...source} components={components} />
    </div>
  )
}

export async function getStaticProps() {
  // MDXテキスト - ローカルファイル、データベース、どこからでも取得可能
  const source =
    'Some **mdx** text, with a component <Test product={product} />'
  const mdxSource = await serialize(source, { scope: data })
  return { props: { source: mdxSource } }
}
```

</details>

<details>
  <summary>
    <code>MDXProvider</code>からのカスタムコンポーネント<a id="mdx-provider"></a>
  </summary>

アプリケーションでレンダリングされる任意の`<MDXRemote />`にコンポーネントを利用可能にしたい場合は、`@mdx-js/react`の[`<MDXProvider />`](https://mdxjs.com/docs/using-mdx/#mdx-provider)を使用できます。

```jsx
// pages/_app.jsx
import { MDXProvider } from '@mdx-js/react'

import Test from '../components/test'

const components = { Test }

export default function MyApp({ Component, pageProps }) {
  return (
    <MDXProvider components={components}>
      <Component {...pageProps} />
    </MDXProvider>
  )
}
```

```jsx
// pages/test.jsx
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote } from 'next-mdx-remote'

export default function TestPage({ source }) {
  return (
    <div className="wrapper">
      <MDXRemote {...source} />
    </div>
  )
}

export async function getStaticProps() {
  // MDXテキスト - ローカルファイル、データベース、どこからでも取得可能
  const source = 'Some **mdx** text, with a component <Test />'
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

</details>

<details>
  <summary>
    ドットを含むコンポーネント名（例：<code>motion.div</code>）
  </summary>

`framer-motion`のようなドット（`.`）を含むコンポーネント名は、他のカスタムコンポーネントと同じ方法でレンダリングできます。コンポーネントオブジェクトに`motion`を渡すだけです。

```js
import { motion } from 'framer-motion'

import { MDXProvider } from '@mdx-js/react'
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote } from 'next-mdx-remote'

export default function TestPage({ source }) {
  return (
    <div className="wrapper">
      <MDXRemote {...source} components={{ motion }} />
    </div>
  )
}

export async function getStaticProps() {
  // MDXテキスト - ローカルファイル、データベース、どこからでも取得可能
  const source = `Some **mdx** text, with a component:

<motion.div animate={{ x: 100 }} />`
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

</details>

<details>
  <summary>遅延ハイドレーション</summary>

遅延ハイドレーションは、クライアント側でのコンポーネントのハイドレーションを遅延させます。これはアプリケーションの初期ロードを改善するための最適化テクニックですが、MDXコンテンツ内の動的コンテンツの対話性に予期せぬ遅延をもたらす可能性があります。

_注意：これはレンダリングされたMDXの周りに追加のラッパー`div`を追加します。これは[レンダリング中のハイドレーションの不一致](https://reactjs.org/docs/react-dom.html#hydrate)を避けるために必要です。_

```jsx
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote } from 'next-mdx-remote'

import Test from '../components/test'

const components = { Test }

export default function TestPage({ source }) {
  return (
    <div className="wrapper">
      <MDXRemote {...source} components={components} lazy />
    </div>
  )
}

export async function getStaticProps() {
  // MDXテキスト - ローカルファイル、データベース、どこからでも取得可能
  const source = 'Some **mdx** text, with a component <Test />'
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

</details>

## API

このライブラリは、`serialize`関数と`<MDXRemote />`コンポーネントを公開しています。これらの2つは意図的に独自のファイルに分離されています。`serialize`は**サーバーサイド**　で実行されることを意図しており、サーバー/ビルド時に実行される`getStaticProps`内で使用されます。一方、`<MDXRemote />`はクライアントサイド、つまりブラウザで実行されることを意図しています。

- **`serialize(source: string, { mdxOptions?: object, scope?: object, parseFrontmatter?: boolean })`**

  **`serialize`** はMDXの文字列を消費します。オプションで、[MDXに直接渡される](https://mdxjs.com/docs/extending-mdx/)オプションとMDXスコープに含めることができるスコープオブジェクトを渡すこともできます。この関数は、`<MDXRemote />`に直接渡すことを意図したオブジェクトを返します。

  ```ts
  serialize(
    // 文字列としての生のMDXコンテンツ
    '# hello, world',
    // オプションのパラメータ
    {
      // カスタムMDXコンポーネントの引数で利用可能
      scope: {},
      // MDXの利用可能なオプション、詳細はMDXのドキュメントを参照してください。
      // https://mdxjs.com/packages/mdx/#compilefile-options
      mdxOptions: {
        remarkPlugins: [],
        rehypePlugins: [],
        format: 'mdx',
      },
      // MDXソースからフロントマターを解析するかどうかを示します
      parseFrontmatter: false,
    }
  )
  ```

  利用可能な`mdxOptions`については<https://mdxjs.com/packages/mdx/#compilefile-options>を参照してください。

- **`<MDXRemote compiledSource={string} components?={object} scope?={object} lazy?={boolean} />`**

  **`<MDXRemote />`** は`serialize`の出力とオプションのcomponents引数を消費します。その結果はコンポーネントに直接レンダリングできます。コンテンツのハイドレーションを遅延させ、静的マークアップを即座に提供するには、`lazy`プロップを渡します。

  ```ts
  <MDXRemote {...source} components={components} />
  ```

### デフォルトコンポーネントの置き換え

レンダリングは内部で[`MDXProvider`](https://mdxjs.com/docs/using-mdx/#mdx-provider)を使用します。これは、HTMLタグをカスタムコンポーネントで置き換えられることを意味します。これらのコンポーネントはMDXJSの[コンポーネントテーブル](https://mdxjs.com/table-of-components/)にリストされています。

使用例としては、好みのスタイリングライブラリでコンテンツをレンダリングすることです。

```jsx
import { Typography } from "@material-ui/core";

const components = { Test, h2: (props) => <Typography variant="h2" {...props} /> }
...
```

お好みであれば、コンポーネントを`<MDXRemote />`に直接渡す代わりに、アプリケーション全体を`<MDXProvider />`でラップすることもできます。上記の[例](#mdx-provider)を参照してください。

注意：コンポーネント名に "/"が含まれるため、`th/td`は機能しません。

## 背景と理論

Next.jsアプリでMDXファイルをロードするための良いデフォルトの方法は実際にありません。以前、MDXファイルをレイアウトにレンダリングし、そのフロントマターをインポートしてインデックスページを作成できるようにするために[`next-mdx-enhanced`](https://github.com/hashicorp/next-mdx-enhanced)を作成しました。

`next-mdx-enhanced`からのこのワークフローは問題ありませんでしたが、`next-mdx-remote`で解決したいくつかの制限がありました：

- **ファイルコンテンツはローカルでなければなりません。** MDXファイルを別のレポジトリやデータベースなどに保存することはできません。十分に大規模な運用では、コンテンツを作成する人々とコンテンツのプレゼンテーションに取り組む人々の間に分割が生じることになります。同じレポジトリでこれら2つの懸念事項を重複させると、全員にとってより困難なワークフローになります。
- **ファイルシステムベースのルーティングに縛られます。** ページはその場所に応じてURLで生成されます。または、`exportPathMap`を使用してリマップすることもできますが、これは作成者に混乱を招きます。いずれにしても、ページを移動すると何かが壊れます。ページのURLまたは`exportPathMap`の設定のいずれかです。
- **パフォーマンスの問題に直面することになります。** WebpackはJavaScriptバンドラーであり、数百/数千ページのテキストコンテンツをロードすることを強制すると、メモリ要件が爆発的に増加します。Webpackは各ページを大量のメタデータを持つ個別のオブジェクトとして保存します。数百ページを持つ我々の実装の1つでは、サイトをコンパイルするのに8GB以上のメモリが必要でした。ビルドには25分以上かかりました。
- **関係データを構造化する方法が制限されます。** フロントマターがJavaScriptオブジェクトにパースされてメモリに保持されるという全データ構造では、コンテンツを動的で関連するカテゴリに整理するのが困難です。

そこで、`next-mdx-remote`はパターン全体を変更し、MDXコンテンツをインポートを通じてではなく、`getStaticProps`や`getServerProps`を通じてロードします。つまり、他のデータをロードするのと同じ方法です。このライブラリは、パフォーマンスの高い方法でMDXコンテンツをシリアライズおよびハイドレートするためのツールを提供します。これにより、上記のすべての制限が解消され、しかも大幅に低コストで実現されます。`next-mdx-enhanced`は多くのカスタムロジックと[いくつかの煩わしい制限](https://github.com/hashicorp/next-mdx-enhanced/issues/17)を持つ非常に重いライブラリです。非公式のテストでは、ビルド時間が50%以上短縮されることが示されています。

このプロジェクトが最初に作成されて以来、Kent C. Doddsが類似のプロジェクト[`mdx-bundler`](https://github.com/kentcdodds/mdx-bundler)を作成しました。このライブラリは、MDXファイル内のインポートとエクスポートをサポートし（各インポートされたファイルの内容を手動で読み取って渡す限り）、フロントマターを自動的に処理します。すべてのファイルが異なるコンポーネントをインポートして使用する場合、`mdx-bundler`を使用することで便利に使える場合があります。現在、`next-mdx-remote`はコンポーネントをインポートしてすべてのページで利用可能にすることしかできないためです。ただし、この機能にはコストがかかることに注意することが重要です。基本的なマークダウンコンテンツの場合、`mdx-bundler`の出力は`next-mdx-remote`の出力よりも少なくとも400%大きくなります。

### これでブログを構築するにはどうすればいいですか？

データによると、すべての開発者ツールの使用例の99%は、不必要に複雑な個人ブログを構築することです。冗談です。しかし、真剣に、個人や小規模ビジネス用のブログを構築しようとしている場合は、通常のHTMLとCSSを使用することを検討してください。シンプルなブログを作成するために重いフルスタックJavaScriptフレームワークを使用する必要は絶対にありません。数年後に更新を行うために戻ってきたとき、すべての依存関係に10回の破壊的なリリースがなかったことに感謝するでしょう。

しかし、本当に主張するなら、[公式のNext.js実装例](https://github.com/vercel/next.js/tree/canary/examples/with-mdx-remote)をチェックしてください。💖

## 注意事項

### 環境ターゲット

`next-mdx-remote`によって生成されるコードは、実際にMDXをレンダリングするために使用され、モジュールサポートを持つブラウザをターゲットにしています。古いブラウザをサポートする必要がある場合は、`serialize`からの`compiledSource`出力をトランスパイルすることを検討してください。

### `import` / `export`

`import`および`export`文は、MDXファイルの**内部**で使用することはできません。MDXファイルでコンポーネントを使用する必要がある場合は、`<MDXRemote />`にプロップとして提供する必要があります。

これは理解できるはずです。なぜなら、機能するためにはインポートがファイルパスに相対的である必要があり、このライブラリは設定されたファイルパスからのローカルコンテンツのみをロードするのではなく、どこからでもコンテンツをロードできるようにするためです。エクスポートに関しては、MDXコンテンツはモジュールではなくデータとして扱われるため、`next-mdx-remote`に渡されたMDXからエクスポートされる可能性のある値にアクセスする方法はありません。

## セキュリティ

このライブラリは、クライアント側でJavaScriptの文字列を評価し、それによってMDXコンテンツをリモート処理します。文字列をJavaScriptに評価することは、注意して行わないと危険を伴う可能性があります。XSS攻撃を可能にする可能性があるためです。ドキュメントで指示されているように、`serialize`関数によって生成された`mdxSource`入力のみを`<MDXRemote />`に渡していることを確認することが重要です。**ユーザー入力を`<MDXRemote />`に直接渡さないでください。**

ウェブサイトに`eval`や`new Function()`を介したコード評価を許可しないCSPがある場合、`next-mdx-remote`を利用するためにその制限を緩和する必要があります。これは[`unsafe-eval`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src#common_sources)を使用して行うことができます。

## TypeScript

このプロジェクトにはTypeScript用のネイティブタイプが含まれています。`serialize`と`<MDXRemote />`の両方が通常予想される通りのタイプを持ち、ライブラリは`getStaticProps`の結果をタイプ付けするために使用できるタイプもエクスポートします。

- `MDXRemoteSerializeResult<TScope = Record<string, unknown>>`：`serialize`の戻り値を表します。`TScope`ジェネリックタイプを渡して、渡すスコープデータのタイプを表すことができます。

以下は、TypeScriptでの単純な実装の例です。TypeScriptのすべての設定でタイプを正確にこの方法で実装する必要はないかもしれません。この例は、必要に応じてタイプをどこに適用できるかを示すデモンストレーションに過ぎません。

```tsx
import type { GetStaticProps } from 'next'
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote, type MDXRemoteSerializeResult } from 'next-mdx-remote'
import ExampleComponent from './example'

const components = { ExampleComponent }

interface Props {
  mdxSource: MDXRemoteSerializeResult
}

export default function ExamplePage({ mdxSource }: Props) {
  return (
    <div>
      <MDXRemote {...mdxSource} components={components} />
    </div>
  )
}

export const getStaticProps: GetStaticProps<{
  mdxSource: MDXRemoteSerializeResult
}> = async () => {
  const mdxSource = await serialize('some *mdx* content: <ExampleComponent />')
  return { props: { mdxSource } }
}
```

## React Server Components (RSC) & Next.js `app` ディレクトリのサポート

サーバーコンポーネント、特にNext.jsの`app`ディレクトリ内での`next-mdx-remote`の使用は、`next-mdx-remote/rsc`からのインポートによってサポートされています。以前は、シリアライズとレンダリングのステップが分離されていましたが、今後はRSCがこの分離を不要にします。

注目すべきいくつかの違い：

- `<MDXRemote />`は現在、`next-mdx-remote/serialize`からのシリアライズされた出力を受け入れる代わりに、`source`プロップを受け入れます
- カスタムコンポーネントは、RSCがReact Contextをサポートしていないため、`@mdx-js/react`の`MDXProvider`コンテキストを使用して提供することはできなくなりました
- `parseFrontmatter: true`を渡す際にMDX外部でフロントマターにアクセスするには、`next-mdx-remote/rsc`から公開されている`compileMdx`メソッドを使用します
- `lazy`プロップはサポートされなくなりました。レンダリングがサーバーで行われるためです
- `<MDXRemote />`は現在非同期コンポーネントであるため、サーバーでレンダリングする必要があります。クライアントコンポーネントはMDXマークアップの一部としてレンダリングできます

RSCの詳細については、[Next.jsのドキュメント](https://nextjs.org/docs/app/building-your-application/rendering/server-components)をチェックしてください。

### 例

_`app`ディレクトリを使用するNext.js 13+アプリケーションでの使用を想定しています。_

#### 基本

```tsx
import { MDXRemote } from 'next-mdx-remote/rsc'

// app/page.js
export default function Home() {
  return (
    <MDXRemote
      source={`# Hello World

      This is from Server Components!
      `}
    />
  )
}
```

#### ローディング状態

```tsx
import { MDXRemote } from 'next-mdx-remote/rsc'

// app/page.js
export default function Home() {
  return (
    // 理想的には、このローディングスピナーはレイアウトシフトがないことを保証します。
    // これはそのようなローディングスピナーを提供する方法の例です。
    // Next.jsでは、これに`loading.js`を使用することもできます。
    <Suspense fallback={<>Loading...</>}>
      <MDXRemote
        source={`# Hello World

        This is from Server Components!
        `}
      />
    </Suspense>
  )
}
```

#### カスタムコンポーネント

```tsx
// components/mdx-remote.js
import { MDXRemote } from 'next-mdx-remote/rsc'

const components = {
  h1: (props) => (
    <h1 {...props} className="large-text">
      {props.children}
    </h1>
  ),
}

export function CustomMDX(props) {
  return (
    <MDXRemote
      {...props}
      components={{ ...components, ...(props.components || {}) }}
    />
  )
}
```

```tsx
// app/page.js
import { CustomMDX } from '../components/mdx-remote'

export default function Home() {
  return (
    <CustomMDX
      // h1は`large-text`クラス名でレンダリングされるようになりました
      source={`# Hello World
      This is from Server Components!
    `}
    />
  )
}
```

#### MDX外部でフロントマターにアクセスする

```tsx
// app/page.js
import { compileMDX } from 'next-mdx-remote/rsc'

export default async function Home() {
  // オプションでフロントマターオブジェクトのタイプを提供します
  const { content, frontmatter } = await compileMDX<{ title: string }>({
    source: `---
title: RSC Frontmatter Example
---
# Hello World
This is from Server Components!
`,
    options: { parseFrontmatter: true },
  })
  return (
    <>
      <h1>{frontmatter.title}</h1>
      {content}
    </>
  )
}
```

## 代替案

`next-mdx-remote`はサポートする機能について意見を持っています。`next-mdx-remote`が提供しない追加機能が必要な場合は、以下のいくつかの代替案を検討してください：

- [`mdx-bundler`](https://github.com/kentcdodds/mdx-bundler)
- [`next-mdx-remote-client`](https://github.com/ipikuka/next-mdx-remote-client)
- [`remote-mdx`](https://github.com/devjiwonchoi/remote-mdx)

### `next-mdx-remote`が必要ない可能性があります

React Server Componentsを使用していて、カスタムコンポーネントを持つ基本的なMDXを使用しようとしているだけなら、コアMDXライブラリ以外に何も必要ありません。

```js
import { compile, run } from '@mdx-js/mdx'
import * as runtime from 'react/jsx-runtime'
import ClientComponent from './components/client'

// MDXはファイルやデータベースなど、どこからでも取得できます。
const mdxSource = `# Hello, world!
<ClientComponent />
`

export default async function Page() {
  // MDXソースコードを関数本体にコンパイルします
  const code = String(
    await compile(mdxSource, { outputFormat: 'function-body' })
  )
  // その後、サーバーでコードを実行してサーバーコンポーネントを生成するか、
  // 最終的なレンダリングのために文字列をクライアントコンポーネントに渡すことができます。

  // ランタイムでコンパイルされたコードを実行し、デフォルトエクスポートを取得します
  const { default: MDXContent } = await run(code, {
    ...runtime,
    baseUrl: import.meta.url,
  })

  // MDXコンテンツをレンダリングし、ClientComponentをコンポーネントとして提供します
  return <MDXContent components={{ ClientComponent }} />
}
```

コンパイルされた文字列をデータベースやクライアントコンポーネントに渡す予定がない場合は、`evaluate`を使用してこのアプローチを簡略化することもできます。`evaluate`は1回の呼び出しでコードをコンパイルして実行します。

```js
import { evaluate } from '@mdx-js/mdx'
import * as runtime from 'react/jsx-runtime'
import ClientComponent from './components/client'

// MDXはファイルやデータベースなど、どこからでも取得できます。
const mdxSource = `
export const title = "MDX Export Demo";

# Hello, world!
<ClientComponent />

export function MDXDefinedComponent() {
  return <p>MDX-defined component</p>;
}
`

export default async function Page() {
  // コンパイルされたコードを実行します
  const {
    default: MDXContent,
    MDXDefinedComponent,
    ...rest
  } = await evaluate(mdxSource, runtime)

  console.log(rest) // { title: 'MDX Export Demo' } をログ出力

  // MDXコンテンツをレンダリングし、ClientComponentをコンポーネントとして提供し、
  // エクスポートされたMDXDefinedComponentをレンダリングします。
  return (
    <>
      <MDXContent components={{ ClientComponent }} />
      <MDXDefinedComponent />
    </>
  )
}
```

## ライセンス

[Mozilla Public License Version 2.0](./LICENSE)