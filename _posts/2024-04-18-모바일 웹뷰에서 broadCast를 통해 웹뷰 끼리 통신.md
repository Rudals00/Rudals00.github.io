---
title: 모바일 웹뷰에서 broadCast를 통해 웹뷰 끼리 통신
date: 2024-04-18 12:30:00 +09:00
categories: [프론트엔드]
tags: [트러블슈팅, 웹뷰]
---

# React로 구현된 모바일 웹뷰에서 broadCast로 통신

하나의 웹뷰에서 행동이 다른 웹뷰에 어떠한 영향을 미칠 때, 이를 효과적으로 감지하고 반응하는 것은 중요하다. 이러한 문제를 해결하기 위해 BroadcastChannel API를 사용할 수 있다.

## 문제상황

문제 상황은 다음과 같았다
window.location.href = ${앱스킴URL} 을 통해서 쿠폰함에서 쿠폰함 상세페이지 웹뷰를 새로 띄우고 쿠폰을 사용한 후 쿠폰함 상세페이지를 닫으면, 쿠폰함에서 refetch를 해줘야하는 상황이었다. 그러나 쿠폰함 상세페이지 웹뷰가 닫혔을 때를 쿠폰함에서 감지할 . 수없어서 refetch함수를 실행시켜줄 방법이 없었다.
당장 떠올릴 수 있는 가능한 방법은

- 쿠폰 상세페이지 웹뷰를 띄울때 쿠폰함 웹뷰를 닫아주었다가 쿠폰함 상세페이지를 나왔을때 다시 쿠폰함을 들어가게하여 refetch를 하는방법
  밖에 떠오르지 않았다

그러나 이러면 ux 측면에서 매우 좋지 않다고 판단하여 찾아보다가 발견한게 broadcast를 통해 웹뷰간 통신을 할 수 있다는 것이었다

## BroadCast를 사용하기 위한 커스텀 훅

```ts
import { useRef, useEffect, useCallback } from "react";
import { BroadcastChannel, BroadcastChannelOptions } from "broadcast-channel";

export interface UseBroadcastChannelOptions
  extends Omit<BroadcastChannelOptions, "node"> {}

export function useBroadcastChannel<T>(
  channelName: string,
  onMessage: (message: T) => void,
  options?: UseBroadcastChannelOptions
) {
  const broadcastChannelRef = useRef<BroadcastChannel<T> | null>(null);
  const onMessageRef = useRef<((message: T) => void) | null>(null);

  const handlePostMessage = useCallback((message: T) => {
    if (broadcastChannelRef.current) {
      broadcastChannelRef.current.postMessage(message);
    }
  }, []);

  useEffect(() => {
    onMessageRef.current = onMessage;
  }, [onMessage]);

  useEffect(() => {
    let mounted = true;
    const channel = new BroadcastChannel<T>(channelName, options);

    const handleMessage = (message: T) => {
      if (!mounted) {
        return;
      }
      onMessageRef.current?.(message);
    };

    channel.onmessage = handleMessage;
    broadcastChannelRef.current = channel;

    return () => {
      mounted = false;
      broadcastChannelRef.current = null;
      channel.close();
    };
  }, [channelName, options]);

  return { postMessage: handlePostMessage };
}
```

위 코드에서 useBroadcastChannel 훅은 두 가지 주요 기능을 제공한다. 첫째로 BroadcastChannel 인스턴스를 생성하여 매개변수로 준 channel 이름에 대한 메시지 교환 채널을 설정한다. 두번째로 onMessage 콜백을 통해 채널로부터 메시지를 수신하고 반응한다. 이렇게 함으로써, 다른 웹뷰에서 발생한 이벤트에 대해 즉시 반응하고 필요한 데이터를 재요청할 수 있다.

## 신호를 보내는쪽 컴포넌트

```ts
...
import { useBroadcastChannel } from 'constants/useBroadcastChannel';

...

const CouponHeader = () => {
  const { postMessage } = useBroadcastChannel(
    'refetchCouponListChannel',
    () => {}
  );

  const app = useApp();

  const onCloseEvent = () => {
    postMessage('refetchCouponList');
    closeWebview();
  };
...

```

해당 컴포넌트가 될때 신호를 받는쪽과 동일한 channel 이름의 채널을 생성하고 postMessage를 가져온다. 그리고 닫기버튼을 눌러서 onCloseEvent가 실행이되면 PostMessage에 특정 message를 넘겨주고 웹뷰를 닫는다

## 신호를 받는쪽 컴포넌트

```ts
...
import { useBroadcastChannel } from 'constants/useBroadcastChannel';

...

useBroadcastChannel('refetchCouponListChannel', (message) => {
if (message === 'refetchCouponList') {
    getCoupons(1, true);
}
});
...

```

받는쪽에서도 동일한 channel 이름의 채널을 생성하고, 특정 메세지가 왔을때 특정 함수를 실행시키게끔 해줄 수 있다.

## 결론

broadcast를 사용하면서 일종의 구독개념이고 소켓 통신과도 매우 유사한 면모를 보이는 것 같다는 생각이 들었다. 그리고 웹뷰 개발을 할 때 필수적이라고 생각을 하였다. 새로연 웹뷰에서 특정행동을 하고 돌아왔을때 데이터를 다시 fetch 해야하는경우가 빈번한데 매번 사용자의 이벤트에 의존하거나 웹뷰를 닫았다 여는행동은 ux를 크게 하락 시키기 떄문이다.
