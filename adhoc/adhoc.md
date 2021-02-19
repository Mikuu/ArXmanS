UI Visual Regression Testing with Micoo
--

## Why UI Visual Regression Testing

Visual regression testing is a very effective method to check display on any user interface, e.g. Web page, mobile 
device screen, it helps to ensure the UI display match exactly the same as you wanted, and catch any tiny mismatch if 
there is and report it in a visualized way. Combine both functional testing and visual testing can make your UI tests 
much stronger than ever before.

There are lots of articles on the internet telling how UI visual regression testing provides the benefits to you, so I 
don't want to waste more time on this part in my article.

## Solutions for UI Visual Regression Testing

How many articles there on the internet telling why you should use visual regression testing, there would be same amount 
of them telling how you could do visual regression testing with different tools, but to summarize those information all, 
there is a great github repository [awesome-regression-testing](https://github.com/mojoaxel/awesome-regression-testing) 
which provides a full picture of all visual regression solutions, which just like its name, awesome.

Generally, all visual regression testing solutions can be divided into 2 dimensions:

- Tool & Framework vs Service
- Commercial vs Open Source

basically, most of the open source solutions, e.g. BackstopJS, AyeSpy, are tools or frameworks, while most of the service
solutions are commercial, e.g. Applitools, Percy.io, Diffy. No matter what ever the solution it is, currently, all 
non-AI-driven visual regression testing follow a similar working pattern as below:

![working-pattern](./image/working-pattern.png)

- `initialize base` is a one time task, it's only necessary for the first time setting up the whole process,
- `compareing screenshots` and `report mismatch` are automatic steps,
- `judge mismatch` and `update baseline` are manual steps.

Despite the similar working pattern, service based visual regression testing solutions are usually more convenient than 
testing tools & frameworks, e.g. updating baseline on visual testing services which usually only need click a button, is much 
easier than visual testing tools which usually need execute separate command. Also, visual testing services can provide 
comparison history to let you review back to previous screenshots, this is quite useful when there is any misjudgement happened.

## UI Visual Regression Testing with Micoo

Visual testing service is more convenient than visual testing tools, but there are quite few of open source visual testing
service, fortunately Micoo is one.

[Micoo](https://github.com/Mikuu/Micoo) is a simple self-host open source visual regression testing service, you can find 
all detailed information about it at its github homepage and its [document page](https://arxman.com/micoo/). 

The biggest characteristic and difference to any other visual regression testing solution is that Micoo doesn't take any 
screenshot from your SUT (Subject Under Test) by itself, Micoo only focuses on comparing screenshots and reporting mismatch.
This requires users to take screenshots in their own automation tests, which provides the flexibility to enable visual regression 
testing on the widest scope, e.g. different team can use their existing automation tests to take screenshots, no matter on 
Web pages, mobile device screens, or even desktop application UIs. As long as the SUT can be tested and taken screenshot by 
automation, it can be visual tested with Micoo.

### Practice with Micoo

Let's try an example to demonstrate how to do visual regression testing with Micoo.

#### Prepare SUT

Creating a UI application from ground is not in our scope for this article, so let's just pick up a public one from 
github to quickly launch our SUT.

```commandline
git clone https://github.com/devias-io/react-material-dashboard.git
cd react-material-dashboard
yarn
yarn start
```

then visit `http://localhost:3000` can get an awesome UI application like this:

![sut](./image/sut.png)

#### Set up Micoo service

To start visual regression testing with Micoo, we need to first launch Micoo service locally, this is quite easy with Micoo 
Docker images.

```commandline
git clone https://github.com/Mikuu/Micoo.git
cd Micoo/env
docker-compose up -d
```

Then Micoo should be ready for use, visit `http://localhost:8123`, we can get Micoo initialization page like this:

![initialization](./image/initialization.png)

This page only displays once when Micoo is first launched, the most important thing is to save the `passcode` displayed 
in this page, this passcode will be required to log in Micoo, Micoo doesn't provide any approach to retrieve this passcode 
again, if we lost this passcode, we can not log in to Micoo anymore. Another thing needs to be understood is that there is
no user management in Micoo, so this passcode need be shared with your team members who need access to Micoo. 

Although Micoo provides a basic passcode based authentication process, publishing your team's Micoo service to public 
internet is not a good idea, it's recommended hosting Micoo service somewhere inside your organization network boundary, 
e.g. not beyond your company firewall.

Click `Start` button will get the login page like this:

![login](./image/login.png)

Log in with the passcode from initialization page, then we can get the dashboard page like this:

![dashboard](./image/dashboard.png)

Let's create a Sandbox project which will be used later:

![dashboard.sandbox](./image/dashboard.sandbox.png)

So far, we have completed all necessary setup and initialization works for Micoo, what we need to do next is sending screenshots 
to a Micoo project, then it will trigger image comparison automatically.

#### Create automation tests scripts

Usually, the best practice of visual regression testing is combine it with functional regression testing, which is to take 
SUT screenshots within functional regression testing steps. 

Now, let's create some functional automation tests to cover our UI application pages. I will use Puppeteer here, but you 
can use any of your won favourite test tools & frameworks, e.g. Selenium, TestCafe, Cypress to do the similar things.

```commandline
mkdir tests
cd tests
npm init
npm install puppeteer
```

After installed puppeteer, create a simple test file `test.js` with below contents:

```javascript
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.setViewport({
    width: 1920,
    height: 1080,
  });

  await page.goto('http://localhost:3000/app/dashboard');
  await page.screenshot({ path: 'screenshots/dashboard.png' });

  await page.goto('http://localhost:3000/app/products');
  await page.screenshot({ path: 'screenshots/products.png' });

  await page.goto('http://localhost:3000/login');
  await page.screenshot({ path: 'screenshots/login.png' });

  await browser.close();
})();
```

The above test is quite straightforward, import puppetter, set viewport and goto different pages to take screenshots, all 
screenshots will be saved in a `screenshots` folder. Let's try this test:

```commandline
mkdir screenshots
node test.js
```

> frankly speaking, above test in `test.js` is absolutely not a functional test at all, because it doesn't make any assertion, 
> but since our purpose is to demonstrate visual testing, so let's ignore the normal functional check part here.

From here, we have a folder which contains some screenshots of our UI application, this is good enough to start our visual 
testing now.

#### Trigger visual regression testing

To trigger visual testing with Micoo, the next steps are uploading screenshots to Micoo and generating test build. Each time 
when we upload several screenshots to Micoo, it will automatically create a test build with those screenshots. To do this, 
we can use Micoo [client](https://github.com/Mikuu/Micoo/tree/master/clients).

I will use `micooc` node library here, but if you need, you can use Python and Java client library too.

```commandline
npm install --save-dev micooc
```

Create another javascript file `visual-test.js` with below content:

```javascript
const { newBuild } = require("micooc");

async function testNewBuild() {
  const host = "http://localhost:8123/engine";
  const apiKey = "AK005fca5cbc9779755f";
  const pid = "PID6fb00c63d17f4596ba831a299edd21b4";
  const buildVersion = "5fafc0478af24af2da45fa19ddd06c17dd5d0d45";
  const screenshotDirectory = "./screenshots";

  await newBuild(host, apiKey, pid, buildVersion, screenshotDirectory);
}

testNewBuild();
```

In above script, `apiKey` and `pid` can be found from Micoo project as below, `buildVersion` is something like our UI 
application's revision number, `screenshotDirectory` is just the folder where contains our SUT screenshots.

![pid-apiKey](./image/pid-apiKey.png)

Now, let's run our visual test script:

```commandline
node visual-test.js
```

It should upload our screenshots to Micoo successfully. Now, let's go to Sandbox project page in Micoo, we can see something 
like this:

![new-build](./image/new-build.png)

![new-build.tests](./image/new-build.tests.png)

Since this is our first build in this project, there is no baseline to compare with, so the build result and tests result 
are all `undetermined`. Then in each test page, click `passed` button to make a judgment like below:

![test.products.baseline](./image/test.products.baseline.png)

After all tests are set as passed, then in the build page, click `Rebase` button will make current build becomes a baseline, 
which means from next time, all latest screenshots will be compared with this build's screenshot, until we rebase another 
new baseline. Next to the `Rebase` button, there is a `Debase` button which can be used to remove a build's baseline position, 
this function is very useful when we rebase an incorrect build by mistake and need to cancel the rebasing.

![rebase](./image/rebase.png)

Back to project page again, we can see the current build result is passed now:

![new-build.rebased](./image/new-build.rebased.png)


#### Redo visual regression testing with no UI change

Let's consider that our SUT UI application have changed some logic in its React component or Redux implementation, 
but however, the UI should have no change at all, in this case, we can rerun our functional test to take screenshots and 
trigger visual testing again.

```commandline
rm screenshots/*.png
node test.js
node visual-test.js
```

Now, check the Sandbox project again in Micoo, we can see there is a new build generated and both its build result and all
tests results are all passed:

![2nd-build.noChange](./image/2nd-build.noChange.png)

![2nd-build.tests.noChange](./image/2nd-build.tests.noChange.png)

Looking at the detailed comparison page, baseline screenshot and latest screenshot are just the same:

![2nd-build.test.dashboard.noChange](./image/2nd-build.test.dashboard.noChange.png)

#### Redo visual regression testing with UI change

To simulate Micoo capturing UI change on the SUT screenshot, we need make something change to the SUT implementation. Let's 
do a fake change in the SUT source code as below:

![source-change](./image/source-change.png)

Then rerun our functional tests and visual tests again:

```commandline
rm screenshots/*.png
node test.js
node visual-test.js
```

Check Sandbox project again in Micoo, we could see the build, and a test are failed now:

![3rd-build.failed](./image/3rd-build.failed.png)

![3rd-build.test.failed](./image/3rd-build.test.failed.png)

And the detailed comparison page looks like below. Sometimes, since Micoo displays 3 images on one page, sometimes it's 
not convenient to find out the detailed mismatch element, then a good tip is to open baseline and latest screenshot into 
two new browser tabs, then switching between these tow tabs can help to locate the accurate mismatch places.

![3rd-build.test.failed.comparison](./image/3rd-build.test.failed.comparison.png)

This is how Micoo find the mismatch and visualize it, simple but straightforward.

#### Accept UI change

Micoo reports mismatch, mismatch expose UI change, but UI change not equal to defect, if the UI change is just a correct 
implementation, then it should be accepted, and a rebase should happen in the visual test. To achieve that, just click 
`Passed` button to the "Failed" test and rebase current build into a new baseline. These steps are quite easy on the Micoo 
UI. BTW, only all tests in a build have been set as passed, that build can then be rebase as a new baseline, that means 
you can't rebase a build with some of its tests are failed.

#### CI integration

Micoo is a standalone service, it doesn't provide any notification feature to tell us when a project build is completed 
and build result is passed or failed, this is somehow not convenient to CI integration. Micoo client ships 2 API call 
which can be used to fetch a specific build's result, or a project's latest build's result. With such APIs we can create 
script to retrieve test build's result in CI job.

Let's create a script `retrieve-visual-test-result.js` with below contents:

```javascript
const { latestBuildStats } = require("micooc");

const sleep = ms => {
    return new Promise(resolve => setTimeout(resolve, ms));
};

const check = async (host, apiKey, pid, timeoutInSeconds) => {
    let stats;

    for (let i = 0; i < Number(timeoutInSeconds); i++) {
        stats = await latestBuildStats(host, apiKey, pid);

        if (stats.status === "completed") {
            const vtsUrl = `${host}/build/${stats.bid}`;

            if (stats.result === "passed") {
                console.log(`Visual Testing All Match: ${vtsUrl}`);
                return;

            } else if (stats.result === "failed") {
                console.log(`Visual Testing Mismatch: ${vtsUrl}`);
                return process.exit(1);
            }
        }

        await sleep(1000);
    }
};

const main = async () => {
    const host = "http://localhost:8123";
    const apiKey = "AK2bb46793de6fda7339";
    const pid = "PID6cdc2fa83b024309a95cd7965f257af1";
    const timeoutInSeconds = 5;

    await check(host, apiKey, pid, timeoutInSeconds);
}

main();
```

In above script, we are using the `latestBuildStats` method in a loop to query the latest build result, then we can use 
this script in a Jenkins Job to construct a UI pipeline with visual regression testing. With all above 3 script, we can 
build an [async pipeline](https://arxman.com/micoo/#ci-setup) as below:

![async-pipeline](./image/async-pipeline.png)

## Summary

Micoo is a quite simple but straightforward visual regression testing solution, it can be self-hosted, and not binding to
any specific testing tools & frameworks, we can use Selenium/Puppeteer/TestCafe/Cypress + Micoo to implement web visual 
regression testing, we can use Appium/Detox + Micoo to implement iOS or Android visual regression testing, even Desktop 
application can also be visual tested if it can be taken screenshots properly.



