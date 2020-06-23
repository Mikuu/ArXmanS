Micoo
--
![dashboard.png](./images/dashboard.png)

## About Micoo
[Micoo](https://github.com/Mikuu/Micoo) is a pixel based screenshots comparison solution for visual regression test, some characters Micoo provides:

* a web application, for inspecting test results, making visual mismatch decision and maintain baseline build,
* an engine service, for comparing the latest screenshots against baseline screenshots, based on pixel difference,
* a methodology, about how to do visual regression test with service,
* quick local setup and server side deployment with Docker Compose,

Micoo does `NOT`:
* take screenshots from your SUT application,
* process screenshots before doing visual comparison,
* provide Email notification for comparison mismatch,
* provide user management to distinguish `teams`,

So, what Micoo targets at is the most stable and straightforward function for comparing screenshots. Micoo is not, and probably, would never be a powerful thing, like `JVM`, but hope to be an always useful helper, like `string.replace()`.

## Installation

According to different purposes, there are 3 ways to launch Micoo as described below.

> all the below steps require the project [repository](https://github.com/Mikuu/Micoo) be cloned to your local, and all the commands are executed from the project's root path.

### launch with Docker images from Docker Hub

```commandline
cd env
docker-compose up
```

Then Micoo should be ready at `http://localhost:8123`.

This is the simplest approach to launch Micoo. You can always use the images from Docker Hub, unless you would like to launch Micoo with your own source code changes.

### launch with local source code

- create exchange directory

```commandline
mkdir exchange
npm install
```

- containerize micoo-nginx service

```commandline
cd env/nginx/containerize
./build-image.sh
```

- launch environment

```commandline
cd env
docker-compose -f docker-compose.env.yaml up
```

- start micoo-dashboard service

```commandline
cd dashboard
npm install
MICOO_DB_USERNAME=micoo-user export MICOO_DB_PASSWORD=micoo-password npm start
```

- start micoo-engine service

```commandline
cd engine
npm install
MICOO_DB_USERNAME=micoo-user export MICOO_DB_PASSWORD=micoo-password npm start
```

- start micoo-postern service

```commandline
cd postern
npm install
MICOO_DB_USERNAME=micoo-user export MICOO_DB_PASSWORD=micoo-password npm start
```

Then Micoo should be ready at `http://localhost:8123`.

With above commands, it should be successfully launch Micoo from the source code locally.

### launch with local docker images

once you have debugged your code change successfully, you can containerize them locally:

- containerize micoo-nginx service

```commandline
cd env/nginx/containerize
./build-image.sh
```

- containerize micoo-dashboard service

```commandline
cd dashboard
./build-image.sh
```

- containerize micoo-engine service

```commandline
cd engine
./build-image.sh
```

- containerize micoo-postern service

```commandline
cd postern
./build-image.sh
```

launch Micoo locally with your own docker images:

- launch all from local image

```commandline
cd env
docker-compose -f docker-compose.local.yaml up
```

Then Micoo should be ready at `http://localhost:8123`.

The last thing, if you wish, you can publish your own docker images to your private or cloud registry, e.g. AWS ECR, to share with teams.

## Usage

Once you have launched Micoo at your localhost, you could see its dashboard page at `http://localhost:8123` like this

![empty-board.png](./images/empty-board.png)

then, we can start our visual regression testing with Micoo.

### Create a Micoo project

First thing first, to create a new Micoo project, just input a project name and click the add button, you will get something like this

![new-project.png](./images/new-project.png)

you can click the newly created project card to get into the project page, so far, it's an empty project, no test build.

![empty-project.png](./images/empty-project.png)

### Prepare SUT application screenshots

Micoo is designed to only focus on screenshots comparison, so the screenshots need be prepared by yourself, most of the case, by your UI automation tests. For this demonstration, let's use Puppeteer to simulate a quick Web application UI automation test.

- install Puppeteer
```commandline
mkdir -p my-test/screenshots
cd my-test
npm init
npm i puppeteer
```

- create test script

generate a test script `ui-test.js` and put in the below code

```javascript
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch({headless: false});
  const page = await browser.newPage();

  await page.goto('https://www.bing.com');
  await page.screenshot({path: 'screenshots/test-bing.png'});

  await page.goto('https://www.microsoft.com');
  await page.screenshot({path: 'screenshots/test-microsoft.png'});

  await browser.close();
})();
```
this test open browser to visit bing and microsoft's home pages, take screenshots and save them in folder `screenshots`.
> the screenshot filename would be used as test case name in Micoo, MUST in the form of [a-zA-Z0-9-_]+.png.

### Upload screenshots to Micoo and trigger visual comparision

Once we have all test screenshots ready, we can upload them to Micoo and trigger visual testing. To achieve that, we need us Micoo's client library,

```commandline
npm i micooc
```
then, we can create another script `visual-test.js`, and put in below codes

```javascript
const { newBuild } = require("micooc");

function test() {
  const host = "http://localhost:8123/engine";
  const pid = "PIDd9c19675fc864b34a74b97232fcc338a";
  const buildVersion = "5fafc0478af24af2da45fa19ddd06c17dd5d0d45";
  const screenshotDirectory = "./screenshots";

  newBuild(host, pid, buildVersion, screenshotDirectory);
}

test();
``` 
there is only one function `newBuild` we need call, and provide it 4 parameters
* `host` - the Micoo's base URL plus `/engine`,
* `pid` - your Micoo project's PID, it can be found from the Micoo project page's URL,
* `buildVersion` - this build version is neither parts of Micoo, nor your UI automation test, it needs to be the version of you SUT application, most of the case, it's the git revision number. `buildVersion` is a useful setup of mappings between your visual tests and the SUT application. Anyway, technically, it's just a string which will be displayed in Micoo's project board, you can use anything which is meaningful to you.
* `screenshotDirectory` - the directory where contains all screenshots to be uploaded, only `.png` file will be uploaded.

it's time to upload the screenshots and start the visual testing

```commandline
node visual-test.js
```
in terminal, you would probably get something like this

![screenshots-uploaded.png](./images/screenshots-uploaded.png)

in Micoo project page, you would get something like this

![init-build.png](./images/init-build.png)

### Initialize Baseline





## Clients

## CI integration

## Postern
