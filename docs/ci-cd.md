# 배포 자동화 환경 구축

## 이유 확인 - 시작 전

> 이 요구사항이 우리 서비스의 어떤 문제와 연결되는지 정의합니다.
> **적용 전 상태(문제, 불편, 위험, 비효율, 학습 공백)를 먼저 남깁니다.**

이전에 CI를 두지 않았을때 매번 로컬에서 직접 CI에서 수행해야할 것들을 하나하나 돌려가며 확인했다.

이 과정에서 작은 수정의 경우 까먹고 수행하지 않을때가 있었다.

그랬을때 CI 과정에서 확인하지 않아서 그 문제가 이후에도 발견되기 전까지 잡히지 않은 상태로 유지되었다.

우리가 매번 작업할때마다 지켜져야 하는 최소한들이 있다.

린트, 테스트, 타입체크와 같은 부분들이다.

이 부분들을 CI에 수행하게 된다면 린트와 같은 일관적인 규칙들을 지켜나가고 테스트와 타입체크와 같은 부분들을 매번 자동적으로 체크하여 안정적인 프로그램을 만들어 나갈 수 있다.

솔직히 말하면 현재 아직 CD를 적용한 상태는 아니다.

https://github.com/Antoliny0919/taste-fe-deploy

하지만 CD를 POC로 만들어 체험해 보면서 **“만약 우리가 CD 를 구축하지 않았을때 어떻게 될까?”**에 대한 고충을 겪어볼 수 있었다.

AI를 통해 우리가 하게될 경우의 수인 Nginx 로 인스턴스에 배포하게 되는 상황을 체험해 보았다.(위 리포지토리)

실제로 개인 AWS 계정으로 인스턴스를 생성하여 테스트 했다.

이 체험을 통해 우리가 만약 CD를 구축하지 않았다면 매번 직접 로컬에서 빌드하고 빌드한 결과물을 직접 인스턴스 서버 특정 경로에 전달해야한다.

너무나도 비효율적이며 우리가 특정 브렌치에 새로운 버전을 푸시하게 되면 자연스럽게 빌드하고 새로운 버전으로 배포되어야 이 수고를 덜어낼 수 있다.

## 적용 - 실제 서비스에

> 기술을 학습하고, 실제 팀 프로젝트 서비스에 적용합니다.

CI 과정에서 테스트를 자동으로 실행하여 코드 변경 시 기존 기능에 문제가 발생했는지 PR 단계에서 확인할 수 있도록 구성한다.

우리의 경우 .github CI를 적용했다.

.github/workflows 하위에 CI와 관련된 workflow 파일을 추가하여 적용했다.

```yaml
name: CI

on:
  push:
    branches: [main, dev]
    paths:
      - 'frontend/**'
  pull_request:
    branches: [main, dev]
    paths:
      - 'frontend/**'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions: read-all

jobs:
  lint:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - name: Checkout
        uses: actions/checkout@v7
      - name: Set up pnpm
        uses: pnpm/action-setup@v4
        with:
          package_json_file: frontend/package.json
      - name: Set up Node
        uses: actions/setup-node@v7
        with:
          node-version-file: frontend/.nvmrc
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml
      - name: Install Dependency
        run: pnpm install
      - name: ESLint
        run: pnpm run lint
      - name: Prettier
        run: pnpm run format:check

  typecheck:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - name: Checkout
        uses: actions/checkout@v7
      - name: Set up pnpm
        uses: pnpm/action-setup@v4
        with:
          package_json_file: frontend/package.json
      - name: Set up Node
        uses: actions/setup-node@v7
        with:
          node-version-file: frontend/.nvmrc
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml
      - name: Install Dependency
        run: pnpm install
      - name: Typecheck
        run: pnpm run type-check

  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - name: Checkout
        uses: actions/checkout@v7
      - name: Set up pnpm
        uses: pnpm/action-setup@v4
        with:
          package_json_file: frontend/package.json
      - name: Set up Node
        uses: actions/setup-node@v7
        with:
          node-version-file: frontend/.nvmrc
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml
      - name: Install Dependency
        run: pnpm install
      - name: Test
        run: pnpm run test
```

CD는 아직 적용하지 않은 상태다.

솔직히 말하면 지금 당장 프로젝트에 적용해봤자 의미가 없는 과정이라고 생각했다.

별 기능도 하나 없는 상태라 이후에 구축해도 충분하고 다른 포인트들에 더 신경을 써야하는 상황이다.

하지만 현재 어떻게 구축해야할지에 대해서 파악해 두었고 스크립트도 또한 미리 만들어 체험해보았다.

POC 리포지토리와 본인계정 인스턴스를 만들어 실제로 main 브렌치에 푸시되었을때 변경사항이 서버에 반영되는걸 확인할 수 있었다.

