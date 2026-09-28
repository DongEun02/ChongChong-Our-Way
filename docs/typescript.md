# TypeScript 사용 기준 수립 및 런타임 타입 안전성 확보

## 이유 확인 - 시작 전

> 이 요구사항이 왜 주어졌는지, 어떤 경험을 만들기 위한 것인지 파악합니다.
> **적용 전 상태(문제, 불편, 위험, 비효율, 학습 공백)를 먼저 남깁니다.**

TypeScript를 사용하고 있어도 타입이 언제나 실제 값을 보장하는 것은 아니다.

프로젝트 안에서도 Props를 어떤 파일은 `interface`로, 어떤 파일은 `type`으로 선언했다. 이벤트 핸들러도 DOM 이벤트를 그대로 넘기는 경우와 도메인 값만 넘기는 경우가 섞여 있었다. 각자 나름의 이유는 있었지만 팀이 공유하는 기준이 없어, 코드를 읽거나 리뷰할 때 “왜 여기만 다른가?”를 반복해 판단해야 했다.

이전에는 API 응답 타입을 정의해 두면 서버가 항상 그 형태로 값을 보낼 것이라고 생각했다. 하지만 `interface`와 `type`은 JavaScript로 변환될 때 사라진다. 서버가 잘못된 응답을 보내도 런타임에서는 그 타입이 내용을 검증해 주지 않는다.

또한 `any`를 사용하면 타입 체커를 통과하기는 쉽지만, 그 시점부터 타입 정보가 전파되지 않는다. 잘못된 값이 들어와도 코드를 실행하기 전에 알기 어렵고, 결국 사용자가 해당 화면에 진입한 뒤에야 런타임 오류로 드러난다.

프로젝트의 상태를 단순한 `string`이나 여러 선택적 속성으로 표현하면 실제로는 존재할 수 없는 조합도 타입으로 허용된다. 예를 들어 공지를 읽지 않은 상태인데 `readAt`이 존재하거나, 과제를 제출하지 않았는데 제출 내용이 존재하는 상태를 만들 수 있다.

마지막으로 API, 폼, 컴포넌트에서 비슷한 타입을 각각 직접 정의하면 하나의 정책이 바뀌었을 때 여러 타입이 서로 다른 모양으로 남을 수 있다. TypeScript를 단순히 JavaScript에 타입을 붙이는 용도로만 사용하지 않고, **잘못된 값이 시스템 안쪽으로 들어오지 못하게 하고 유효한 상태만 표현하는 기준**이 필요했다.

## 적용 - 실제 서비스에

> 기술을 학습하고, 실제 팀 프로젝트 서비스에 적용합니다.

### `interface`, `type` 선택 기준

`interface`와 `type`은 대부분의 객체 타입에서 서로 대체할 수 있다. 따라서 취향으로 하나를 고르기보다 언어 기능이 필요한지로 구분했다.

| 상황                                     | 기준                           |
| ---------------------------------------- | ------------------------------ |
| 확장 가능한 객체 계약과 컴포넌트 Props   | `interface`                    |
| 유니온, 조건부 타입, 함수 타입           | `type`                         |
| 기존 타입을 그대로 별칭                  | `type`                         |
| 기존 Props를 확장하고 일부 속성을 재정의 | `interface` + `extends`/`Omit` |

즉, 단순 Props라는 이유만으로 무조건 `interface`를 강제하지 않는다. 단순히 기존 React Props의 별칭이라면 `type`이 자연스럽고, 우리가 속성을 추가하는 객체 계약이라면 `interface`를 사용한다.

### 외부 값은 `unknown`으로 받고 런타임에서 검증

타입 정보는 런타임에 사라지므로 외부 API의 실제 값은 별도로 검증해야 한다. 알 수 없는 값은 `any`가 아닌 `unknown`으로 받고, 사용하기 전에 타입을 좁힌다.

이를 적용해 API의 `response.json()` 결과를 바로 응답 타입으로 단언하지 않고 `unknown`으로 받았다.

```ts
const data: unknown = await response.json();

if (!isNoticeListResponse(data)) {
  throw new Error('공지 목록 응답 형식이 올바르지 않습니다.');
}

return data;
```

`isNoticeListResponse`는 Zod 스키마의 `safeParse`로 실제 값을 검증하는 사용자 정의 타입 가드다.

```ts
export function isNoticeListResponse(
  data: unknown,
): data is NoticeListResponse {
  return noticeListSchema.safeParse(data).success;
}
```

단순히 `data as NoticeListResponse`로 잘못된 값을 숨기지 않았다. 검증을 통과한 뒤에만 `NoticeListResponse`로 좁혀지도록 했고, 단언은 호출부에 반복하지 않고 검증 함수 안에 감추었다.

서버 오류도 동일하게 `unknown`으로 받은 뒤 `instanceof`, `typeof`, `Array.isArray`, 사용자 정의 타입 가드를 통해 `ValidationError`와 `ApiError`로 변환했다. 이때 오류의 `code`, `status`, `fieldErrors`는 `readonly`로 정의해 생성된 뒤 바뀌지 않는 값임을 타입에 표현했다.

