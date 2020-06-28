# PACT JVM Example

These codes provide an example about how to do Contract Test with PACT JVM Junit, 
which uses Junit in Consumer side and Gradle task in Provider side, it will cover:

- Microservices examples created with Spring Boot.
- Example of one Provider to two Consumers.
- Write Consumer tests in different ways including using Basic Junit, Junit Rule and DSL method.
- Example of Provider State.
- Example of utilizing Pact Broker.


Contents
========
- [PACT JVM Example](#pact-jvm-example)
  * [1. Understand The Example Applications](#1-understand-the-example-applications)
    + [1.1. Example Provider](#11-example-provider)
    + [1.2. Example Consumer Miku](#12-example-consumer-miku)
    + [1.3. Example Consumer Nanoha](#13-example-consumer-nanoha)
  * [2. Contract Test between Provider and Consumer Miku](#2-contract-test-between-provider-and-consumer-miku)
    + [2.1. Create Test Cases](#21-create-test-cases)
      - [2.1.1. Basic Junit](#211-basic-junit)
      - [2.1.2. Junit Rule](#212-junit-rule)
      - [2.1.3. Junit DSL](#213-junit-dsl)
    + [2.2. Run the Tests at Consumer Miku side](#22-run-the-tests-at-consumer-miku-side)
    + [2.3. Publish Pacts to Pact Broker](#23-publish-pacts-to-pact-broker)
    + [2.4. Run the Contract Test at Provider side](#24-run-the-contract-test-at-provider-side)
  * [3. Gradle Configuration](#3-gradle-configuration)
  * [4. Contract Test between Provider and Consumer Nanoha](#4-contract-test-between-provider-and-consumer-nanoha)
    + [4.1. Preparation for Provider State](#41-preparation-for-provider-state)
    + [4.2. Create Test Case at Consumer Nanoha side](#42-create-test-case-at-consumer-nanoha-side)
    + [4.3. Run Contract Test at Provider side](#43-run-contract-test-at-provider-side)
      - [4.3.1. start Provider application](#431-start-provider-application)
      - [4.3.2. update Gradle configuration](#432-update-gradle-configuration)
      - [4.3.3. run the contract test](#433-run-the-contract-test)
  * [5. Break Something](#5-break-something)
    + [5.1. Break something in Provider](#51-break-something-in-provider)
    + [5.2. Retest](#52-retest)
  * [6. Contribution](#6-contribution)  
    

## 1. Understand The Example Applications
Clone the codes to your local, then you can find:

### 1.1. Example Provider
This is an API backend service which serves at http://localhost:8080/information, consumers
can retrieve some person information by calling this endpoint with a query parameter **name**,
to start the provider:

`./gradlew :example-provider:bootRun`

then call http://localhost:8080/information?name=Miku will get:

![](./images/provider.miku.png)

and call http://localhost:8080/information?name=Nanoha will get:

![](./images/provider.nanoha.png)

### 1.2. Example Consumer Miku
This is the first example consumer we called [Miku](https://en.wikipedia.org/wiki/Hatsune_Miku), to start it:

`./gradlew :example-consumer-miku:bootRun`

then visit http://localhost:8081/miku in your browser, you can get this:

![](./images/consumer.miku.png)

compare with Provider's payload and the information on the web page, you can find that the attributes `salary` and 
`nationality` are not used by Miku.

### 1.3. Example Consumer Nanoha
This is the second example consumer we called [Nanoha](http://nanoha.wikia.com/wiki/Nanoha_Takamachi), to start it:

`./gradlew :example-consumer-nanoha:bootRun`

then visit http://localhost:8082/nanoha in your browser, you can get this:

![image](./images/consumer.nanoha.png)

similar to Miku, Nanoha does not use the attribute `salary` neither but uses attribute `nationality`, so this 
is a little difference between the two consumers when consuming the response from the Provider's same endpoint.  


## 2. Contract Test between Provider and Consumer Miku

Now, it's time to look into the tests. 

> This README will not go through all tests line by line, because 
the tests themselves are very simple and straightforward, so I will only point out some highlights for 
each test. For detailed explanation about the codes, please refer the [official document](https://github.com/DiUS/pact-jvm/tree/be4a32b08ebbd89321dc37cbfe838fdac37774b3/pact-jvm-consumer-junit)

### 2.1. Create Test Cases

By the time this example is created, PACT JVM Junit provides 3 ways to write the pact test file 
at consumer side, the **Basic Junit**, the **Junit Rule** and **Junit DSL**.


#### 2.1.1. Basic Junit
`PactBaseConsumerTest.java`
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class PactBaseConsumerTest extends ConsumerPactTest {

    @Autowired
    ProviderService providerService;

    @Override
    @Pact(provider="ExampleProvider", consumer="BaseConsumer")
    public RequestResponsePact createPact(PactDslWithProvider builder) {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        return builder
                .given("")
                .uponReceiving("Pact JVM example Pact interaction")
                .path("/information")
                .query("name=Miku")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body("{\n" +
                        "    \"salary\": 45000,\n" +
                        "    \"name\": \"Hatsune Miku\",\n" +
                        "    \"nationality\": \"Japan\",\n" +
                        "    \"contact\": {\n" +
                        "        \"Email\": \"hatsune.miku@ariman.com\",\n" +
                        "        \"Phone Number\": \"9090950\"\n" +
                        "    }\n" +
                        "}")

                .toPact();
    }

    @Override
    protected String providerName() {
        return "ExampleProvider";
    }

    @Override
    protected String consumerName() {
        return "BaseConsumer";
    }

    @Override
    protected void runTest(MockServer mockServer, PactTestExecutionContext context) {
        providerService.setBackendURL(mockServer.getUrl());
        Information information = providerService.getInformation();
        assertEquals(information.getName(), "Hatsune Miku");
    }
}
```
The `providerService` is the same one used in consumer Miku, we just use it to do a self 
integration test, the purpose for this is to check if consumer Miku can handle the mocked 
response correctly, then ensure the Pact content created are just as we need before we send 
it to Provider.

`mockServer.getUrl()` can return the mock server's url, which is to be used in our handler.

#### 2.1.2. Junit Rule
`PactJunitRuleTest.java`
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class PactJunitRuleTest {

    @Autowired
    ProviderService providerService;

    @Rule
    public PactProviderRule mockProvider = new PactProviderRule("ExampleProvider", this);

    @Pact(consumer = "JunitRuleConsumer")
    public RequestResponsePact createPact(PactDslWithProvider builder) {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        return builder
                .given("")
                .uponReceiving("Pact JVM example Pact interaction")
                .path("/information")
                .query("name=Miku")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body("{\n" +
                        "    \"salary\": 45000,\n" +
                        "    \"name\": \"Hatsune Miku\",\n" +
                        "    \"nationality\": \"Japan\",\n" +
                        "    \"contact\": {\n" +
                        "        \"Email\": \"hatsune.miku@ariman.com\",\n" +
                        "        \"Phone Number\": \"9090950\"\n" +
                        "    }\n" +
                        "}")
                .toPact();
    }

    @Test
    @PactVerification
    public void runTest() {
        providerService.setBackendURL(mockProvider.getUrl());
        Information information = providerService.getInformation();
        assertEquals(information.getName(), "Hatsune Miku");
    }
}
```
This test uses Junit Rule which can simplify the writing of test cases comparing with the Basic Junit.

`PactJunitRuleMultipleInteractionsTest.java`
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class PactJunitRuleMultipleInteractionsTest {

    @Autowired
    ProviderService providerService;
    
    @Rule
    public PactProviderRule mockProvider = new PactProviderRule("ExampleProvider",this);

    @Pact(consumer="JunitRuleMultipleInteractionsConsumer")
    public RequestResponsePact createPact(PactDslWithProvider builder) {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        return builder
                .given("")
                .uponReceiving("Miku")
                .path("/information")
                .query("name=Miku")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body("{\n" +
                        "    \"salary\": 45000,\n" +
                        "    \"name\": \"Hatsune Miku\",\n" +
                        "    \"nationality\": \"Japan\",\n" +
                        "    \"contact\": {\n" +
                        "        \"Email\": \"hatsune.miku@ariman.com\",\n" +
                        "        \"Phone Number\": \"9090950\"\n" +
                        "    }\n" +
                        "}")
                .given("")
                .uponReceiving("Nanoha")
                .path("/information")
                .query("name=Nanoha")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body("{\n" +
                        "    \"salary\": 80000,\n" +
                        "    \"name\": \"Takamachi Nanoha\",\n" +
                        "    \"nationality\": \"Japan\",\n" +
                        "    \"contact\": {\n" +
                        "        \"Email\": \"takamachi.nanoha@ariman.com\",\n" +
                        "        \"Phone Number\": \"9090940\"\n" +
                        "    }\n" +
                        "}")
                .toPact();
    }

    @Test
    @PactVerification()
    public void runTest() {
        providerService.setBackendURL(mockProvider.getUrl());
        Information information = providerService.getInformation();
        assertEquals(information.getName(), "Hatsune Miku");

        providerService.setBackendURL(mockProvider.getUrl(), "Nanoha");
        information = providerService.getInformation();
        assertEquals(information.getName(), "Takamachi Nanoha");
    }
}
```
This case uses Junit Rule too, but with two interactions in one Pact file.

#### 2.1.3. Junit DSL
`PactJunitDSLTest`
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class PactJunitDSLTest {

    @Autowired
    ProviderService providerService;

    private void checkResult(PactVerificationResult result) {
        if (result instanceof PactVerificationResult.Error) {
            throw new RuntimeException(((PactVerificationResult.Error) result).getError());
        }
        assertThat(result, is(instanceOf(PactVerificationResult.Ok.class)));
    }

    @Test
    public void testPact1() {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        RequestResponsePact pact = ConsumerPactBuilder
                .consumer("JunitDSLConsumer1")
                .hasPactWith("ExampleProvider")
                .given("")
                .uponReceiving("Query name is Miku")
                .path("/information")
                .query("name=Miku")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body("{\n" +
                        "    \"salary\": 45000,\n" +
                        "    \"name\": \"Hatsune Miku\",\n" +
                        "    \"nationality\": \"Japan\",\n" +
                        "    \"contact\": {\n" +
                        "        \"Email\": \"hatsune.miku@ariman.com\",\n" +
                        "        \"Phone Number\": \"9090950\"\n" +
                        "    }\n" +
                        "}")
                .toPact();

        MockProviderConfig config = MockProviderConfig.createDefault();
        PactVerificationResult result = runConsumerTest(pact, config, (mockServer, context) -> {
            providerService.setBackendURL(mockServer.getUrl(), "Miku");
            Information information = providerService.getInformation();
            assertEquals(information.getName(), "Hatsune Miku");
            return null;
        });

        checkResult(result);
    }

    @Test
    public void testPact2() {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        RequestResponsePact pact = ConsumerPactBuilder
                .consumer("JunitDSLConsumer2")
                .hasPactWith("ExampleProvider")
                .given("")
                .uponReceiving("Query name is Nanoha")
                .path("/information")
                .query("name=Nanoha")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body("{\n" +
                        "    \"salary\": 80000,\n" +
                        "    \"name\": \"Takamachi Nanoha\",\n" +
                        "    \"nationality\": \"Japan\",\n" +
                        "    \"contact\": {\n" +
                        "        \"Email\": \"takamachi.nanoha@ariman.com\",\n" +
                        "        \"Phone Number\": \"9090940\"\n" +
                        "    }\n" +
                        "}")
                .toPact();

        MockProviderConfig config = MockProviderConfig.createDefault();
        PactVerificationResult result = runConsumerTest(pact, config, (mockServer, context) -> {
            providerService.setBackendURL(mockServer.getUrl(), "Nanoha");
            Information information = providerService.getInformation();
            assertEquals(information.getName(), "Takamachi Nanoha");
            return null;
        });

        checkResult(result);
    }
}
```
Comparing with Basic Junit and Junit Rule usage, the DSL provides the ability to create multiple Pact files 
in one test class.

`PactJunitDSLJsonBodyTest`
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class PactJunitDSLJsonBodyTest {

    @Autowired
    ProviderService providerService;

    private void checkResult(PactVerificationResult result) {
        if (result instanceof PactVerificationResult.Error) {
            throw new RuntimeException(((PactVerificationResult.Error) result).getError());
        }
        assertThat(result, is(instanceOf(PactVerificationResult.Ok.class)));
    }

    @Test
    public void testWithPactDSLJsonBody() {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        DslPart body = new PactDslJsonBody()
                .numberType("salary", 45000)
                .stringType("name", "Hatsune Miku")
                .stringType("nationality", "Japan")
                .object("contact")
                .stringValue("Email", "hatsune.miku@ariman.com")
                .stringValue("Phone Number", "9090950")
                .closeObject();

        RequestResponsePact pact = ConsumerPactBuilder
                .consumer("JunitDSLJsonBodyConsumer")
                .hasPactWith("ExampleProvider")
                .given("")
                .uponReceiving("Query name is Miku")
                .path("/information")
                .query("name=Miku")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body(body)
                .toPact();

        MockProviderConfig config = MockProviderConfig.createDefault(PactSpecVersion.V3);
        PactVerificationResult result = runConsumerTest(pact, config, (mockServer, context) -> {
            providerService.setBackendURL(mockServer.getUrl());
            Information information = providerService.getInformation();
            assertEquals(information.getName(), "Hatsune Miku");
            return null;
        });

        checkResult(result);
    }

    @Test
    public void testWithLambdaDSLJsonBody() {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        DslPart body = newJsonBody((root) -> {
            root.numberValue("salary", 45000);
            root.stringValue("name", "Hatsune Miku");
            root.stringValue("nationality", "Japan");
            root.object("contact", (contactObject) -> {
                contactObject.stringMatcher("Email", ".*@ariman.com", "hatsune.miku@ariman.com");
                contactObject.stringType("Phone Number", "9090950");
            });
        }).build();

        RequestResponsePact pact = ConsumerPactBuilder
                .consumer("JunitDSLLambdaJsonBodyConsumer")
                .hasPactWith("ExampleProvider")
                .given("")
                .uponReceiving("Query name is Miku")
                .path("/information")
                .query("name=Miku")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body(body)
                .toPact();

        MockProviderConfig config = MockProviderConfig.createDefault(PactSpecVersion.V3);
        PactVerificationResult result = runConsumerTest(pact, config, (mockServer, context) -> {
            providerService.setBackendURL(mockServer.getUrl());
            Information information = providerService.getInformation();
            assertEquals(information.getName(), "Hatsune Miku");
            return null;
        });

        checkResult(result);
    }
}
```
When use Json Body in DSL usage, we can control the test accuracy by defining whether we check the response 
body's attributes by Value or by Type, or even with Regular Expression. Comparing with normal Json body, the 
Lambda statements can avoid using `.close**()` methods to make the codes more clean. You can find more 
description about it [here](https://github.com/DiUS/pact-jvm/tree/be4a32b08ebbd89321dc37cbfe838fdac37774b3/pact-jvm-consumer-java8).  

### 2.2. Run the Tests at Consumer Miku side
Because we are using Junit, so to run the tests and create Pact files are very easy, just as what we always 
run our usual Unit Test:

`./gradlew :example-consumer-miku:clean test`

After that, you can find 7 JSON files created in folder `Pacts\Miku`. These are the Pacts which contain the 
contract between Miku and Provider, and these Pacts will be used to drive the Contract Test later at Provider 
side.

### 2.3. Publish Pacts to Pact Broker
The JSON files generated with task `test` are in the local folder which are only reachable to our local Provider, 
while for real project practice, it's highly recommended to use [Pact Broker](https://github.com/pact-foundation/pact_broker) 
to transport the Pacts between Consumers and Providers.

> There's a `docker-compose.yml` file that aids setting up an instance of the Pact Broker.
> Run `docker-compose up` and you'll have a Broker running at `http://localhost/`.
> It'll set up an instance of PostgreSQL as well, but the data will be lost at every restart.
> In order to use it, in `build.gradle`, set `pactBrokerUrl` to `http://localhost` and both `pactBrokerUsername` and `pactBrokerPassword` to `''`.

After you set up a Pact Broker server for your own, you can easily share your Pacts to the broker with the `pactPublish` command:

`./gradlew :example-consumer-miku:pactPublish`

This command will upload your Pacts to the broker server:

```commandline
> Task :example-consumer-miku:pactPublish 
Publishing JunitDSLConsumer1-ExampleProvider.json ... HTTP/1.1 200 OK
Publishing JunitDSLJsonBodyConsumer-ExampleProvider.json ... HTTP/1.1 200 OK
Publishing JunitDSLLambdaJsonBodyConsumer-ExampleProvider.json ... HTTP/1.1 200 OK
Publishing BaseConsumer-ExampleProvider.json ... HTTP/1.1 200 OK
Publishing JunitRuleConsumer-ExampleProvider.json ... HTTP/1.1 200 OK
Publishing JunitRuleMultipleInteractionsConsumer-ExampleProvider.json ... HTTP/1.1 200 OK
Publishing JunitDSLConsumer2-ExampleProvider.json ... HTTP/1.1 200 OK
```

Then you can find the *Relationships* in [our Pact Broker](https://ariman.pact.dius.com.au/)

> you can find their are '7' consumers to 1 provider, in real project, it should NOT like that, because we have only one 
consumer Miku here, it should be only 1 consumer to 1 provider, while in this example, making it's '7' is only to show 
that how Pact Broker can display the relationships beautifully.

![image](./images/pact-broker.png)

Later, our Provider can fetch these Pacts from broker to drive the Contract Test.

### 2.4. Run the Contract Test at Provider side
We are using the Pact Gradle task in Provider side to run the Contract Test, which can be very easy without 
writing any code, just execute the `pactVerify` task. **Before we run the test, make sure the Provider API is already started and running at our localhost.**

Then we can run the test as:

`./gradlew :example-provider:pactVerify`

and you can find the test results at the command line, would be something likes:

```commandline
Arimans-MacBook-Pro:pact-jvm-example ariman$ ./gradlew :example-provider:pactVerify                                          

> Task :example-provider:pactVerify_ExampleProvider 

Verifying a pact between Miku - Base contract and ExampleProvider
  [Using File /Users/ariman/Workspace/Pacting/pact-jvm-example/Pacts/Miku/BaseConsumer-ExampleProvider.json]
  Given 
         WARNING: State Change ignored as there is no stateChange URL
  Consumer Miku
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json;charset=UTF-8" (OK)
      has a matching body (OK)
  Given 
         WARNING: State Change ignored as there is no stateChange URL
  Pact JVM example Pact interaction
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json;charset=UTF-8" (OK)
      has a matching body (OK)
      
  ...
  
  Verifying a pact between JunitRuleMultipleInteractionsConsumer and ExampleProvider
    [from Pact Broker https://ariman.pact.dius.com.au/pacts/provider/ExampleProvider/consumer/JunitRuleMultipleInteractionsConsumer/version/1.0.0]
    Given 
           WARNING: State Change ignored as there is no stateChange URL
    Miku
      returns a response which
        has status code 200 (OK)
        includes headers
          "Content-Type" with value "application/json;charset=UTF-8" (OK)
        has a matching body (OK)
    Given 
           WARNING: State Change ignored as there is no stateChange URL
    Nanoha
      returns a response which
        has status code 200 (OK)
        includes headers
          "Content-Type" with value "application/json;charset=UTF-8" (OK)
        has a matching body (OK)
  

```

You can find the tests checked the Pacts from both local and remote Pact Broker.

> Since this example is open to all Pact learners, means everyone can upload their own Pacts to this Pact Broker, including 
correct and incorrect pacts, so don't be surprised if your tests run failed with the pacts from this Pact Broker. You can 
always upload correct pacts to the broker, but just don't rely on it, the pacts in this broker may be cleared at anytime 
 for reset a clean environment.

## 3. Gradle Configuration

Before we continue to Consumer Nanoha, let's look at the Gradle Configuration first:

```groovy
project(':example-consumer-miku') {
    ...
    test {
        systemProperties['pact.rootDir'] = "$rootDir/Pacts/Miku"
    }
    
    pact {
            publish {
                pactDirectory = "$rootDir/Pacts/Miku"
                pactBrokerUrl = mybrokerUrl
                pactBrokerUsername = mybrokerUser
                pactBrokerPassword = mybrokerPassword
            }
    }
    ...
}


project(':example-consumer-nanoha') {
    ...
    test {
        systemProperties['pact.rootDir'] = "$rootDir/Pacts/Nanoha"
    }
    ...
}

import java.net.URL

project(':example-provider') {
    ...
    pact {
            serviceProviders {
                ExampleProvider {
                    protocol = 'http'
                    host = 'localhost'
                    port = 8080
                    path = '/'
    
                    // Test Pacts from local Miku
                    hasPactWith('Miku - Base contract') {
                        pactSource = file("$rootDir/Pacts/Miku/BaseConsumer-ExampleProvider.json")
                    }
    
                    hasPactsWith('Miku - All contracts') {
                        pactFileLocation = file("$rootDir/Pacts/Miku")
                    }
    
                    // Test Pacts from Pact Broker
                    hasPactsFromPactBroker(mybrokerUrl, authentication: ['Basic', mybrokerUser, mybrokerPassword])
    
                    // Test Pacts from local Nanoha
    //                hasPactWith('Nanoha - With Nantionality') {
    //                    pactSource = file("$rootDir/Pacts/Nanoha/ConsumerNanohaWithNationality-ExampleProvider.json")
    //                }
    
    //                hasPactWith('Nanoha - No Nantionality') {
    //                    stateChangeUrl = new URL('http://localhost:8080/pactStateChange')
    //                    pactSource = file("$rootDir/Pacts/Nanoha/ConsumerNanohaNoNationality-ExampleProvider.json")
    //                }
                }
            }
        }
    }
```

Here we have configuration separately for Consumer Nanoha, Consumer Miku and Provider.

- `systemProperties['pact.rootDir']` defines the path where consumer Miku and Nanoha generate their Pacts files locally. 


- `pact { ... }` defines the the Pact Broker URL and where to fetch pacts to upload to the broker.
 
- Then in Provider configuration, `hasPactWith()` and `hasPactsWith()` specify the local path to find Pact files, 
and `hasPactsFromPactBroker` specify the remote Pact Broker to fetch Pact files.
 
 *Why comment Nanoha's Pacts path?* Because we haven't created Nanoha's Pacts, so it will raise exception with 
 that configuration, we can uncomment that path later after we created the Nanoha's Pacts files.

## 4. Contract Test between Provider and Consumer Nanoha

The Contract Test between Nanoha and Provider is similar to Miku, the difference I want to demonstrate here is 
**Provider State**, you can find more detailed explanation about the Provider State [here](https://docs.pact.io/documentation/provider_states.html).

### 4.1. Preparation for Provider State

In our example Provider, the returned payload for both Miku and Nanoha have a property `.nationality`, this kind 
of properties values, in real project, could always be queried from database or dependency service, but in this 
example, to simplify the Provider application logic, I just create a static property to simulate the data storage:

`provider.ulti.Nationality`

```java
public class Nationality {
    private static String nationality = "Japan";

    public static String getNationality() {
        return nationality;
    }

    public static void setNationality(String nationality) {
        Nationality.nationality = nationality;
    }
}
```

Then we can change the `.nationality` value at anytime to achieve our test purpose. To change its value, I created 
a `PactController` which can accept and reset the state value by sending a POST to endpoint `/pactStateChange`:

`provider.PactController`

```java
@Profile("pact")
@RestController
public class PactController {

    @RequestMapping(value = "/pactStateChange", method = RequestMethod.POST)
    public PactStateChangeResponseDTO providerState(@RequestBody PactState body) {
        switch (body.getState()) {
            case "No nationality":
                Nationality.setNationality(null);
                System.out.println("Pact State Change >> remove nationality ...");
                break;
            case "Default nationality":
                Nationality.setNationality("Japan");
                System.out.println("Pact Sate Change >> set default nationality ...");
                break;
        }

        // This response is not mandatory for Pact state change. The only reason is the current Pact-JVM v4.0.4 does
        // check the stateChange request's response, more exactly, checking the response's Content-Type, couldn't be
        // null, so it MUST return something here.
        PactStateChangeResponseDTO pactStateChangeResponse = new PactStateChangeResponseDTO();
        pactStateChangeResponse.setState(body.getState());

        return pactStateChangeResponse;
    }
}
```

Because this controller should only be available at **Non-Production environment**, so I added a Profile annotation 
to limit that this endpoint is only existing when Provider application is running with a "pact" profile, in another 
words, it only works in test environment.

**So, to summary all above things, when Provider is running with a "pact" profile, it serves an endpoint `/pactStateChange` 
which accept a POST request to set the `.nationality` value to be "Japan" (by default) or "null"**. 

### 4.2. Create Test Case at Consumer Nanoha side

The test cases at Nanoha side is using DSL Lambda, here we have two cases in our test, the main difference between 
them is the expectation for `.nationality` and the Provider State we set in `.given()`.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class NationalityPactTest {

    @Autowired
    ProviderService providerService;

    private void checkResult(PactVerificationResult result) {
        if (result instanceof PactVerificationResult.Error) {
            throw new RuntimeException(((PactVerificationResult.Error) result).getError());
        }
        assertThat(result, is(instanceOf(PactVerificationResult.Ok.class)));
    }

    @Test
    public void testWithNationality() {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        DslPart body = newJsonBody((root) -> {
            root.numberType("salary");
            root.stringValue("name", "Takamachi Nanoha");
            root.stringValue("nationality", "Japan");
            root.object("contact", (contactObject) -> {
                contactObject.stringMatcher("Email", ".*@ariman.com", "takamachi.nanoha@ariman.com");
                contactObject.stringType("Phone Number", "9090940");
            });
        }).build();

        RequestResponsePact pact = ConsumerPactBuilder
                .consumer("ConsumerNanohaWithNationality")
                .hasPactWith("ExampleProvider")
                .given("")
                .uponReceiving("Query name is Nanoha")
                .path("/information")
                .query("name=Nanoha")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body(body)
                .toPact();

        MockProviderConfig config = MockProviderConfig.createDefault(PactSpecVersion.V3);
        PactVerificationResult result = runConsumerTest(pact, config, (mockServer, context) -> {
            providerService.setBackendURL(mockServer.getUrl());
            Information information = providerService.getInformation();
            assertEquals(information.getName(), "Takamachi Nanoha");
            assertEquals(information.getNationality(), "Japan");
            return null;
        });

        checkResult(result);
    }

    @Test
    public void testNoNationality() {
        Map<String, String> headers = new HashMap<String, String>();
        headers.put("Content-Type", "application/json;charset=UTF-8");

        DslPart body = newJsonBody((root) -> {
            root.numberType("salary");
            root.stringValue("name", "Takamachi Nanoha");
            root.stringValue("nationality", null);
            root.object("contact", (contactObject) -> {
                contactObject.stringMatcher("Email", ".*@ariman.com", "takamachi.nanoha@ariman.com");
                contactObject.stringType("Phone Number", "9090940");
            });
        }).build();

        RequestResponsePact pact = ConsumerPactBuilder
                .consumer("ConsumerNanohaNoNationality")
                .hasPactWith("ExampleProvider")
                .given("No nationality")
                .uponReceiving("Query name is Nanoha")
                .path("/information")
                .query("name=Nanoha")
                .method("GET")
                .willRespondWith()
                .headers(headers)
                .status(200)
                .body(body)
                .toPact();

        MockProviderConfig config = MockProviderConfig.createDefault(PactSpecVersion.V3);
        PactVerificationResult result = runConsumerTest(pact, config, (mockServer, context) -> {
            providerService.setBackendURL(mockServer.getUrl());
            Information information = providerService.getInformation();
            assertEquals(information.getName(), "Takamachi Nanoha");
            assertNull(information.getNationality());
            return null;
        });

        checkResult(result);
    }
}
```

To run the test:

```commandline
./gradlew :example-consumer-nanoha:clean test
```

Then you can find two Pacts files generated at `Pacts\Nanoha`.

### 4.3. Run Contract Test at Provider side

#### 4.3.1. start Provider application

As we mentioned above, before we running our test, we need start Provider application with profile "pact" (if your 
Provider application is already started when you ran test with Miku, please kill it first), the command 
is:

```commandline
SPRING_PROFILES_ACTIVE=pact ./gradlew :example-provider:bootRun
```

#### 4.3.2. update Gradle configuration

it's time to uncomment the Pacts file searching path in Provider configuration:

`build.gralde`

```groovy

    hasPactWith('Nanoha - With Nantionality') {
        pactSource = file("$rootDir/Pacts/Nanoha/ConsumerNanohaWithNationality-ExampleProvider.json")
    }

    hasPactWith('Nanoha - No Nantionality') {
        stateChangeUrl = new URL('http://localhost:8080/pactStateChange')
        pactSource = file("$rootDir/Pacts/Nanoha/ConsumerNanohaNoNationality-ExampleProvider.json")
    }
```

The first Pact will check the payload with default `.nationality` which value is `Japan`, while the second Pact, we 
define a `stateChangeUrl` which is implemented by `PactController` in Provider as we talked above, and the POST body 
is defined in `.given()` in the Pact. 

#### 4.3.3. run the contract test

the same command as we did with Consumer Miku:

```commandline
./gradlew :example-provider:pactVerify
```

Then you can get the test log in the same terminal.

## 5. Break Something

Is everything done? Yes, if you followed all above introductions until here, you should be able to create your Contract 
Test by yourself. But before you go to real, let's try to break something to see how Pact could find the defect.

### 5.1. Break something in Provider

Both Miku and Nanoha consume Provider's response for a property `.name`, in real project, there might be a case that 
Provider would like to change this property name to `.fullname`, this could be a classic Contract Breaking which would 
be captured by Pact. 

To do that, we need modify several lines of codes in Provider which could be a little difficult to 
understand, especially for a some testers who are not familiar to Spring Boot application. 

So to make the breaking simple, let's just force Provider returns `null` as the value to `.name` to all consumers, this can be easily done by 
adding only a single line:

`provider.InformationController`
```java
@RestController
public class InformationController {
        ...
        
        information.setName(null);
        return information;
    }
}
```

### 5.2. Retest

Restart your Provider application and run the test again, you can get the test failures as this:

```commandline
...

Verifying a pact between Nanoha - With Nantionality and ExampleProvider
  [Using File /Users/Biao/Workspace/Pacting/Pact-JVM-Example/Pacts/Nanoha/ConsumerNanohaWithNationality-ExampleProvider.json]
  Given
         WARNING: State Change ignored as there is no stateChange URL
  Query name is Nanoha
    returns a response which
      has status code 200 (OK)
      has a matching body (FAILED)

Failures:

0) Verifying a pact between Nanoha - With Nantionality and ExampleProvider - Query name is Nanoha  Given  returns a response which has a matching body
      Verifying a pact between Nanoha - With Nantionality and ExampleProvider - Query name is Nanoha  Given  returns a response which has a matching body=BodyComparisonResult(mismatches={$.nationality=[BodyMismatch(expected="Japan", actual=null, mismatch=Expected "Japan" but received null, path=$.nationality, diff=null)], $.name=[BodyMismatch(expected="Takamachi Nanoha", actual=null, mismatch=Expected "Takamachi Nanoha" but received null, path=$.name, diff=null)]}, diff=[{, -  "nationality": "Japan",, +  "salary": 80000,, +  "name": null,, +  "nationality": null,,   "contact": {,     "Phone Number": "9090940", -  },, -  "name": "Takamachi Nanoha",, -  "salary": 294875537, +  }, }])

...
```

## 6. Contribution
I'm not always tracking this example repository with the latest Pact-JVM version, so if you find there is new Pact-JVM released, and you'd like to put it
into this example, any PR is welcome.