```yaml
name: Deploy to EC2

on:
  push:
    branches: [main]
    paths:
      - 'frontend/**'
      - '.github/workflows/deploy-ec2.yml'
  workflow_dispatch:

concurrency:
  group: deploy-ec2
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 11.9.0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
          cache-dependency-path: frontend/pnpm-lock.yaml

      - name: Install
        working-directory: frontend
        run: pnpm install --frozen-lockfile

      - name: Build
        working-directory: frontend
        run: pnpm build
        env:
          DEPLOY_TARGET: ec2
          API_BASE: /api

      - name: Pack
        run: tar -czf dist.tar.gz -C frontend/dist .

      - name: Copy to EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: dist.tar.gz
          target: /tmp

      - name: Activate release
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            set -e
            BASE=/var/www/chongchong
            RELEASE=$BASE/releases/$(date +%Y%m%d%H%M%S)
            mkdir -p $RELEASE
            tar -xzf /tmp/dist.tar.gz -C $RELEASE
            ln -sfn $RELEASE $BASE/current-tmp
            mv -Tf $BASE/current-tmp $BASE/current   # 원자적 교체
            ls -1dt $BASE/releases/* | tail -n +4 | xargs -r rm -rf  # 최근 3개만 보관
            rm -f /tmp/dist.tar.gz
```

위와 같은 CD 스크립트를 사용했다.

nginx의 root에 정적파일을 서빙할 경로를 지정했다.

우리는`/var/www/chongchong/current` 경로를 지정했다.

그렇기에 해당 경로 하위에 빌드한 파일들을 전달하기만 하면 된다.

여기서 더 나아가 release + symlink 교체 패턴을 사용하여 이전 배포 버전들을 보관하고 무중단 배포를 하도록 했다.

새로운 버전(빌드)이 출시되면 release에 해당 gzip을 전달한뒤에 심링크만 옮겨주기만 하면 된다.

자연스럽게 이전버전이 새로운버전으로 옮겨지는 형태이다.

만약 새로운버전이 이전 파일들을 덮어씌우게 된다면 그 잠깐의 과정에 접속한 사용자들은 잘못된 화면을 보게 된다.

하지만 지금처럼 심링크 방식을통해 새로운 버전을 가리키는 방식으로 하면 중단없이 깔끔하게 새로운 버전으로 보이게 할 수 있다.

또 이전 릴리스를 보관하기 때문에 문제가 생길경우 심링크를 옮겨 이전 릴리스로 되돌릴 수 있다.

## 관찰 - 적용 후

> 적용 후 무엇이 달라졌는지 확인합니다.
> 데이터, 로그, 화면, 대시보드, 팀 피드백, 사용자 반응 등 확인한 것을 남깁니다.

매번 PR을 제출할때나 병합되어 특정 브랜치에 커밋이 추가되었을때 CI가 트리거되도록 했다.

Test, TypeCheck, Lint 체크를 의무적으로 하게하여 안전성을 높였다.

정확히 말하면 ‘의무적’이라기 보다는 우리에게 이러한 정보가 리마인드 된다는 점이다. (근데 사실상 의무적인거 같기는 하다 → 안지킬 이유가 거의 드물기 때문에.)

!Screenshot 2026-08-13 at 4.06.48 PM.png

만약 CD를 구축하게 된다면 우리가 매번 빌드하고 인스턴스의 특정 경로에 전달하는 과정을 하지 않아도 자연스럽게 특정 브렌치에 푸시하게 되면 배포를 할 수 있게된다.

놀랍게도 이 과정은 우리가 아닌 새로운 누군가가 오더라도 익숙하지 않은 어려운 방식을 안내하지 않게 만든다.

“저 이거 배포 어떻게 해야하나요 ? “

라는 질문에 우리는 “일단 먼저 빌드하구요 그 다음에 spc라는 커맨드로 이 인스턴스 특정 경로에 전달하세요 그 다음에 심링크를 …”

이런 답변이 아닌 “그냥 main 브렌치에 푸시하세요”로 끝나게 된다.

## 기록과 설명

> 수행 내용과 판단 근거를 문서로 남깁니다.

### CI

CI 문서인 workflow 파일을 어떻게 작성해야할지 조금 어려웠다.

핵심적인 전략은 다른 사람들의 CI파일을 확인하고 거기서 사용하는 문법들에서 중요한 부분이나 흐름들을 파악하는 것이었다.

CI로 테스트, 린트, 타입체크를 추가한 이유는 매번 체크되어야할 기초적인 포인트라고 느꼈다.

기본적으로 지켜야할 규칙이나 최소한의 안정성을 채우기 위해 이 부분들을 작업자인 우리에게 리마인드 해야할 필요가 있다고 생각했다.

### CD

지금 당장 CD를 어떻게 구축해야할지 어려웠다.

필요성을 지금 시점에는 별로 느끼지 못했기 때문이다.

이 상황에서 AI를 통해 우리가 배포할 방향인 ec2 + nginx 와 S3 + CloudFront 전략을 데모버전으로 체험해 볼 수 있었다.

한 가지 아쉬운점은 S3 + CloudFront 전략은 해보지 못했다.

CloudFront를 안다뤄봐서 약간의 러닝커브가 존재했다. (다른곳에 시간을 써야하기도 했다)

더군다나 S3를 하게 된다면 정적 파일 서빙을 S3를 거치기 때문에 ec2로 가는 트래픽을 줄일 수 있다는 이점이 존재한다.

놀라운건 이 이점대비 비용적인 부분은 거의 없다.

심지어 CD로 구축해야할 부분이 어렵지 않다.

이전에는 ec2에 정적파일을 보냈다면 S3를 사용할때는 S3에 정적파일을 보내기만 하면 된다.

그래서 나중에 S3 + CloudFront까지 체험해보고 CD를 구축할 생각이다.

별개로 이 과정이 우리에게 얼마나 편함을 안겨줄지 PoC를 통해 느껴봐서 너무나도 좋았다.
