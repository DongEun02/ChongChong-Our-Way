# development / production 빌드 및 배포 환경 분리

## 이유 확인 - 시작 전

> 이 요구사항이 왜 주어졌는지, 어떤 경험을 만들기 위한 것인지 파악합니다.
> **적용 전 상태(문제, 불편, 위험, 비효율, 학습 공백)를 먼저 남깁니다.**

개발 환경과 운영 환경은 같은 코드를 사용하지만 빌드의 목적은 다르다.
개발 환경에서는 디버깅과 빠른 검증이 중요하고, 운영 환경에서는 최적화된 번들과 안전한 소스맵 처리, 효율적인 캐시가 중요하다.

하나의 Webpack 설정과 하나의 빌드 명령만 사용하면 다음 문제가 생길 수 있다.

- 개발용 API 주소로 빌드한 결과물이 운영 S3에 올라갈 수 있다.
- 개발 환경의 모킹(MSW) 설정이 배포 빌드에 포함되어 실제 API를 호출하지 않을 수 있다.
- 개발과 운영 오류가 같은 환경으로 수집되면 재현 범위와 우선순위를 판단하기 어렵다.
- 파일명이 고정된 자산을 장기 캐시하면 배포 후에도 이전 버전이 노출될 수 있다.
- 운영 소스맵을 정적 파일과 함께 공개하면 원본 코드와 구조가 노출될 수 있다.

단순히 `dev`, `prod` 브랜치를 나누는 것만으로는 이 문제를 막을 수 없다.
어떤 환경을 위한 빌드인지가 빌드 명령, 환경 변수, 산출물, 배포 대상에 일관되게 반영되어야 한다.
따라서 잘못된 환경 조합은 빌드 전에 실패시키고, 정상적인 빌드만 배포하는 것을 목표로 잡았다.

개발 환경과 운영 환경을 각각 두지 않았을때의 불편함또한 존재한다.
실제로 우리 총총 서비스의 첫 MVP가 나오고 몇 번의 버전업 전까지 개발 서버가 존재하지 않았다.
오로지 운영 서버만 존재했고 각자의 변경사항을 로컬 개발서버에서만 확인한뒤 개발 브랜치로 보내고 개발 브랜치에서 바로 운영 브랜치로 향했다.
그렇기 때문에 실제로 우리가 변경한 부분들이 운영 서버에서 잘 작동하는지 미리 확인할 수 없었다.

이 문제로 백엔드 API 응답 형태와 일치하지 않은 문제가 자주 발생했을때 이 문제를 개발 서버에서 미리 캐치하지 못하는 상황이 발생했다.
그렇게 그 날 릴리스를 다섯번이나 다시했던 기억이 있다.

이외에도 결국 로컬 개발서버에서 잘 동작하던 기능들이 운영 서버에서 실제 API가 연동되고 했을때는 또 다르게 예상하지 못한 문제점들이 존재했다.
이 문제점들을 조금 더 빨리 사용자에게 닿기전에 확인할 수 있어야했다.
그렇게 우리는 서서히 개발서버의 필요성을 느꼈다.

## 적용 - 실제 서비스에

> 기술을 학습하고, 실제 팀 프로젝트 서비스에 적용합니다.

### 환경을 local, dev, prod로 구분

로컬 개발과 배포된 개발 환경은 모두 Webpack의 `development` 모드를 사용할 수 있지만 운영 방식은 다르다.
로컬에서는 Webpack Dev Server와 HMR(Hot Module Replacement)이 필요하지만, 배포된 개발 환경은 S3와 CloudFront에서 제공하는 정적 산출물이다.
이 차이를 설정에 드러내기 위해 세 환경을 분리했다.

| 구분           | local              | dev 배포                          | prod 배포                                   |
| -------------- | ------------------ | --------------------------------- | ------------------------------------------- |
| 실행 명령      | `pnpm start:local` | `pnpm build:development`          | `pnpm build:production`                     |
| Webpack 설정   | `webpack.local.js` | `webpack.dev.js`                  | `webpack.prod.js`                           |
| Webpack `mode` | `development`      | `development`                     | `production`                                |
| 소스맵         | 생성하지 않음      | `source-map`                      | `hidden-source-map` 생성 후 Sentry에 업로드 |
| JS 파일명      | `bundle.js`        | `bundle.js`                       | `[name].[contenthash:16].js`                |
| 자산 파일명    | 고정 파일명        | 고정 파일명                       | 콘텐츠 해시 포함                            |
| 환경 변수 공급 | `.env.local`       | CodeBuild                         | CodeBuild                                   |
| 배포 위치      | 로컬 메모리        | S3 `uploads/dev` + dev CloudFront | S3 `uploads/prod` + prod CloudFront         |
| 정적 자산 캐시 | 해당 없음          | `no-cache`                        | `public,max-age=31536000,immutable`         |

