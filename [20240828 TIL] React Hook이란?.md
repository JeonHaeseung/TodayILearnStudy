# TIL - React Hook이란?

## React Hook은 무엇일까
그렇다, 난 여기서부터 모른다. [공식문서](https://react.dev/reference/react/hooks)에 따르면 Hooks를 사용하면 컴포넌트에서 다양한 React 기능을 사용할 수 있으며, 내장된 Hooks를 사용하거나 Hooks를 결합하여 직접 빌드할 수 있다고 한다. 여전히 훅이 무엇인지는 설명해주고 있지 않다. [Reddit](https://www.reddit.com/r/react/comments/11ftu0p/what_are_hooks/)에 따르면 React Hook은 React의 내부 메모리에 접근할 수 있는 함수로, 값과 그 값을 수정하는 메소드에 접근 가능하다고 한다. 우리가 함수를 사용하는 바로 그 이유들(기능 분할 및 아웃소싱, 코드 간편화 등) 때문에 리엑트 훅으로 분리한 것이다. 레딧이 공식 문서보다 친절하다.

## 주요 React Hook들

### useState

리엑트에는 다양한 내장 hook들이 존재한다. 첫 번째는 사용자 입력 정보 등을 기억하고 상태(state)를 관리하는 `useState`이다. 아래 코드에서,`count`는 상태값을, `useState(0)`는 상태값에 대한 초기값을, `setCount`는 상태값을 업데이트하는 함수이다. `setCount`가 호출될 때마다 컴포넌트는 새로운 상태값으로 re-render한다. 소셜 미디어에서 '좋아요'를 누를 때마다 숫자가 올라가는 기능에서 사용된다고 보면 된다.

```javascript
//useState
const [count, setCount] = useState(0);
```

### useReducer

`useReducer`는 reducer 함수 내부에 업데이트 로직을 포함하는 상태 변수를 선언한다. `useState`를 사용한 복잡한 상태 관리 대신, dispatched action(예시: `dispatch({type: 'INCREMENT'})`)에 따라서 어떻게 상태가 변화하는지 정의내린 `counterReducer` 함수를 통해서 변경 가능하다. 예를 들어, 다양한 값을 입력하는 multi-step form의 경우에는 프로그래스와 값을 적절하게 관리하기 어려운데, 이때 `useReducer`를 사용한다.

```javascript
const counterReducer = (state, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    case 'DECREMENT':
      return { count: state.count - 1 };
    default:
      return state;
  }
};

//useReducer
const [state, dispatch] = useReducer(counterReducer, { count: 0 });
```

### useEffect

`useEffect`는 컴포넌트가 외부 시스템에 연결하고 동기화할 수 있게 해준다. 공식 문서에 따르면, Effect는 React의 패러다임에서 벗어나는 방식이므로 외부 시스템과의 연동이 없으면 사용하지 않아도 된다고 한다. 아래의 `useEffect`는 `count`값이 변경되었고, render가 일어났을 때마다 호출된다. 현재의 `count` 값에 따라서 문서의 제목을 변경하는 역할을 한다. 블로그 서비스에서 컴포넌트 로딩 시 API 데이터가 도착하면 patch & display하는 기능을 생각해보면 좋을 것 같다.

```javascript
//useEffect
useEffect(() => {
  document title = 'Count is ${count}';
}, [count]);
```

### useContext

`useContext`를 통해서 컨텍스트를 사용하면 컴포넌트가 `props`로 전달하지 않고도 먼 부모로부터 정보를 받을 수 있다고 한다. 예를 들어, 앱의 최상위 컴포넌트는 아무리 깊더라도 현재 UI 테마를 아래의 모든 컴포넌트에 전달할 수 있다. 쉽게 말하자면 shared data를 만들어준다고 생각하면 된다. 예를 들어, 로그인한 유저의 디테일을 담아 놓은 global user object가 있다고 할 때, 사용자 페이지에서 해당 유저 이름을 보여주고 싶은 경우에 사용한다.

```javascript
//useContext
const user = useContext(UserContext);
```

### useMemo

`useMemo`는 re-rendering 과정에서 성능이 저하되는 것을 줄이기 위해 cache를 사용해서 이전에 계산된 값을 사용하도록 하는 것이다. 아래 코드에서는 캐시된 `slowFunction`의 결과를 사용하고 있다. `useMemo`를 사용하면 `count`값이 변경될 때만 `slowFunction`을 다시 계산하고, 그렇지 않으면 cached calculation을 다시 재사용해 비효율적인 재랜더링을 줄인다. 
```javascript
//useMemo
const expensiveValue = useMemo(() => slowFunction(count), [count]);
```

### useRef

`useRef`는 컴포넌트가 DOM Node나 timeout ID와 같이 렌더링에 사용되지 않는 정보를 보유 하도록 한다. Ref 또한 React의 패러다임에서 벗어나는 방식이고, 내장된 브라우저 API와 같이 React가 아닌 시스템에서 작업해야 할 때 유용하다고 한다. 이 방식은 DOM element의 직접적인 조종이 가능한 방식이다. 예를 들어서, `inputRef.currect.focus()`등을 통해서 직접 focus하는 것이 가능하다. 로그인 페이지가 로딩될 때 input form에서 email input field가 focus되도록 만들고 싶다면 이 방식을 쓸 수 있다. 이렇게 하면 마치 사용자가 해당 요소를 클릭하거나 탭 키를 눌러 포커스를 이동시킨 것과 비슷한 효과를 준다.
```javascript
//useContext
const inputRef = useRef(null);

<input ref={inputRef} />
```
