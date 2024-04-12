---
title: 모바일 웹뷰에서 input 자동 focus방법
date: 2024-04-12 14:30:00 +09:00
categories: [프론트엔드]
tags: [트러블슈팅, 웹뷰]
---

# React 모바일 웹뷰에서 react로 input 자동 focus

모바일 환경에서 웹뷰를 띄워 검색버튼을 누르면 자동으로 input field에 focus가 가도록하여 os별 키보드 노출을 구현하고 싶었다.
기존 웹 환경이었다면 useRef를 사용하여 로그인 화면일때 자동으로 id 입력창에 커서가 가게끔 했던 기억이 있어서 해당 방법으로 시도하였지만, 실패하였다.
실패한 방법과 매우 간단하지만 성공한 방법을 소개하겠다.

## 실패한 방법

### useRef로 포커스 설정하기

```ts
import React, { useEffect, useRef } from 'react';

...

export const Search = () => {
  const dispatch = useDispatch();
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    if (inputRef.current) {
      inputRef.current.focus();
    }
  }, []);

  return (
    <StyledWrapper>
      <div className="search">
        <div className="input-field">
          <img src="/images/icon/shopping/ic-search-gray@2x.png" alt="검색" />
          <input
            ref={inputRef}
            type="text"
            placeholder="카테고리 내 검색"
            onChange={(e) => dispatch(setSearchFilter(e.target.value))}
          />

...
```

처음 시도했던 방식은 useRef로 포커스를 주는 방식이었다.
useRef를 사용하면 특정 DOM요소를 직접 참조할 수 있다.
그래서 의존성 배열에 빈배열을 주고 해당 컴포넌트가 렌더링될때 바로 input에 focus를 주게끔 하여 기술적으론 오류가 없다고 생각하였다.
그러나 모바일 환경에서는 사용자가 명시적인 액션을 취하지 않는 한 키보드가 자동으로 나타나지 않게 끔 설계가 되어있었다

## 해결

### autoFocus 속성사용하기

생각보다 해결법은 간단했다

```ts
<input autoFocus type="text" placeholder="카테고리 내 검색" />
```

HTML에서 autoFocus 속성을 사용하여 컴포넌트 렌더링시 자동으로 포커스를 맞추는 기능을 제공하고있었다. 이 방법이 useRef보다 더 관대하게 허용하기 때문에 모바일 환경에서 키보드를 나타나게 할 수 있었다.