런타임 검증의 범위는 다음과 같다.

- 응답 내용을 화면에 그리거나, 상태 분기에 사용하거나, 토큰처럼 보관하는 API는 검증한다.
- 삭제·수정 요청처럼 응답 body를 사용하지 않는 API는 HTTP 성공 여부만 확인한다.

### 유효한 상태만 만들 수 있도록 타입을 설계

여러 속성을 선택적으로 두기보다 각 상태를 별도의 타입으로 정의하고 유니온으로 합쳤다. 또한 `string`보다 리터럴 유니온처럼 더 좁은 타입을 사용해 오타와 잘못된 상태를 막았다.

공지 읽음 상태는 다음과 같이 태그된 유니온으로 표현했다.

```ts
export type MemberReadStatus =
  | { readStatus: 'READ'; readAt: string }
  | { readStatus: 'UNREAD' }
  | { readStatus: 'NOT_ASSIGNED' };
```

`readStatus` 값이 `READ`일 때만 `readAt`이 존재한다. 따라서 `UNREAD`인데 `readAt`이 있는 잘못된 조합은 타입 단계에서부터 만들 수 없다. Zod에서도 `z.discriminatedUnion('readStatus', ...)`으로 같은 규칙을 표현해 컴파일 타임과 런타임의 판단이 일치하도록 했다.

과제 제출 상태도 `SUBMITTED`, `NOT_SUBMITTED`, `NOT_ASSIGNED`를 각각 다른 인터페이스로 만들었다. 제출한 경우에만 생성 시각과 제출 내용을 제공하도록 해 상태와 데이터 사이의 관계를 타입으로 나타나도록 했다.

### 넓은 `string` 대신 리터럴 유니온 사용

역할과 상태처럼 허용되는 값이 정해져 있다면 넓은 `string`을 사용하지 않고 리터럴 유니온으로 제한했다.

```ts
export type Role = 'LEADER' | 'MEMBER';
export type SubmissionStatus = 'NOT_ASSIGNED' | 'NOT_SUBMITTED' | 'SUBMITTED';
```

이로써 오타나 허용하지 않은 문자열이 들어오는 것을 타입 검사 단계에서 막았다.

### 기존 타입으로부터 새 타입 만들기

동일한 필드 구조를 여러 타입에 반복하지 않고 기존 타입에서 파생했다.

- `AssignmentValue`는 `AssignmentDetail`을 다시 적지 않고 `Omit<AssignmentDetail, 'id'>`로 만들었다.
- 수정 API의 입력인 `UpdateAssignmentValue`는 `Partial<AssignmentValue>`로 정의했다.
- `InputFieldProps`는 React 컴포넌트의 props를 따로 복사하지 않고 `ComponentPropsWithoutRef<typeof Input>`에서 파생했다.
- `FeaturePreview`는 훅의 반환 타입을 직접 재작성하지 않고 `ReturnType<typeof useFeaturePreview>`으로 추출했다.
- 목(mock) 데이터의 타입은 `z.infer<typeof schema>`로 스키마에서 파생했다.

원본 컴포넌트나 도메인 타입이 바뀌면 파생된 타입도 같이 바뀌므로, 둘 사이가 다른 모양으로 남는 것을 막았다.

### `readonly`, `as const`, `satisfies`

함수가 매개변수를 수정하지 않는다면 `readonly`로 그 사실을 타입에 드러냈다. `useIntegerParams`의 `params: readonly K[]`를 통해 함수 안에서 인자를 수정하지 않는다는 약속을 남겼다.

리터럴 타입의 문맥을 유지하기 위해 `as const`도 사용했다. `useBooleanState`의 반환값을 `as const`로 두어 일반 배열이 아닌 readonly 튜플로 추론되게 했다.

객체나 Zod 스키마가 예상한 타입을 만족하는지 검사할 때는 단언보다 `satisfies`를 사용했다.

```ts
const studyValidator = {
  name: (value) => {
    /* ... */
  },
  description: (value) => {
    /* ... */
  },
} satisfies Record<keyof StudyInput, FieldValidator>;
```

이 표현은 `StudyInput`의 필드가 추가되면 검증기가 누락되었다는 오류를 바로 발생시킨다. 한편 각 함수의 매개변수와 반환값은 객체 리터럴이 원래 가진 구체적인 타입으로 유지된다. 타입을 억지로 바꾸는 단언과 달리, 조건을 만족하는지 검사하면서 추론 정보도 잃지 않는다.

### 기준마다 자동화할 수 있는 범위 구분

모든 기준을 린트 규칙으로 만드는 것은 오히려 의도를 잃게 할 수 있다. 자동으로 판단할 수 있는 오류와 문맥이 필요한 설계 판단을 구분했다.

