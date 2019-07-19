---
layout: post
title:  "vue-cli + nightwatch on docker 삽질"
date:   2019-07-19 08:20:00 +0900
categories: tech node
tags: [node.js, node, 노드, vue.js, vue, vue-cli, test, 테스트, e2e, nightwatch, selenium, 셀레늄, docker, 도커]
---
테스트란걸 날로 먹다가 (맨날 유닛 테스트 정도만 돌림) 그냥 한번 심심해서 e2e 테스트까지 돌리기로 했다.
뭐 유닛 테스트나 서버쪽 e2e 테스트는 그냥 잘 돌아간다.

하지만 UI가 출동하면 어떨까?

# vue-cli + nightwatch on docker 삽질

## 삽질의 원인

GitLab에서 CI를 굴리는데, 유닛 테스트나 서버쪽 e2e 테스트는 잘 돌아간다.
문제는 UI e2e 테스트인데, 도커 기반으로 CI가 굴러가는 GitLab 특성상[^1] 테스트를 위한 적당한 도커 이미지를 찾아야 했다.

근데 그런거 없.어...

그래서 그냥 테스트용으로 node.js 컨테이너를 만들고 셀레늄 컨테이너를 별도로 띄우기로 했다.
그리고 그대로 헬게이트 오픈!

## 삽질기
 
진짜 기본적인 설정만 두고 하면 **`http://localhost:8080`** 을 열려고 하기 때문에...

<figure>
  ![셀레늄 파폭 에러](/_assets/images/2019-07-19-01/selenium-result-firefox.png)
  <figcaption>셀레늄 파폭: 모라고요?</figcaption>
</figure>

<figure>
  ![셀레늄 크롬 에러](/_assets/images/2019-07-19-01/selenium-result-chrome.png)
  <figcaption>셀레늄 크롬: 잘 안보이지 말입니다?</figcaption>
</figure>

Aㅏ...
결국 내 랩탑에 디버깅용 셀레늄 컨테이너를 올리고 vnc로 붙어서 일일히 확인하는 삽질을 해서야 해결할 수 있었다.

### 삽질 내역

 1. 처음에는 렌더링이 끝나기 전에 확인하나 싶어서 타임아웃 값을 늘려봤는데, 그게 아니더라...
 2. `VUE_DEV_SERVER_URL` 이라는 환경변수를 보게 되어있는데, 이건 vue-cli의 nightwatch 플러그인에서 때려박아준다. 그래서 이걸 따로 설정해봤는데 맛있게 씹어먹더라.
 3. `vue-cli-service test:e2e` 옵션을 보면 `--url` 옵션이 있는데 이걸 설정하면 devserver를 띄우지 않으므로 테스트를 할 수 없다. ~~EPIC FAIL~~.
 4. `VUE_NIGHTWATCH_USER_OPTIONS` 환경변수에 JSON을 때려박아서 설정 커스터마이징을 할 수 있는데, 괜시리 복잡해져서 포기.

## 해결
`vue.config.js` 에 `devServer.public` 옵션이 있는데, 이걸 설정해주니까 잘 되더라.
근데 나는 docker에서 돌릴거잖아? 아이피가 랜덤이라 난 안될거야 아마.

가 아니지. 답은 만들면 된다.

일단 환경변수 셀레늄 컨테이너 호스트명을 정의할 `E2E_SELENIUM_HOST`와 테스트할 브라우저를 정의할  `E2E_SELENIUM_BROWSER`를 만들었다.

그리고...

**`vue.config.js`**
```javascript
const os = require('os');

function findNetworkAddress() {
  const ifaces = os.networkInterfaces();
  const filtered = Object.keys(ifaces).map((ifname) => ifaces[ifname].filter((i) => i.family === 'IPv4' && !i.internal)).flat();
  if (filtered.length === 0) {
    throw new Error('no ip address found');
  }
  return filtered[0].address;
}

module.exports = {
  devServer: {
    ...(process.env.E2E_SELENIUM_HOST ? {public: `${findNetworkAddress()}:8080`} : undefined),
  },
  // 이하 생략
}
```

**nightwatch 설정** (내 경우에는 원격용으로 `nightwatch.remote.config.js`라는 이름으로 만듬)
```javascript
// headless 모드에서 쓰실 분들은 주석을 해제하시면 되겄습니다.
module.exports = (function(settings) {
  const desiredCapabilities = settings.test_settings.default.desiredCapabilities;
  const name = desiredCapabilities.browserName;
  switch (name) {
    case 'chrome':
      desiredCapabilities.chromeOptions = {
        args: [
          //'--headless',
          '--no-sandbox',
          '--disable-gpu',
          '--disable-dev-shm-usage',
        ],
      };
      break;
    case 'firefox':
      /*
      desiredCapabilities['moz:firefoxOptions'] = {
        args: [
          '-headless'
        ],
      };
      */
      break;
  }
  return settings;
})({
  src_folders: ['tests/e2e/specs'],
  output_folder: 'tests/e2e/reports',
  custom_assertions_path: ['tests/e2e/custom-assertions'],
  selenium: {
    start_process: false,
  },
  test_settings: {
    default: {
      globals: {
        waitForConditionTimeout: 30000,
        waitForConditionPollInterval: 5000,
      },
      selenium_port: 4444,
      selenium_host: process.env.E2E_SELENIUM_HOST || 'localhost',
      silent: false,
      desiredCapabilities: {
        browserName: process.env.E2E_SELENIUM_BROWSER || 'chrome',
        javascriptEnabled: true,
        acceptSslCerts: true,
        acceptInsecureCerts: true,
      },
    },
    chrome: {},
  },
});
```

`package.json`의 `scripts`에는 적절하게 다음을 추가해주고
```
"test:e2e:remote": "vue-cli-service test:e2e --config nightwatch.remote.config.js",
```

그리고 `.gitlab-ci.yml` 의 일부
```yaml
image: node:12
stages:
  - test:unit
  - test:coverage
  - test:e2e
# 중간은 대충 생략
test:e2e:ui:firefox:
  stage: test:e2e
  services:
    - selenium/standalone-firefox
  variables:
    E2E_SELENIUM_BROWSER: firefox
    E2E_SELENIUM_HOST: selenium__standalone-firefox
  before_script:
    - npm i
  script:
    - npm run test:e2e:remote
test:e2e:ui:chrome:
  stage: test:e2e
  services:
    - selenium/standalone-chrome
  variables:
    E2E_SELENIUM_BROWSER: chrome
    E2E_SELENIUM_HOST: selenium__standalone-chrome
  before_script:
    - npm i
  script:
    - npm run test:e2e:remote
```

몇시간동안 삽질한 것 같다.

[^1]: kubernote도 된다고는 하는데 나는 도커가 편해서...

