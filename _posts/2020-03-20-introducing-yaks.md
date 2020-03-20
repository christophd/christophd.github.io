---
title: "Introducing YAKS"
categories:
  - Blog
---

YAKS is Cloud-Native BDD testing or simply "Yet Another Kubernetes Service".

# What is "YAKS"?

YAKS is an Open Source test automation platform to run your tests as Cloud-Native resources. This means it runs natively on
[Kubernetes](https://kubernetes.io/) and [OpenShift](https://www.openshift.com/) and is specifically designed to verify your serverless and Microservice components.
  
The tests that you write with YAKS follow the BDD (Behavior Driven Development) concepts so you can run 
[Gherkin](https://cucumber.io/docs/gherkin/reference/) (Given-When-Then) feature files as Pod in your cluster.

YAKS provides a specific operator that leverages the Operator SDK to perform operations on Kubernetes resources. Each time you provide
a test as a [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) the operator takes care on
preparing a JVM test runtime and executes the test as a Pod in the cluster.
  
The test framework also brings a set of ready-to-use [Cucumber](https://cucumber.io/) steps so you can just start writing 
your feature files and run them to verify your deployed services. 

## Why would you want to do that?

Why would you want to run tests as Cloud-Native resources in Kubernetes? Both Kubernetes and OpenShift have become standard target platforms for serverless 
and microservices architectures. Developing those architectures is different compared to what we have done for decades before that. 

Writing a serverless application for instance with [Camel K](https://camel.apache.org/projects/camel-k/) is very declarative. As a developer you write a single 
Camel route and run this via the Camel K operator in the Kubernetes cluster. In this approach the unit testing part that we have been used to over so many years 
gets more and more transformed into pure integration testing and end-to-end testing. Given by the nature of serverless architectures we rely on a given runtime infrastructure
and it is hard to run tests outside of that infrastructure.

Let us have a look at a sample Camel K integration:

Sample: _Camel K integration.groovy_
{% highlight groovy %}
// expose a rest endpoint that routes messages to a Kafka topic
rest().post("/resources")
  .route()
    .to("kafka:messages")
    
// transform all messages and publish them on a HTTP endpoint
from("kafka:messages")
  .transform()... // any kind of transformation
  .to("http://myendpoint/messages")
{% endhighlight %}

While we develop this integration we always run the Camel K integration source within the target infrastructure including its messaging transports, databases and other services. It is only natural 
to also move the verifying tests into this very same infrastructure. 

This way the tests can make use of all Kubernetes services including access to internal services, too. Also the tests are able to simulate 3rd party services 
(like the `http://myendpoint/messages` endpoint in the sample above) or other microservices that are part of the message processing logic in system under test.

In additional to all of that the BDD tests describe the given context, the events to occur and the expected outcome all in one single feature file. This declarative testing approach
is a perfect match to the concept of operators and custom resources on Kubernetes that is being used in so many Cloud-Native services these days.

## How does it work? 

YAKS provides a Kubernetes operator and a set of CRDs (custom resources) that we need to install in the cluster. The best way to install YAKS is to use the _yaks_ CLI tools 
that you can download on the [github release pages](https://github.com/citrusframework/yaks/releases).

Simply run:
```
$ yaks install
```         

This command prepares your Kubernetes cluster for running tests with YAKS. It will take care of installing the YAKS CRDs, setting up privileges and create the 
operator in the current namespace.

__Important:__ in some cluster configurations, you need to be a cluster admin to install a CRD (it’s a operation that should be done only once for the entire cluster). 
The _yaks_ binary will help you troubleshoot. If you want to work on a development cluster like Minishift or Minikube, you can easily follow the dev cluster setup guide.
                                                                              
Once the cluster is prepared and the operator installed in the current namespace we can start to write a first BDD feature:

File: _http-to-kafka.feature_
{% highlight gherkin %}
Feature: Http To Kafka

  Background:
    Given URL: https://camelk-sample.namespace.domain.example/api/resources
  
  Scenario: Get a result from API
    When send GET /
    Then receive HTTP 200 OK

{% endhighlight %}

You can run the test with:
```
$ yaks test http-to-kafka.feature
```                              

When you run this you will be provided with the test output from the Pod that is running your test in the current namespace on Kubernetes.

```
$ yaks test http-to-kafka.feature
    
[...] 

[INFO]
[INFO] CITRUS TEST RESULTS
[INFO]
[INFO]  http-to-kafka.feature:6 .................. SUCCESS
[INFO] 
[INFO] TOTAL:	1
[INFO] FAILED:	0 (0.0%)
[INFO] SUCCESS:	1 (100.0%)
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] Generated test report: target/citrus-reports/citrus-test-results.html

1 Scenarios (1 passed)
2 Steps (2 passed)
0m1.711s

[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.274 s - in org.citrusframework.yaks.YaksTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 6.453 s
[INFO] Finished at: 2020-03-06T15:30:53Z
[INFO] ------------------------------------------------------------------------
Test result: Passed
```                

Also you can revisit the test Pod outcome with:

```
$ kubectl get tests
```                

This is an example output you should get:

```
NAME            PHASE
http-to-kafka   Passed
```

The test we just have run calls the sample Camel K integration over Http. It invokes the Http request on the Camel K service and verifies the _Http 200 OK_ response code. The
test uses some of the ready-to-use steps for handling Http communication provided by YAKS. But before we have a closer look at the predefined YAKS steps that you can use in a feature file 
we have a closer look at what happens behind the scenes when running this test.

[<img src="{{ site.baseurl }}/assets/images/yaks-architecture.png" alt="YAKS architecture"/>]({{ site.baseurl }}/assets/images/yaks-architecture.png)

The `yaks` tool will sync your test code with a Kubernetes custom resource of Kind _Test_. The resource is named `http-to-kafka` (after the file name) in the current namespace. So every time 
you run the test the custom resource is updated and executed.

The YAKS operator is the component that makes all this possible by configuring all Kubernetes resources needed for running your tests.

Now lets have a look at the predefined YAKS step implementations that you can use out of the box.

# YAKS test steps

The test framework provides several ready-to-use [Cucumber](https://cucumber.io/) steps that you can just use in your feature files. These steps should enable you to verify serverless applications by
exchanging data over various messaging transports. Have a look at the predefined steps we have so far:

 Module | Description
 --- | ---
 yaks-standard      | Basic steps (e.g. for logging messages to the console)
 yaks-http          | Call Http REST endpoints and verify the response content
 yaks-swagger       | Import Swagger OpenAPI specifications and use defined operations and test data to call Http REST endpoints
 yaks-jdbc          | Connect to a database for updating data or verifying query result sets
 yaks-camel         | Create, start and stop [Apache Camel](https://camel.apache.org/) routes as part of the test. This opens access to all __200+__ Camel components for testing!
 yaks-camel-k       | Create and verify Camel K integrations in a test

The list of ready-to-use steps is constantly growing and you can also write your own steps and use them in a YAKS test! Have a look at the [examples](https://github.com/citrusframework/yaks/tree/master/examples) 
to see all those steps in action.

The steps provided in YAKS are implemented using the integration test framework [Citrus](https://citrusframework.org/). 

Citrus itself also has some ready-to-use steps (e.g. [Selenium steps](https://citrusframework.org/citrus/reference/2.8.0/html/index.html#selenium-steps) and 
[Docker steps](https://citrusframework.org/citrus/reference/2.8.0/html/index.html#docker-steps)). You can also use these steps as part of YAKS testing. 

# Demo

The following demo video shows an example of what you can do with YAKS. It has the YAKS operator already installed and invokes a system under test via Http 
verifying the response content. It continues to more complex scenarios where the test uses a Swagger OpenAPI specification for generating requests and test data.

Take a look:

<iframe width="560" height="315" src="https://www.youtube.com/embed/fR-UgzvZkuA" frameborder="0" allowfullscreen></iframe>

# What's next

This is just the start of a new testing platform for BDD testing in Cloud-Native environments! We are happy to receive feedback, ideas and of course contributions. Please add your thoughts 
on the github repository by opening [new issues](https://github.com/citrusframework/yaks/issues) or just share your appreciation for instance by [giving a star](https://github.com/citrusframework/yaks) on the repository. 
