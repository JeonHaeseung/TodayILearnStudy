# TIL - React의 동작 방식, TS, 가상 돔

## React의 동작 방식

어플리케이션의 진입점은 보통 index.html 파일에서 정의된다. 이 파일에 React 애플리케이션이 렌더링될 DOM 요소가 포함된다. 여기에 포함되는 jsx 파일이 바로 진입점이 된다고 할 수 있다.

```html
<!doctype html>
<html lang="en">

<head>
  <meta charset="UTF-8" />
  <link rel="icon" type="image/svg+xml" href="/vite.svg" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Hello-React</title>
</head>

<body>
  <div id="root"></div> <!-- 이 요소가 React 애플리케이션의 진입점 -->
  <script type="module" src="/src/qr-generator/main.jsx"></script>
</body>

</html>
```

이때 React 컴포넌트를 직접 `<script>` 태그로 로드하면, React의 렌더링 로직이 정상적으로 작동하지 않을 수 있다. React 컴포넌트는 일반적으로 `ReactDOM.render`를 사용하여 HTML에 마운트된다. 다음과 같이 `main.jsx`를 추가하고, 여기에서 마운트를 시켜야 한다.

```javascript
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import QrCodeGenerator from './QrCodeGenerator.jsx'

createRoot(document.getElementById('root')).render(
    <StrictMode>
        <QrCodeGenerator />
    </StrictMode>,
)
```
## Cora의 강의 - 가상 돔, TS, NEXT.js

프론트엔드가 정말 너무 처음인 나머지(기존에는 바닐라 JS만 경험해봤다), 리엑트, TypeScript, NEXT.js의 차이마저도 모르겠어서 천재 프론트엔드 친구인 Cora에게 카카오톡으로 물어봤다. 그러자 구글 미트를 켜서 속성 강의를 해주었는데, 아래는 그 내용이다(참고로 이해를 위해서 간단하게 설명한 것이지, 이 내용을 그대로 면접에서 대답하면 절대 안된다고 했다).

원래 바닐라 JS를 사용하면 화면 하나당 하나의 HTML이 있고, 그 HTML 마다 DOM이 있다. 

그런데 리엑트는 `index.html` 하나만 있고, 이 HTML에 하나의 돔만 있다. 그 이유는 리엑트는 **가상 돔**이 있어서 그게 하나의 페이지인 것처럼 역할을 하기 때문이다. 즉, HTML은 하나고, 그 하나에 어떤 컴포넌트를 보여줄지를 결정해서 해당 컴포넌트를 불러와 랜더링한다. 페이지도 하나의 컴포넌트라고 볼 수 있으며, 단지 `App.jsx`에서 `Route`로 페이지 전환을 한다고 보면 된다.

이런 가상 돔이라는 특징 때문에 더 무거워지는 단점이 있다. 가상 돔은 개발할 때 편리한 것이지, 효율적인 구조는 아니다. 기존의 바닐라 JS는 모든 HTML, JS 파일이 브라우저에 있고, 그 파일들을 전환하면서 보여주는 것이라면, 리엑트는 가상 돔이기 때문에 "~상황에서는 ~ 컴포넌트를 보여준다"라는 로직이 있어서 필요할 때마다 랜더링하기 때문에 효율적이지 못하다. 이런 이유 때문에 앞으로는 가상 돔 구조를 사용하지 않는 프레임워크가 대세가 될 것이라는 의견도 있다. 

타입 스크립트는 일종의 JS의 서브셋이라고 보면 된다. React의 구조를 그대로 유지하면서 TS를 사용할 수 있다. 기존의 JS에서 타입 검사하는 로직만 추가되었다고 보면 된다. (타입을 명시하지 않으면 run할 때 에러가 발생해서 돌릴 수 없다.) 타입 스크립트 프로젝트에서는 모든 jsx 파일이 tsx 파일이 된다. 타입 스크립트에서 타입이란 우리가 기본적으로 프로그래밍 언어에서 아는 것처럼 Int, Boolean 값 등이 될 수도 있지만, 직접 타입을 명시할 수도 있다.

```typescript
export interface IStoreInfoView {
	storeId: number;
	meanRating: number;
}
```

NEXT.js는 리엑트와 다른 프레임워크인데, 거의 비슷하게 사용할 수는 있다. 다만 리엑트는 브라우저에서 모두 랜더링하는 구조라면, NEXT는 어느 정도 웹서버에서 랜더링되어서 도착한다. 이 때문에 `console.log`를 찍으면 리엑트는 브라우저에서 보이지만, NEXT는 터미널에서 보인다.
