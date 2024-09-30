<!-- markdownlint-disable-file MD033 MD041 -->

<!--

# next-mdx-remote

`getStaticProps` 또는 `getServerSideProps` 내에서 MDX를 로드하고 클라이언트에서 올바르게 하이드레이션할 수 있게 해주는 가벼운 유틸리티 세트입니다.

-->

[![next-mdx-remote](./header.png)](.)

---

## 설치

```sh
npm install next-mdx-remote
```

Turbopack과 함께 사용하는 경우, [이 이슈](https://github.com/vercel/next.js/issues/64525)가 해결될 때까지 `next.config.js`에 다음을 추가해야 합니다:

```diff
const nextConfig = {
+  transpilePackages: ['next-mdx-remote'],
}
```

## 예제

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
  // MDX 텍스트 - 로컬 파일, 데이터베이스, 어디서든 가져올 수 있습니다
  const source = 'Some **mdx** text, with a component <Test />'
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

이 두 가지가 같은 파일에 있는 것이 이상해 보일 수 있지만, 이것이 Next.js의 멋진 점 중 하나입니다. `getStaticProps`와 `TestPage`는 같은 파일에 나타나지만 두 개의 다른 장소에서 실행됩니다. 결국 브라우저 번들에는 `getStaticProps`나 서버에서만 사용하는 함수가 전혀 포함되지 않으므로, `serialize`는 브라우저 번들에서 완전히 제거됩니다.

> **중요**: `next-mdx-remote` 코드를 별도의 "유틸리티" 파일에 넣는 것에 대해 매우 주의해야 합니다. 그렇게 하면 Next.js의 코드 분할 기능에 문제가 생길 가능성이 높습니다. Next.js는 서버 사이드에서만 사용되는 것과 클라이언트 번들에 남겨져야 하는 것을 명확하게 구분할 수 있어야 합니다. `next-mdx-remote` 코드를 외부 유틸리티 파일에 넣고 뭔가 깨진 경우, 제거하고 위의 간단한 예제에서 시작한 다음 이슈를 제출하세요.

### 추가 예제

<details>
  <summary>프론트매터 파싱</summary>

마크다운은 일반적으로 프론트매터와 함께 사용되며, 보통 이는 마크다운 처리 방식에 약간의 추가 사용자 정의 처리를 추가하는 것을 의미합니다. 이를 해결하기 위해 `next-mdx-remote`는 프론트매터의 선택적 파싱을 제공하며, `serialize`에 `parseFrontmatter: true`를 전달하여 활성화할 수 있습니다.

다음은 그 모습입니다:

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
  // MDX 텍스트 - 로컬 파일, 데이터베이스, 어디서든 가져올 수 있습니다
  const source = `---
title: Test
---

Some **mdx** text, with a component <Test name={frontmatter.title}/>
  `

  const mdxSource = await serialize(source, { parseFrontmatter: true })
  return { props: { mdxSource } }
}
```

_프론트매터를 파싱하기 위해 [`vfile-matter`](https://github.com/vfile/vfile-matter)가 사용됩니다._

</details>

<details>
  <summary>`scope`를 사용하여 컴포넌트에 사용자 정의 데이터 전달하기</summary>

`<MDXRemote />`는 `scope` prop을 받아들이며, 이는 MDX에서 사용할 수 있는 모든 값을 제공합니다.

`scope` 인수의 각 키/값 쌍은 JavaScript 변수로 노출됩니다. 예를 들어, `{ foo: 'bar' }`와 같은 scope가 있다면, `const foo = 'bar'`로 해석될 것입니다.

이는 특히 `scope` 인수의 키 이름이 유효한 JavaScript 변수 이름이어야 함을 의미합니다. 예를 들어, `{ 'my-variable-name': 'bar' }`를 전달하면 키 이름이 유효한 JavaScript 변수 이름이 아니기 때문에 _오류_ 가 발생합니다.

또한 `scope` 변수는 _컴포넌트의 인수_ 로 소비되어야 하며, 텍스트 중간에 렌더링될 수 없다는 점에 주의해야 합니다. 이는 아래 예제에서 보여집니다.

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
  // MDX 텍스트 - 로컬 파일, 데이터베이스, 어디서든 가져올 수 있습니다
  const source =
    'Some **mdx** text, with a component using a scope variable <Test product={product} />'
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

</details>

<details>
  <summary>대신 `serialize` 함수에 `scope` 전달하기</summary>

`serialize`에 사용자 정의 데이터를 전달할 수도 있으며, 이는 값을 통과시키고 결과에서 사용 가능하게 만듭니다. `source`의 결과를 `<MDXRemote />`에 전개하면 데이터를 사용할 수 있게 됩니다.

`serialize`에 전달된 모든 scope 값은 직렬화 가능해야 합니다. 즉, 함수나 컴포넌트를 전달하는 것은 불가능합니다. 또한 `scope` 인수에 명명된 모든 키는 유효한 JavaScript 변수 이름이어야 합니다. 직렬화할 수 없는 사용자 정의 scope를 전달해야 하는 경우, 렌더링되는 곳에서 `<MDXRemote />`에 직접 `scope`를 전달할 수 있습니다. 이에 대한 예제는 이 섹션 위에 있습니다.

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
  // MDX 텍스트 - 로컬 파일, 데이터베이스, 어디서든 가져올 수 있습니다
  const source =
    'Some **mdx** text, with a component <Test product={product} />'
  const mdxSource = await serialize(source, { scope: data })
  return { props: { source: mdxSource } }
}
```

