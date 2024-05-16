# React 19 소개

React 19 Beta 공개 기념 새로운 기능 및 개선점 소개

## Actions

**Pending, Error, Optimistic updates를 자동으로 처리하는 기능**

기존의 리액트에서는 비동기처리에서의 상태관리들을 직접 처리해야했음

```tsx
// 액션에서 대기 상태 사용하기
function UpdateName({}) {
  const [name, setName] = useState("");
  const [error, setError] = useState(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    startTransition(async () => {
      const error = await updateName(name);
      if (error) {
        setError(error);
        return;
      }
      redirect("/path");
    });
  };

  return (
    <div>
      <input value={name} onChange={(event) => setName(event.target.value)} />
      <button onClick={handleSubmit} disabled={isPending}>
        Update
      </button>
      {error && <p>{error}</p>}
    </div>
  );
}
```

트랜지션에서 **비동기 함수**를 사용하여 대기상태, 에러, 낙관적 업데이트를 자동으로 처리하는 기능이 추가됨

- Pending State : 요청이 시작될 때 함께 시작, 최종 상태 업데이트가 커밋되면 자동으로 종료됨
- Error State : 요청이 실패한 경우 에러바운더리에 걸림, Optimistic update로 수정된 값을 자동으로 원복
- Optimistic update : 새로운 훅인 `useOptimistic`을 지원함, 사용자가 요청을 제출할 때 즉각적인 응답을 제공

### useActionState

```TSX
// <form> 액션과 useActionState 사용
function ChangeName({ name, setName }) {
  const [state, submitAction, isPending] = useActionState(
    async (previousState, formData) => {
      const error = await updateName(formData.get("name"));
      if (error) {
        return error;
      }
      redirect("/path");
      return null;
    },
    null,
  );

  return (
    <form action={submitAction}>
      <input type="text" name="name" />
      <button type="submit" disabled={isPending}>Update</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

`useActionState`는 함수("액션")를 받아 호출할 래핑된 액션을 반환합니다. 이는 각 액션이 조합되므로 가능합니다. 래핑된 액션이 호출되면, `useActionState`는 액션의 마지막 결과를 data로, 액션의 대기 상태를 pending으로 반환합니다.

### \<form\> 액션

### useFormStatus

form에 대한 정보를 접근해야 할 때, 프로퍼티를 드릴링하지 않고 사용할 수 있는 `useFormStatus` 훅이 추가됨

부모 `<form>`의 상태를 읽음

```TSX
import { useFormStatus } from 'react-dom';

function DesignButton() {
  const { pending, data, method, action } = useFormStatus();
  return <button type="submit" disabled={pending} />
}
```

### useOptimistic

```TSX
function ChangeName({currentName, onUpdateName}) {
  const [optimisticName, setOptimisticName] = useOptimistic(currentName);

  const submitAction = async formData => {
    const newName = formData.get("name");
    setOptimisticName(newName);
    const updatedName = await updateName(newName);
    onUpdateName(updatedName);
  };

  return (
    <form action={submitAction}>
      <p>Your name is: {optimisticName}</p>
      <p>
        <label>Change Name:</label>
        <input
          type="text"
          name="name"
          disabled={currentName !== optimisticName}
        />
      </p>
    </form>
  );
}
```

`useOptimistic` 훅은 `updateName` 요청이 진행되면 `optimisticName` 값을 바로 보여줌.
업데이트가 종료되거나 실패하면 리액트는 자동으로 `currentName` 값으로 다시 전환

transition이나 form action 내에서만 사용 가능

### use

Promise또는 Context를 읽는 훅
`use`로 프로미스를 읽는 경우 프로미스가 리졸브 될 때까지 리액트는 일시 중단됨

```TSX
import {use} from 'react';

function Comments({commentsPromise}) {
  // `use`는 프로미스가 해결될 때까지 일시 중단됩니다.
  const comments = use(commentsPromise);
  return comments.map(comment => <p key={comment.id}>{comment}</p>);
}

function Page({commentsPromise}) {
  // Comments에서 `use`가 일시 중단되면,
  // 이 컴포넌트의 서스펜스 바운더리가 보여집니다.
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  )
}
```

주요 특징

- Suspense, Errorboundary와도 연동됨
- 조건문 내에서 또는 얼리 리턴되는 조건문 뒤에서도 사용 가능

## 개선된점

### ref

더이상 forwardRef를 사용하지 않아도 됨. 함수 컴포넌트에서 ref를 프로퍼티로 접근할 수 있게 되었음

### ref를 위한 클린업 함수

컴포넌트가 언마운트 되었을 때, 리액트는 ref 콜백에서 반환된 클린업 함수를 호출.

```TSX
<input
  ref={(ref) => {
    // ref가 생성됨

    // 추가된 사항: 요소가 DOM에서 제거되었을 때
    // ref를 재설정하는 클린업 함수를 반환합니다
    return () => {
      // ref 클린업
    };
  }}
/>
```

### 프로바이더가 된 \<Context\>

```TSX
const ThemeContext = createContext('');

function App({children}) {
  return (
    <ThemeContext value="dark">
      {children}
    </ThemeContext>
  );
}
```

`<Context.Provider>` 대신 `<Context>`를 프로바이더로서 렌더링할 수 있고, 향후에 `<Context.Provider>` 는 폐기될 예정

### 문서 메타데이터 지원

`<title>` `<link>` `<meta>`와 같은 메타데이터 태그들을 사용할 수 있도록 지원

```TSX
function BlogPost({post}) {
  return (
    <article>
      <h1>{post.title}</h1>
      <title>{post.title}</title>
      <meta name="author" content="Josh" />
      <link rel="author" href="https://twitter.com/joshcstory/" />
      <meta name="keywords" content={post.keywords} />
      <p>
        Eee equals em-see-squared...
      </p>
    </article>
  );
}

```

컴포넌트를 렌더링할 때 `<head>`섹션에 위치하도록 자동으로 호이스팅 처리

### 비동기 스크립트 지원

```TSX
function MyComponent() {
  return (
    <div>
      <script async={true} src="..." />
      Hello World
    </div>
  )
}
```

모든 렌더링 환경에서 여러 다른 컴포넌트에 의해 비동기 스크립트가 렌더링되더라도 리액트가 스크립트를 한 번만 로드하고 실행하므로 중복이 제거됩니다

## 느낀점

공부하면서 사실 React 18에 도입된 기능도 잘 안쓰는데 이거라고 쓰긴 할까 싶긴 했다.

비동기 API 상태관리를 추상화하는데 힘을 많이 준 것 같긴 하지만, 기존 코드로 작성하면 길긴해도 구현가능한 내용들이었고 완전 새로운 내용은 없었던것 같다. 개선된 점에서는 forwardRef나 script, 메타태그들 같이 내가 특히 처리하기 귀찮다고 생각했던 것들을 개선 사항을 내놓아서 제법 기특했다. 아직 베타버전이기에 당장 프로젝트에 마이그레이션 할 것은 아니니까 리액트 컴파일러가 나올때쯤 다시 한 번 쓰임새를 고민해봐야겠다.

## Reference

- https://react.dev/blog/2024/04/25/react-19
- https://velog.io/@typo/react-19-beta#usedeferredvalue-%EC%B4%88%EA%B9%83%EA%B0%92
- FEConf 2023 [A2] use 훅이 바꿀 리액트 비동기 처리의 미래 맛보기
- https://velog.io/@kimkanu/React-%ED%9B%85-useOptimistic
