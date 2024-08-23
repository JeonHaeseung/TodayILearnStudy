# TIL - Tailwind CSS 설치 및 개념

## Tailwind CSS란?

HTML와 CSS 파일을 별도로 두지 않고, HTML 코드 내에서 관리할 수 있도록 돕는다. 또한, 클래스 명에 대해서 고민할 필요가 없기 개발할 수 있도록 돕는다. 이게 가능한 이유는 Tailwind CSS 내부에 이미 정의된 유틸리티 클래스가 있는 Utility-First Fundamentals이기 때문이다. dark: 를 통해서 다크모드에 대한 스타일링 정의를 할 수 있고, md:등의 수식어로 빠르게 반응형을 만들 수도 있다.  `first-child, last-child, required, invalid, disabled` 등의 다양한 수식어 또한 적용 가능하다.

## Tailwind CSS 설치하기

난 패키지 매니저로 `pnpm`을 사용하고 있기 때문에 다음과 같이 설치해주었다. 여기서 (p)npm과 npx의 차이는 무엇일까?  `npm`은 **노드 패키지 관리자**로, javascript로 작성된 node.js의 모든 패키지와 모듈을 관리한다. NPX (Node Package eXecute)는  `npm 5.2.0` 버전 이상부터 `npm`을 설치 시 자동으로 `npx`가 설치되는 **npm package runner**로, 일회용 패키지처럼 사용된다.

```bash
pnpm install -D tailwindcss
npx tailwindcss init
```

![Install Tailwind CSS](https://github.com/user-attachments/assets/e19a6e13-a01a-4be7-a63a-b7c9acc4e94c)

설치가 완료되면 tailwind.config.js 파일이 자동 생성된다. 해당 파일 내용을 아래 코드로 변경해준다.

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

그리고 `./src/index.css` 파일에서 Tailwind의 레이어에 대한 `@tailwind` 지시문을 추가한다.

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## PostCSS 설치하기

처음 Tailwind를 사용하면 아래와 같은 에러가 뜬다.

![Tailwind CSS error](https://github.com/user-attachments/assets/2948c898-ac8c-4db6-abeb-ba73866616f9)

**PostCSS Language Support** 확장 프로그램 및 모듈을 설치해 문제를 해결한다.

```bash
pnpm i -D postcss autoprefixer 
```

![Install PostCSS Language Support](https://github.com/user-attachments/assets/7c307d60-1465-4151-abe8-f6a02f9e71e4)

그 후 `postcss.config.cjs` 파일을 프로젝트 루트에 만들어준다.

```javascript
module.exports = {
    plugins: {
        tailwindcss: {},
        autoprefixer: {},
    },
};
```

참고로, `package.json` 파일이 현재 `"type": "module"` 이므로 `postcss.config.js`식의 파일 확장자를 쓰면 에러가 난다. `"type": "module"`이 설정되면, Node.js는 `.js` 파일을 ES 모듈로 해석하기 때문이다. 따라서 ES 모듈 문법(import, export)을 사용할 수 있다. `postcss.config` 파일이 `CommonJS` 문법을 사용하고 있기 때문에 `.cjs` 형식이어야 한다.

```javascript
{
  "type": "module"
}
```

## 참고 자료
- [Tailwind CSS 컨셉](https://velog.io/@94applekoo/Tailwind-CSS-1.%ED%95%B5%EC%8B%AC-%EC%BB%A8%EC%85%89)
- [Tailwind CSS 장점](https://wonny.space/writing/dev/hello-tailwind-css)
- [NPM과 NPX 차이](https://youngmin.hashnode.dev/npm-npx)