</details>

<details>
  <summary>
    <code>MDXProvider</code>의 사용자 정의 컴포넌트<a id="mdx-provider"></a>
  </summary>

애플리케이션에서 렌더링되는 모든 `<MDXRemote />`에 컴포넌트를 사용할 수 있게 하려면 `@mdx-js/react`의 [`<MDXProvider />`](https://mdxjs.com/docs/using-mdx/#mdx-provider)를 사용할 수 있습니다.

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
  // MDX 텍스트 - 로컬 파일, 데이터베이스, 어디서든 가져올 수 있습니다
  const source = 'Some **mdx** text, with a component <Test />'
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

</details>

<details>
  <summary>
    점이 포함된 컴포넌트 이름 (예: <code>motion.div</code>)
  </summary>

`framer-motion`과 같이 점(`.`)이 포함된 컴포넌트 이름은 다른 사용자 정의 컴포넌트와 동일한 방식으로 렌더링할 수 있습니다. 컴포넌트 객체에 `motion`을 전달하기만 하면 됩니다.

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
  // MDX 텍스트 - 로컬 파일, 데이터베이스, 어디서든 가져올 수 있습니다
  const source = `Some **mdx** text, with a component:

<motion.div animate={{ x: 100 }} />`
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

</details>

<details>
  <summary>지연 하이드레이션</summary>

지연 하이드레이션은 클라이언트에서 컴포넌트의 하이드레이션을 지연시킵니다. 이는 애플리케이션의 초기 로드를 개선하기 위한 최적화 기술이지만, MDX 콘텐츠 내의 동적 콘텐츠의 상호 작용성에 예상치 못한 지연을 초래할 수 있습니다.

_참고: 이는 렌더링된 MDX 주위에 추가적인 `div` 래퍼를 추가합니다. 이는 [렌더링 중 하이드레이션 불일치](https://reactjs.org/docs/react-dom.html#hydrate)를 피하기 위해 필요합니다._

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
  // MDX 텍스트 - 로컬 파일, 데이터베이스, 어디서든 가져올 수 있습니다
  const source = 'Some **mdx** text, with a component <Test />'
  const mdxSource = await serialize(source)
  return { props: { source: mdxSource } }
}
```

</details>

## API

이 라이브러리는 `serialize`와 `<MDXRemote />` 함수와 컴포넌트를 노출합니다. 이 두 가지는 의도적으로 자체 파일로 분리되어 있습니다. `serialize`는 **서버 사이드**에서 실행되도록 의도되었으므로, 서버/빌드 시간에 실행되는 `getStaticProps` 내에서 실행됩니다. 반면 `<MDXRemote />`는 클라이언트 사이드, 즉 브라우저에서 실행되도록 의도되었습니다.

- **`serialize(source: string, { mdxOptions?: object, scope?: object, parseFrontmatter?: boolean })`**

  **`serialize`** 는 MDX 문자열을 소비합니다. 선택적으로 [MDX에 직접 전달되는](https://mdxjs.com/docs/extending-mdx/) 옵션과 MDX 스코프에 포함될 수 있는 스코프 객체를 전달할 수도 있습니다. 이 함수는 `<MDXRemote />`에 직접 전달되도록 의도된 객체를 반환합니다.

  ```ts
  serialize(
    // 문자열로 된 원시 MDX 내용
    '# hello, world',
    // 선택적 매개변수
    {
      // 모든 사용자 정의 MDX 컴포넌트의 인수로 사용 가능
      scope: {},
      // MDX의 사용 가능한 옵션, 자세한 정보는 MDX 문서를 참조하세요.
      // https://mdxjs.com/packages/mdx/#compilefile-options
      mdxOptions: {
        remarkPlugins: [],
        rehypePlugins: [],
        format: 'mdx',
      },
      // MDX 소스에서 프론트매터를 파싱할지 여부를 나타냅니다
      parseFrontmatter: false,
    }
  )
  ```

  사용 가능한 `mdxOptions`에 대해서는 <https://mdxjs.com/packages/mdx/#compilefile-options>를 방문하세요.

- **`<MDXRemote compiledSource={string} components?={object} scope?={object} lazy?={boolean} />`**

  **`<MDXRemote />`** 는 `serialize`의 출력과 선택적인 components 인수를 소비합니다. 그 결과는 컴포넌트에 직접 렌더링될 수 있습니다. 콘텐츠의 하이드레이션을 지연시키고 정적 마크업을 즉시 제공하려면 `lazy` prop을 전달하세요.

  ```ts
  <MDXRemote {...source} components={components} />
  ```

### 기본 컴포넌트 교체하기

렌더링은 내부적으로 [`MDXProvider`](https://mdxjs.com/docs/using-mdx/#mdx-provider)를 사용합니다. 이는 HTML 태그를 사용자 정의 컴포넌트로 교체할 수 있음을 의미합니다. 이러한 컴포넌트는 MDXJS의 [컴포넌트 테이블](https://mdxjs.com/table-of-components/)에 나열되어 있습니다.

사용 사례의 예는 선호하는 스타일링 라이브러리로 콘텐츠를 렌더링하는 것입니다.

```jsx
import { Typography } from "@material-ui/core";

const components = { Test, h2: (props) => <Typography variant="h2" {...props} /> }
...
```

원하는 경우, `<MDXRemote />`에 직접 컴포넌트를 전달하는 대신 전체 애플리케이션을 `<MDXProvider />`로 감쌀 수도 있습니다. 위의 [예제](#mdx-provider)를 참조하세요.

참고: 컴포넌트 이름에 "/"가 있기 때문에 `th/td`는 작동하지 않습니다.

## 배경 & 이론

Next.js 앱에서 MDX 파일을 로드하는 좋은 기본 방법이 없었습니다. 이전에 우리는 MDX 파일을 레이아웃으로 렌더링하고 프론트매터를 가져와 인덱스 페이지를 만들 수 있도록 [`next-mdx-enhanced`](https://github.com/hashicorp/next-mdx-enhanced)를 작성했습니다.

`next-mdx-enhanced`의 이 워크플로우는 괜찮았지만, `next-mdx-remote`에서 제거한 몇 가지 제한 사항이 있었습니다:

- **파일 내용이 로컬이어야 합니다.** MDX 파일을 다른 repo나 데이터베이스 등에 저장할 수 없습니다. 충분히 큰 운영의 경우, 콘텐츠를 작성하는 사람과 콘텐츠의 프레젠테이션을 작업하는 사람 사이에 분리가 있게 될 것입니다. 같은 repo에서 이 두 가지 관심사를 겹치게 하면 모든 사람에게 더 어려운 워크플로우가 됩니다.
- **파일 시스템 기반 라우팅에 묶여 있습니다.** 페이지는 위치에 따라 URL로 생성됩니다. 또는 `exportPathMap`을 사용하여 재매핑할 수 있지만, 이는 작성자에게 혼란을 줍니다. 어쨌든 페이지를 어떤 식으로든 이동하면 문제가 발생합니다. 페이지의 URL이나 `exportPathMap` 구성이 깨집니다.
- **결국 성능 문제에 부딪힐 것입니다.** Webpack은 JavaScript 번들러이며, 수백/수천 페이지의 텍스트 콘텐츠를 강제로 로드하면 메모리 요구 사항이 급격히 증가할 것입니다. Webpack은 각 페이지를 많은 메타데이터가 있는 별개의 객체로 저장합니다. 수백 개의 페이지가 있는 우리의 구현 중 하나는 사이트를 컴파일하는 데 8GB 이상의 메모리가 필요했습니다. 빌드에 25분 이상 걸렸습니다.
- **관계형 데이터를 구조화하는 방법이 제한될 것입니다.** 전체 데이터 구조가 JavaScript 객체로 파싱된 프론트매터이고 메모리에 유지되는 경우, 콘텐츠를 동적이고 관련된 카테고리로 구성하는 것은 어렵습니다.

따라서 `next-mdx-remote`는 전체 패턴을 변경하여 MDX 콘텐츠를 import를 통해 로드하는 것이 아니라 `getStaticProps` 또는 `getServerProps`를 통해 로드합니다. 다른 데이터를 로드하는 것과 같은 방식으로 말이죠. 이 라이브러리는 MDX 콘텐츠를 성능이 좋은 방식으로 직렬화하고 하이드레이션하는 도구를 제공합니다. 이는 위에 나열된 모든 제한 사항을 제거하며, 상당히 낮은 비용으로 이를 수행합니다 . `next-mdx-enhanced`는 많은 사용자 정의 로직과 [일부 성가신 제한 사항](https://github.com/hashicorp/next-mdx-enhanced/issues/17)이 있는 매우 무거운 라이브러리입니다. 우리의 비공식적인 테스트에서는 빌드 시간이 50% 이상 감소한 것으로 나타났습니다.

이 프로젝트가 처음 만들어진 이후, Kent C. Dodds가 비슷한 프로젝트인 [`mdx-bundler`](https://github.com/kentcdodds/mdx-bundler)를 만들었습니다. 이 라이브러리는 MDX 파일 내에서 import와 export를 지원하며(각 가져온 파일의 내용을 수동으로 읽고 전달하는 한), 자동으로 프론트매터를 처리합니다. 모든 페이지에서 다른 컴포넌트를 가져와 사용하는 많은 파일이 있는 경우 `mdx-bundler`를 사용하는 것이 도움이 될 수 있습니다. 현재 `next-mdx-remote`는 모든 페이지에서 사용 가능한 컴포넌트만 가져오고 사용할 수 있도록 허용합니다. 하지만 이 기능에는 비용이 따른다는 점에 유의해야 합니다. 기본 마크다운 콘텐츠의 경우 `mdx-bundler`의 출력은 `next-mdx-remote`의 출력보다 최소 400% 더 큽니다.

### 이걸로 어떻게 블로그를 만들 수 있나요?

데이터에 따르면 모든 개발자 도구 사용 사례의 99%가 불필요하게 복잡한 개인 블로그를 만드는 것입니다. 농담입니다. 하지만 진지하게, 개인 또는 소규모 비즈니스용 블로그를 만들려고 한다면 일반 HTML과 CSS를 사용하는 것을 고려해보세요. 간단한 블로그를 만드는 데 무거운 풀 스택 JavaScript 프레임워크를 사용할 필요는 절대 없습니다. 몇 년 후에 업데이트하러 돌아왔을 때 모든 의존성에 10번의 주요 변경 사항이 없었다면 감사할 것입니다.

하지만 정말 고집한다면, [우리의 공식 Next.js 예제 구현](https://github.com/vercel/next.js/tree/canary/examples/with-mdx-remote)을 확인해보세요. 💖

## 주의 사항

### 환경 대상

`next-mdx-remote`에 의해 생성된 코드는 모듈을 지원하는 브라우저를 대상으로 합니다. 더 오래된 브라우저를 지원해야 한다면, `serialize`의 `compiledSource` 출력을 트랜스파일하는 것을 고려해보세요.

### `import` / `export`

`import`와 `export` 문은 MDX 파일 **내부**에서 사용할 수 없습니다. MDX 파일에서 컴포넌트를 사용해야 한다면, `<MDXRemote />`에 prop으로 제공해야 합니다.

이는 이해가 될 것입니다. import가 작동하려면 파일 경로에 상대적이어야 하는데, 이 라이브러리는 설정된 파일 경로에서만 로컬 콘텐츠를 로드하는 대신 어디서든 콘텐츠를 로드할 수 있게 해주기 때문입니다. export의 경우, MDX 콘텐츠는 **모듈**이 아닌 데이터로 취급되므로, `next-mdx-remote`에 전달된 MDX에서 export될 수 있는 값에 접근할 방법이 없습니다.

## 보안

이 라이브러리는 클라이언트 측에서 JavaScript 문자열을 평가하여 MDX 콘텐츠를 원격으로 처리합니다. JavaScript를 문자열로 평가하는 것은 주의 깊게 수행하지 않으면 위험한 방식이 될 수 있습니다. XSS 공격을 가능하게 할 수 있기 때문입니다. 문서에 지시된 대로 `serialize` 함수에 의해 생성된 `mdxSource` 입력만 `<MDXRemote />`에 전달하는 것이 중요합니다. **사용자 입력을 `<MDXRemote />`에 직접 전달하지 마세요.**

웹사이트에 `eval` 또는 `new Function()`을 통한 코드 평가를 허용하지 않는 CSP가 있다면, `next-mdx-remote`를 사용하기 위해 그 제한을 완화해야 합니다. 이는 [`unsafe-eval`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src#common_sources)을 사용하여 수행할 수 있습니다.

## TypeScript

이 프로젝트는 TypeScript 사용을 위한 네이티브 타입을 포함하고 있습니다. `serialize`와 `<MDXRemote />` 모두 예상대로 일반적으로 타입이 지정되어 있으며, 라이브러리는 `getStaticProps`의 결과를 타입 지정하는 데 사용할 수 있는 타입도 export합니다.

- `MDXRemoteSerializeResult<TScope = Record<string, unknown>>`: `serialize`의 반환 값을 나타냅니다. `TScope` 제네릭 타입을 전달하여 전달한 스코프 데이터의 타입을 나타낼 수 있습니다.

다음은 TypeScript에서 간단한 구현 예시입니다. TypeScript의 모든 구성에 대해 정확히 이런 방식으로 타입을 구현할 필요는 없을 수 있습니다. 이 예제는 단지 필요한 경우 타입을 어디에 적용할 수 있는지 보여주는 데모일 뿐입니다.

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

## React 서버 컴포넌트(RSC) & Next.js `app` 디렉토리 지원

서버 컴포넌트, 특히 Next.js의 `app` 디렉토리 내에서 `next-mdx-remote`를 사용하는 것은 `next-mdx-remote/rsc`에서 import하여 지원됩니다. 이전에는 직렬화와 렌더링 단계가 분리되어 있었지만, 앞으로 RSC는 이러한 분리를 불필요하게 만듭니다.

주목할 만한 몇 가지 차이점:

- `<MDXRemote />`는 이제 `next-mdx-remote/serialize`의 직렬화된 출력을 받아들이는 대신 `source` prop을 받아들입니다.
- 사용자 정의 컴포넌트는 더 이상 `@mdx-js/react`의 `MDXProvider` 컨텍스트를 사용하여 제공할 수 없습니다. RSC는 React Context를 지원하지 않기 때문입니다.
- `parseFrontmatter: true`를 전달할 때 MDX 외부에서 프론트매터에 접근하려면 `next-mdx-remote/rsc`에서 노출된 `compileMdx` 메서드를 사용하세요.
- `lazy` prop은 더 이상 지원되지 않습니다. 렌더링이 서버에서 발생하기 때문입니다.
- `<MDXRemote />`는 이제 비동기 컴포넌트이기 때문에 서버에서 렌더링되어야 합니다. 클라이언트 컴포넌트는 MDX 마크업의 일부로 렌더링될 수 있습니다.

RSC에 대한 자세한 정보는 [Next.js 문서](https://nextjs.org/docs/app/building-your-application/rendering/server-components)를 확인하세요.

### 예제

_Next.js 13+ 애플리케이션에서 `app` 디렉토리를 사용한다고 가정합니다._

#### 기본

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

#### 로딩 상태

```tsx
import { MDXRemote } from 'next-mdx-remote/rsc'

// app/page.js
export default function Home() {
  return (
    // 이상적으로 이 로딩 스피너는 레이아웃 시프트가 없도록 보장해야 합니다.
    // 이는 그러한 로딩 스피너를 제공하는 방법의 예시입니다.
    // Next.js에서는 이를 위해 `loading.js`를 사용할 수도 있습니다.
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

#### 사용자 정의 컴포넌트

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
      // h1은 이제 `large-text` 클래스 이름으로 렌더링됩니다
      source={`# Hello World
      This is from Server Components!
    `}
    />
  )
}
```

#### MDX 외부에서 프론트매터에 접근하기

```tsx
// app/page.js
import { compileMDX } from 'next-mdx-remote/rsc'

export default async function Home() {
  // 선택적으로 프론트매터 객체에 대한 타입을 제공합니다
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

## 대안

`next-mdx-remote`는 지원하는 기능에 대해 의견이 있습니다. `next-mdx-remote`에서 제공하지 않는 추가 기능이 필요한 경우, 고려할 수 있는 몇 가지 대안이 있습니다:

- [`mdx-bundler`](https://github.com/kentcdodds/mdx-bundler)
- [`next-mdx-remote-client`](https://github.com/ipikuka/next-mdx-remote-client)
- [`remote-mdx`](https://github.com/devjiwonchoi/remote-mdx)

### `next-mdx-remote`가 필요하지 않을 수도 있습니다

React 서버 컴포넌트를 사용하고 있고 단순히 사용자 정의 컴포넌트와 함께 기본 MDX를 사용하려고 한다면, 코어 MDX 라이브러리 외에 다른 것은 필요하지 않습니다.

```js
import { compile, run } from '@mdx-js/mdx'
import * as runtime from 'react/jsx-runtime'
import ClientComponent from './components/client'

// MDX는 파일이나 데이터베이스 등 어디서든 가져올 수 있습니다.
const mdxSource = `# Hello, world!
<ClientComponent />
`

export default async function Page() {
  // MDX 소스 코드를 함수 본문으로 컴파일합니다
  const code = String(
    await compile(mdxSource, { outputFormat: 'function-body' })
  )
  // 그런 다음 코드를 서버에서 실행하여 서버 컴포넌트를 생성하거나,
  // 최종 렌더링을 위해 문자열을 클라이언트 컴포넌트에 전달할 수 있습니다.

  // 런타임과 함께 컴파일된 코드를 실행하고 기본 export를 가져옵니다
  const { default: MDXContent } = await run(code, {
    ...runtime,
    baseUrl: import.meta.url,
  })

  // MDX 콘텐츠를 렌더링하고, ClientComponent를 컴포넌트로 제공합니다
  return <MDXContent components={{ ClientComponent }} />
}
```

컴파일된 문자열을 데이터베이스나 클라이언트 컴포넌트에 전달할 계획이 없다면, `evaluate`를 사용하여 이 접근 방식을 단순화할 수도 있습니다. `evaluate`는 단일 호출로 코드를 컴파일하고 실행합니다.

```js
import { evaluate } from '@mdx-js/mdx'
import * as runtime from 'react/jsx-runtime'
import ClientComponent from './components/client'

// MDX는 파일이나 데이터베이스 등 어디서든 가져올 수 있습니다.
const mdxSource = `
export const title = "MDX Export Demo";

# Hello, world!
<ClientComponent />

export function MDXDefinedComponent() {
  return <p>MDX-defined component</p>;
}
`

export default async function Page() {
  // 컴파일된 코드 실행
  const {
    default: MDXContent,
    MDXDefinedComponent,
    ...rest
  } = await evaluate(mdxSource, runtime)

  console.log(rest) // { title: 'MDX Export Demo' } 로그

  // MDX 콘텐츠를 렌더링하고, ClientComponent를 컴포넌트로 제공하며,
  // export된 MDXDefinedComponent를 렌더링합니다.
  return (
    <>
      <MDXContent components={{ ClientComponent }} />
      <MDXDefinedComponent />
    </>
  )
}
```

## 라이선스

[Mozilla Public License Version 2.0](./LICENSE)