운영 환경에서는 콘텐츠 해시가 포함된 정적 자산에 1년 캐시 정책을 적용했다.
index.html은 캐시하지 않도록 하고 배포 후 최신 HTML이 변경된 해시 파일명을 참조하도록 했다.
이를 통해 변경되지 않은 자산은 캐시를 재사용하고, 변경된 자산은 새 URL로 불러오도록 구성했다.

### 공통 설정과 환경별 설정 분리

기존 단일 `webpack.config.js`를 총 4개의 파일로 나눴다.

```text
webpack.common.js
├── webpack.local.js
├── webpack.dev.js
└── webpack.prod.js
```

`webpack.common.js`는 entry, TypeScript자산 loader, HTML 생성, output 경로, 환경 변수 주입처럼 환경과 관계없이 공통인 설정을 관리한다.
각 환경 설정은 `webpack-merge`로 공통 설정을 합성한 뒤 `mode`, `devtool`, 파일명, Dev Server, Sentry 플러그인처럼 환경별로 달라야 하는 값만 추가한다.

하나의 설정 함수 안에 모든 분기를 두는 방법도 가능하다.
하지만 이 방식은 각 환경에 다른 값이 적용되게 될 경우 분기가 늘어나고 이 분기문은 결국 흐름을 읽기 어렵게 만든다.
그렇기에 각 환경을 분리하고 배포 명령에서 어떤 설정을 사용하는지 직접 드러나도록 했다.
또 특정 여러 환경에만 적용되어야 하는 로직같은 경우에는 함수로 추출해 중복과 환경 간 설정 누락을 줄였다.

```ts
export function createSentryPlugin(authToken) {
  return sentryWebpackPlugin({
    org: "woowacourse-31",
    project: "chongchong",
    authToken,
    errorHandler(error) {
      throw error;
    },
    sourcemaps: {
      assets: "./dist/**/*.{js,map}",
    },
  });
}
```

-> sentry플러그인은 로컬, 개발, 운영 중에 개발, 운영 중에만 사용한다.

### 환경 변수를 빌드 시점에 확정

Web 앱에서 사용하는 `API_BASE_URL`, Kakao REST API KEY, PostHog Config, `USE_MSW`, Sentry DSN과 환경 이름을 `Webpack DefinePlugin`으로 번들에 주입했다.
따라서 이 설정들은 브라우저 실행 시점이 아니라 빌드 시점에 확정된다. 즉, 변경하려면 다시 빌드해야 한다.

로컬에서만 `webpack.local.js`가 `.env.local`을 읽는다.
dev와 prod 배포 빌드는 저장소의 `.env` 파일을 읽지 않고 CodeBuild 환경 변수만 사용한다.
이로써 배포 설정과 민감한 빌드용 값을 소스 코드와 분리했다.
Sentry Auth Token은 플러그인의 소스맵 업로드에만 사용하고 번들에는 주입하지 않는다.

### 잘못된 환경 조합을 배포 전에 차단

`frontend/codebuild/buildspec.yml`의 `pre_build`에서 다음 조건을 검증한다.

- `DEPLOY_ENV`는 `dev` 또는 `prod`여야 한다.
- dev의 `API_BASE_URL`은 `https://dev.chongchong.kr`, prod는 `https://chongchong.kr`와 일치해야 한다.
- `API_BASE_URL`, `KAKAO_REST_API_KEY`, `SENTRY_AUTH_TOKEN`은 빈 값이어서는 안 된다.
- API 주소는 경로와 trailing slash가 없는 HTTPS origin이어야 한다.
- 배포 빌드에서 `USE_MSW`는 반드시 `false`여야 한다.
- PostHog Host와 Project Token은 둘 다 설정하거나 둘 다 설정하지 않아야 한다.

```yml
const required = ['API_BASE_URL', 'KAKAO_REST_API_KEY', 'SENTRY_AUTH_TOKEN'];
for (const name of required) {
  if (!process.env[name]?.trim()) {
    throw new Error(`Missing CodeBuild environment variable: ${name}`);
  }
}
```

(위와 같은 형식으로 빌드 스크립트에서 특정 환경변수가 제공되지 않으면 에러를 반환한다.)

추가로 `webpack.dev.js`와 `webpack.prod.js`도 각각 자신의 설정과 `DEPLOY_ENV`가 일치하는지 다시 검증한다.
CodeBuild의 분기가 잘못되거나 사람이 잘못된 스크립트를 직접 실행해도 다른 환경의 결과물이 생성되지 않도록 중복 방어했다.

```ts
// webpack.dev.js
if (process.env.DEPLOY_ENV !== "dev") {
  throw new Error("DEPLOY_ENV must be dev for a development deployment build.");
}
```

