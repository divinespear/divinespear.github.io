---
layout: post
title: "다소 변태적인 Typescript의 Type Constraints"
date: 2021-04-13T23:32:00+09:00
categories: [typescript]
tags: [typescript, generic]
---

```typescript
interface ResponseType<R> {
  responseType: R;
}

interface VariableType<V> {
  variableType: V;
}

interface MessageType<R, V = undefined> extends ResponseType<R>, VariableType<V> {
  // 클래스 타입이 아니면 deep type constraints가 작동하지 않는다.
  new: () => MessageType<R, V>;
}

type ResponseOf<R extends ResponseType<unknown>> = R['responseType'];
type VariableOf<V extends VariableType<unknown>> = V['variableType'];

interface Message {
  currentUser: MessageType<UserObject | undefined>;
  userList: MessageType<UserObjectPage, UserSearchArgs>;
  // 이하 생략
}

// message는 Message 인터페이스의 필드명으로 제한된다.
// variable과 리턴 타입 역시 인터페이스에 설정한 대로...
function call<K extends keyof Message, R = ResponseOf<Message[K]>>(message: K, variable: VariableOf<Message[K]>): R {
  // 대충 생략
}

// variable은 optional로 하지 않았기 때문에 이 경우에는 무조건 undefined를 넘겨줘야 한다.
// 리턴 값은 UserObject 또는 undefined
const user = call('currentUser', undefined);

// variable은 UserSearchArgs
// 리턴 값은 UserObjectPage
const list = call('userList', { ... });
```

lib.dom.d.ts에 이벤트 타입 선언을 저런 식으로 해놨길래 한번 삽질해봤는데 타입 체크 엄청 잘된다. \
이게 무슨 C++ 템플릿 메타프로그래밍도 아니고...

그나저나 내 취침 시간은 어디에... ㅠㅠ
