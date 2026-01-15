[![Promo](https://brightdata.co.kr/static/github_promo_15.png?md5=105367-daeb786e)](https://brightdata.co.kr/?promo=github15) 

# Node.js에서 프록시 서버를 사용하는 방법
`node-fetch`, Playwright, Puppeteer에 프록시를 통합하는 방법을 알아봅니다. 또한 Bright Data의 [レジデンシャル프록시](https://brightdata.co.kr/proxy-types/residential-proxies)를 Axios에서 사용하는 방법도 확인할 수 있습니다. 이 가이드는 [Bright Data blog](https://brightdata.co.kr/blog/how-tos/nodejs-proxy-servers)에서도 확인할 수 있습니다.

- [Requirements](#requirements)
  * [Set Up a Local Proxy Server](#set-up-a-local-proxy-server)
- [Integrating Proxies in Node.js](#integrating-proxies-in-nodejs)
  * [Local Proxy Integration With `node-fetch`](#local-proxy-integration-with-node-fetch)
  * [Local Proxy Integration in Playwright](#local-proxy-integration-in-playwright)
  * [Local Proxy Integration Using Puppeteer](#local-proxy-integration-using-puppeteer)
- [Bright Data Proxy Integration in Node.js](#bright-data-proxy-integration-in-nodejs)
  * [Residential Proxy Configuration](#residential-proxy-configuration)
  * [Axios Proxy Setup](#axios-proxy-setup)
  * [Testing IP Rotation](#testing-ip-rotation)

## Requirenments
머신에 Node.js가 설치되어 있는지 확인합니다. 설치되어 있지 않다면 [official site](https://nodejs.org/en/download)에서 설치 프로그램을 다운로드하고 실행한 다음 설치 마법사의 안내에 따라 설치합니다.

Node.js 프로젝트용 폴더를 만들고 해당 폴더로 이동한 뒤, 그 안에서 npm 애플리케이션을 초기화합니다:
```bash
mkdir <NODE_PROJECT_FOLDER_NAME>
cd <NODE_PROJECT_FOLDER_NAME>
npm init -y
```

### Set Up a Local Proxy Server
[mitmproxy](https://mitmproxy.org/)는 오픈 소스 대화형 HTTPS 프록시입니다. 이를 사용하여 로컬 프록시 서버를 설정합니다.

[OS용 installation guide](https://docs.mitmproxy.org/stable/overview-installation/)를 따라 mitmproxy를 설치한 다음 실행합니다:
```bash
mitmproxy
```
다음 인터페이스가 표시됩니다:
![The mitmproxy interface](resources/mitmproxy-interface.png)

이제 포트 `8080`에서 로컬로 수신 대기하는 로컬 프록시 서버가 준비되었습니다. 다음 명령으로 이를 확인합니다:

```
curl --proxy http://localhost:8080 "http://wttr.in/Paris?0"
```
**Note**: Windows에서는 `curl` 대신 `curl.exe`를 사용합니다.

결과는 다음과 비슷해야 합니다:
```
Weather report: Paris

                Overcast
       .--.     -2(-6) °C
    .-(    ).   ↙ 11 km/h
   (___.__)__)  10 km
                0.0 mm
```

mitmproxy 인터페이스로 돌아가서, 요청를 가로챈 것을 확인합니다:
![GET request intercepted by mitmproxy](resources/mitmproxy-interface-proxy.png)

## Integrating Proxies in Node.js
다음 기술을 사용하여 로컬 프록시 서버를 통해 사이트에 연결하는 Node.js 스크립트를 작성합니다:
* [`node-fetch`](https://www.npmjs.com/package/node-fetch)
* [Playwright](https://playwaright.dev/)
* [Puppeteer](https://pptr.dev/)

### Local Proxy Integration With `node-fetch`
`node-fetch`에서 프록시를 구성하려면 `http-proxy-agent` 라이브러리가 필요합니다.

다음으로 둘 다 설치합니다:
```bash
npm install node-fetch http-proxy-agent
```
**Note**: `node-fetch` v3.x는 ESM-only 모듈입니다. 동작하도록 하려면 `package.json`에 `"type"="module"`을 설정합니다.

`fetch()` 메서드를 사용하여 프록시 서버를 통해 요청를 전송합니다:
```javascript
// node-fetch-proxy.js

import fetch from "node-fetch";
import { HttpProxyAgent } from "http-proxy-agent";

async function fetchData(url) {
  try {
    // initialize the local proxy agent
    const proxyAgent = new HttpProxyAgent(
      "http://localhost:8080"
    );
    // connect to the target site through the
    // local proxy
    const response = await fetch(url, {
      agent: proxyAgent,
    });

    // retrieve the HTML returned by the
    // server and print it
    const data = await response.text();
    console.log(data);
  } catch (error) {
    console.error("Error fetching data:", error);
  }
}

fetchData("http://toscrape.com/");
```

### Local Proxy Integration in Playwright
프로젝트의 dependencies에 Playwright를 추가합니다:
```bash
npm install playwright
```

다음으로 Playwright 설치를 마무리합니다:
```bash
npx playwright install --with-deps
```
**Note**: 브라우저 바이너리와 그 dependencies도 설치하므로 시간이 다소 소요됩니다.

`playwright-proxy.js` 파일을 만들고 Playwright에 프록시를 통합합니다:
```javascript
// playwright-proxy.js

import { chromium } from "playwright";

(async () => {
  // create a Chromium instance with
  // the local proxy configuration
  const browser = await chromium.launch({
    proxy: {
      server: "http://localhost:8080",
    },
  });

  // add a new tab and connect to the
  // target page
  const page = await browser.newPage();
  await page.goto("http://toscrape.com/");

  // extract the HTML content of the page
  // and log it
  const content = await page.content();
  console.log(content);

  // release the browser resources
  await browser.close();
})();
```

### Local Proxy Integration Using Puppeteer
Puppeteer를 설치합니다:
```bash
npm install puppeteer
```

`puppeteer-proxy.js` 스크립트에서처럼 로컬 프록시를 Puppeteer에 통합합니다:
```javascript
// puppeteer-proxy.js

import puppeteer from "puppeteer";

(async () => {
    // pass the URL of the local proxy to
    // the `--proxy-server` flag to configure it in Chrome
    const browser = await puppeteer.launch({
        args: ["--proxy-server=http://localhost:8080"]
    });

    // open a new page and visit the target site
    const page = await browser.newPage();
    await page.goto("http://toscrape.com/");

    // retrieve the HTML content of the page
    // and log it
    const content = await page.content();
    console.log(content);

    // release the browse recources
    await browser.close();
})();
```

## Testing Proxy Integration in Node.js
다음으로 Node.js 프록시 통합 스크립트를 실행합니다:
```bash
node <NODE_SCRIPT_NAME>
```

다음 HTML이 출력됩니다:
```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <title>Scraping Sandbox</title>
        <link href="./css/bootstrap.min.css" rel="stylesheet">
        <link href="./css/main.css" rel="stylesheet">
    </head>
    <body>
    <!-- omitted for brevity ... -->
```

mitmproxy 인터페이스에는 스크립트가 수행한 모든 요청가 기록됩니다. `node-fetch`에서는 `http://toscrape.com/`에 대한 GET 요청만 표시됩니다. Playwright와 Puppeteer에서는 페이지의 JS 및 CSS 파일을 로드하기 위해 브라우저가 수행하는 요청도 포함됩니다:
![Browser requests in the mitmproxy interface](resources/mitmproxy-interface-requests.png)

## Bright Data Proxy Integration in Node.js
Bright Data는 출구 IP를 자동으로 로ーテーティング프록시해 주는 [premium proxies](https://brightdata.co.kr/proxy-types)를 제공합니다. Node.js 스크립트에서 Axios와 함께 이를 통합하는 방법을 살펴보겠습니다.

### Residential Proxy Configuration
무료 체험을 시작하려면 [Sign up for Bright Data](https://brightdata.co.kr/cp/start)합니다. "Proxies & Scraping Infrastructure" 대시보드로 이동하여 새 レジデンシャル프록시를 구성합니다.

다음 인증 정보를 가져옵니다:
* `<BRIGHTDATA_PROXY_HOST>`
* `<BRIGHTDATA_PROXY_PORT>`
* `<BRIGHTDATA_PROXY_USERNAME>`
* `<BRIGHTDATA_PROXY_PASSWORD>`

### Axios Proxy Setup
Axios를 설치합니다:
```bash
npm install axios
```

아래 `axios-proxy.js` 스크립트와 같이 Bright Data レジデンシャル프록시를 Axios에 통합합니다:
```javascript
import axios from "axios";

async function fetchDataWithBrightData(url) {
    // configuration to instruct Axios
    // to route the traffic through the specified proxy
    const proxyOptions = {
        proxy: {
            host: "<BRIGHTDATA_PROXY_HOST>",
            port: "<BRIGHTDATA_PROXY_PORT>",
            auth: {
                username: "<BRIGHTDATA_PROXY_USERNAME>",
                password: "<BRIGHTDATA_PROXY_PASSWORD>"
            }
        }
    };


    try {
        // connect to the target page
        // and log the server response
        const response = await axios.get(url, proxyOptions);
        console.log(response.data);
    } catch (error) {
        console.error('Error:', error);
    }
}

fetchDataWithBrightData("http://lumtest.com/myip.json");
```

### Testing IP Rotation
다음으로 Axios 프록시 통합 스크립트를 실행합니다:
```
node axios-proxy.js
```

`http://lumtest.com/myip.json`은 IP에 대한 정보를 반환하는 특별한 엔드포인트입니다.

스크립트를 여러 번 실행하면 매번 서로 다른 위치의 서로 다른 IPが表示됩니다.