| 기준                                             | 강제 수단                                   | 확인 시점                           |
| ------------------------------------------------ | ------------------------------------------- | ----------------------------------- |
| 암묵적 `any` 금지, null 안전성, 함수 인자 호환성 | TypeScript `strict`                         | 코드 작성 중, `pnpm type-check`, CI |
| 명시적 `any` 금지                                | ESLint `@typescript-eslint/no-explicit-any` | 에디터, `pnpm lint`, CI             |
| Hooks·React Query 사용 규칙                      | ESLint React Hooks, TanStack Query 규칙     | 에디터, `pnpm lint`, CI             |
| 정적 타입과 Zod 스키마의 호환성                  | `satisfies z.ZodType<T>`                    | 타입 검사 중                        |
| API 응답의 실제 형태                             | Zod `safeParse`                             | 브라우저에서 API 응답을 받을 때     |
| `interface`/`type` 선택                          | 문서와 코드 리뷰                            | 설계·리뷰 시점                      |

`interface`와 `type`의 선택은 스타일에 가깝고, 어느 한쪽을 쓰는지만으로 안정성이 바뀌지는 않는다. 반면 `any`, 런타임 검증 누락, 잘못된 유니온 설계는 실제 오류로 이어질 수 있다. 따라서 스타일 기준은 리뷰에서 의도를 확인하고, 안전성 기준은 가능한 한 컴파일러·린트·런타임 스키마로 강제한다.

## 관찰 - 적용 후

> 적용 후 무엇이 달라졌는지 확인합니다.
> 데이터, 로그, 화면, 대시보드, 팀 피드백, 사용자 반응 등 확인한 것을 남깁니다.

`frontend/src`의 TypeScript 코드를 기준으로 확인했을 때 명시적인 `any`는 0개이고 `unknown`은 61개다. 알 수 없는 값을 편의를 위해 `any`로 열어 두지 않고, 확인하기 전에는 사용할 수 없는 값으로 다루고 있다.

`interface`와 `type`은 서로 다른 사람의 선호로 구분하지 않고, 확장·유니온·조건부 타입 등 필요한 언어 기능으로 선택할 수 있게 되었다.

스터디, 멤버, 공지, 과제 도메인에는 총 4개의 `responseSchemas.ts`가 있고, 코드에서 `safeParse`를 사용한 런타임 검증은 21곳이다. 서버 응답이 바뀌거나 필드가 누락되면, 잘못된 값이 컴포넌트까지 전파되기 전에 API 경계에서 즉시 오류로 처리된다. 어떤 응답이 잘못되었는지도 도메인별 메시지로 확인할 수 있다.

리터럴 유니온과 태그된 유니온을 사용하면서 오타나 존재할 수 없는 상태 조합이 코드를 실행하기 전에 오류로 드러난다. 예를 들어 `MemberReadStatus`에 새로운 상태를 추가할 때는 정적 타입과 Zod의 `discriminatedUnion`을 함께 수정해 컴파일 타임과 런타임의 규칙을 일치시켜야 한다.

타입을 기존 타입과 실제 값에서 파생하면서 동일한 필드 목록을 여러 곳에 반복해 적지 않게 되었다. 원본이 바뀌었는데 수정 입력과 컴포넌트 props는 이전 모양에 머무는 문제를 줄였다.

마지막으로 `pnpm run type-check`를 실행해 `strict` 설정으로 전체 코드의 타입 검사가 통과하는 것을 확인했다. 이 명령은 CI의 독립된 `typecheck` job에서도 실행되므로 변경으로 타입 약속이 깨지면 PR 단계에서 확인할 수 있다.

## 기록과 설명

> 수행 내용과 판단 근거를 문서로 남깁니다.

현재 프로젝트에 적용한 TypeScript 사용 기준은 다음과 같다.

1. 확장할 객체 계약은 `interface`, 유니온·조건부 타입·별칭은 `type`으로 표현한다.
2. 런타임에서 사라지는 타입만으로 외부 값을 신뢰하지 않는다.
3. API 응답과 `catch`의 오류처럼 알 수 없는 값은 `any`가 아닌 `unknown`으로 받고, Zod·내장 검사·타입 가드로 좁힌다.
4. 응답 body를 화면·분기·보관에 사용하면 런타임에 검증하고, body를 사용하지 않으면 성공 여부만 확인한다.
5. 상태 값은 넓은 `string`보다 리터럴 유니온으로 정의하고, 여러 속성이 함께 바뀌어야 한다면 태그된 유니온으로 관계를 표현한다.
6. 제네릭은 입력과 결과 등 두 곳 이상의 타입 관계를 표현할 때 사용한다.
7. 기존 타입으로부터 파생할 수 있는 타입은 `Omit`, `Partial`, `keyof`, `typeof`, `ReturnType`, `z.infer`로 만든다.
8. 함수가 수정하지 않는 매개변수와 생성 후 바뀌지 않는 값은 `readonly`로 의도를 표현한다.
9. 리터럴 정보를 유지해야 할 때만 `as const`를 사용하고, 일반 배열을 받는 함수는 readonly 튜플도 받을 수 있도록 설계한다.
10. 단언으로 오류를 숨기기보다 `satisfies`로 타입 조건을 검사하고 원래의 추론 정보를 유지한다.
11. 스타일 기준은 리뷰로, 안전성 기준은 컴파일러·린트·런타임 검증·CI로 강제한다.