(이런식으로 선언한 배포 환경과 Webpack 설정이 일치하지 않으면 빌드를 중단한다.)

### 소스맵과 오류 환경 분리

dev와 prod 모두 빌드 시 Sentry Webpack Plugin이 소스맵을 업로드한다.
`SENTRY_ENVIRONMENT`에는 `DEPLOY_ENV`를 주입해 같은 Sentry 프로젝트 안에서도 dev와 prod 오류를 구분할 수 있게 했다.

prod는 `hidden-source-map`으로 맵을 생성하여 번들에 소스맵 참조를 남기지 않는다.
Webpack 빌드 중 Sentry 업로드가 완료된 뒤 `dist` 안의 `.map` 파일을 삭제하므로 S3에는 소스맵이 배포되지 않는다.(아래 프로덕션 스크립트 명령어를 보면 알 수 있다.)
dev는 브라우저에서 원본 코드를 기준으로 디버깅할 수 있도록 `source-map`을 유지한다.

```json
{
  "name": "frontend",
  "version": "2.0.5",
  "description": "chongchong frontend",
  "main": "index.js",
  "scripts": {
    "build:production": "webpack --config webpack.prod.js && find dist -type f -name '*.map' -delete && cp -R src/pwa dist/pwa"
  }
}
```

## 관찰 - 적용 후

> 적용 후 무엇이 달라졌는지 확인합니다.
> 데이터, 로그, 화면, 대시보드, 팀 피드백, 사용자 반응 등 확인한 것을 남깁니다.

적용 후 다음 변화를 확인할 수 있다.

- `package.json`의 명령만 보아도 로컬, dev 배포, prod 배포에 사용하는 설정을 구분할 수 있다.
- 빌드 환경과 API 주소가 일치하지 않거나 MSW가 켜져 있으면 S3 업로드 전에 실패한다.(더 안정적으로 빌드하고 배포한다)
- dev와 prod가 서로 다른 S3 prefix와 CloudFront Distribution을 사용한다.
- prod의 해시 자산은 장기 캐시하면서, 교체 가능한 HTML과 PWA 파일은 매번 재검증하도록 캐시 정책이 분리되었다.
- prod 소스맵은 Sentry에만 업로드되고 정적 서비스에는 포함되지 않는다.

다만 저장소의 설정으로 확인할 수 있는 범위와 실제 배포 검증은 구분해야 한다.
CodeBuild 설정에 주입된 환경 변수, S3의 실제 오브젝트 메타데이터, CloudFront의 무효화 완료, Sentry에 업로드된 dev/prod 이벤트와 소스맵은 각 배포 후에 별도로 확인해야 한다.

## 기록과 설명

> 수행 내용과 판단 근거를 문서로 남깁니다.

### 환경 분리 전체 흐름

```text
CodeBuild 환경 변수
        │
        ├─ DEPLOY_ENV=dev  ─→ 값 검증 ─→ webpack.dev.js  ─→ dev S3(techcourse-project-2026chongchong/uploads/dev/)  ─→ dev CloudFront
        │
        └─ DEPLOY_ENV=prod ─→ 값 검증 ─→ webpack.prod.js ─→ prod S3(techcourse-project-2026chongchong/uploads/prod/) ─→ prod CloudFront
                                      │
                                      └─→ Sentry 소스맵 업로드
```

### 배포 후 확인 목록

다음과 같은 과정을 실제로 확인해 보면서 환경 분리가 의도적으로 잘 적용되었는지 확인했습니다.

1. dev와 prod 각 URL이 기대한 API origin으로 요청하는지 확인한다.
2. 배포된 앱에서 MSW가 시작하지 않는지 확인한다.(서비스워커 확인)
3. prod의 JS/font 자산 파일명에 content hash가 포함되었는지 확인한다.(배포 결과물 or 네트워크에서 확인)
4. `index.html`과 PWA 파일은 `no-cache`, prod 해시 자산은 `immutable`을 응답하는지 확인한다.(네트워크 캐시 컨트롤 확인)
5. prod S3 경로에 `.map` 파일(소스맵)이 없고 Sentry에서는 원본 코드 기준의 stack trace를 확인할 수 있는지 검증한다.(버킷에 소스맵이 없는지 확인 + Sentry에서는 원본 ts/tsx가 파일명/행 번호/코드가 표시되는지 확인)
6. 같은 오류를 dev와 prod에서 각각 발생시켜 Sentry의 `environment`가 분리되는지 확인한다.
7. CloudFront 무효화가 `Completed` 상태가 된 후 새 버전이 제공되는지 확인한다.(CF 무효화 확인 + 새로운 버전 기능들이 제공됮는지 확인)
