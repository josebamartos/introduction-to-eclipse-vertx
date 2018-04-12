# [Post 1] My First Vert.x Application

Based on [My First Vert.x Application](https://developers.redhat.com/blog/2018/03/13/eclipse-vertx-first-application/) written by [Clement Escoffier ](https://developers.redhat.com/blog/author/cescoffier/).

All the exercises are supossing that their root directory is `post-1`.

## Project structure

Minimal Vert.x project structure

```text
.
├── pom.xml
├── src
│   ├── main
│   │   └── java
│   └── test
│       └── java
```

Create project structure:

```text
$ mkdir -p ./src/{main/test}/java
$ curl -O https://raw.githubusercontent.com/josebamartos/introduction-to-eclipse-vertx/master/post-1/pom.xml
```

The project configuration is stored into `pom.xml` file:
  - Property `vertx.verticle` instructs Vert.x to deploy this class when it starts.
  - `maven-compiler-plugin` is configured to use Java 8. Older Java versions will not work with Vert.x.
  - Mandatory dependencies
    - [vertx-core](https://mvnrepository.com/artifact/io.vertx/vertx-core)
  - Recommended dependencies for unit testing
    - [vertx-unit](https://mvnrepository.com/artifact/io.vertx/vertx-unit)
    - [junit](https://mvnrepository.com/artifact/junit/junit)
  - Mandatory plugins
    - [maven-compiler-plugin](https://mvnrepository.com/artifact/org.apache.maven.plugins/maven-compiler-plugin)
  - Recommended plugins
    - [vertx-maven-plugin](https://mvnrepository.com/artifact/io.fabric8/vertx-maven-plugin) or [maven-shade-plugin](https://mvnrepository.com/artifact/org.apache.maven.plugins/maven-shade-plugin) to package the application in a fat jar, a standalone executable Jar file containing all the dependencies required to run the application. We are using `vertx-maven-plugin` because is good enough.


## Developing the first verticle

`pom.xml` file describes the Verticle to deploy at startup:

```xml
<vertx.verticle>io.vertx.intro.first.MyFirstVerticle</vertx.verticle>
```

Create the directory structure for the Verticle:

```text
$ mkdir -p src/main/java/io/vertx/intro/first/
```

We will create the verticle in `src/main/java/io/vertx/intro/first/MyFirstVerticle.java` file with this content:

```java
package io.vertx.intro.first;
 
import io.vertx.core.AbstractVerticle;
import io.vertx.core.Future;
 
public class MyFirstVerticle extends AbstractVerticle {
 
    @Override
    public void start(Future fut) {
        vertx
            .createHttpServer()
            .requestHandler(r -&gt;
                r.response()
                 .end("
<h1>Hello from my first Vert.x application</h1>
 
"))
            .listen(8080, result -&gt; {
                if (result.succeeded()) {
                    fut.complete();
                } else {
                    fut.fail(result.cause());
                }
            });
    }
}
```

> This is actually our not fancy application. The class extends AbstractVerticle. In the Vert.x world, a verticle is a component. By extending AbstractVerticle, our class gets access to the vertx field, and the vertx instance on which the verticle is deployed.
>
> The start method is called when the verticle is deployed. We could also implement a stop method (called when the verticle is undeployed), but in this case Vert.x takes care of the garbage for us. The start method receives a Future object that lets us notify Vert.x when our start sequence has been completed or reports an error if anything bad happens. One of the particularities of Vert.x is its asynchronous and non-blocking aspect. When our verticle is going to be deployed it won’t wait until the start method has been completed. So, the Future parameter is important for notification of the completion. Notice that you can also implement a version of the start method without the Future parameter. In this case, Vert.x considers the verticle deployed when the start method returns.
>
> The start method creates an HTTP server and attaches a request handler to it. The request handler is a function (here a lambda), which is passed into the requestHandler method, and is called every time the server receives a request. In this code, we just reply Hello (Nothing fancy I told you). Finally, the server is bound to the 8080 port. As this may fail (because the port may already be used), we pass another lambda expression called with the result (and so are able to check whether or not the connection has succeeded). As mentioned above, it calls either fut.complete in case of success, or fut.fail to report an error.

We can run the project using maven:

```text
$ mvn compile vertx:run
```

Visit [localhost:8080](http://localhost:8080) and you will receive this response:

```text
$ curl -i http://localhost:8080
HTTP/1.1 200 OK
Content-Length: 47

<h1>Hello from my first Vert.x application</h1>
```


## Unit testing

We will create the test class in `src/test/java/io/vertx/intro/first/MyFirstVerticleTest.java` file with this content:

```text
package io.vertx.intro.first;
 
import io.vertx.core.Vertx;
import io.vertx.ext.unit.Async;
import io.vertx.ext.unit.TestContext;
import io.vertx.ext.unit.junit.VertxUnitRunner;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
 
@RunWith(VertxUnitRunner.class)
public class MyFirstVerticleTest {
 
    private Vertx vertx;
 
    @Before
    public void setUp(TestContext context) {
        vertx = Vertx.vertx();
        vertx.deployVerticle(MyFirstVerticle.class.getName(),
            context.asyncAssertSuccess());
    }
 
    @After
    public void tearDown(TestContext context) {
        vertx.close(context.asyncAssertSuccess());
    }
 
    @Test
    public void testMyApplication(TestContext context) {
        final Async async = context.async();
 
        vertx.createHttpClient().getNow(8080, "localhost", "/",
            response ->
                response.handler(body -> {
                    context.assertTrue(body.toString().contains("Hello"));
                    async.complete();
                }));
    }
}
```

> This is a JUnit test case for our verticle. The test uses vertx-unit, so we configure a custom runner (with the @RunWith annotation). vertx-unit makes easy to test asynchronous interactions, which are the basis of Vert.x applications.
>
>In the setUp method (called before each test), we create an instance of vertx and deploy our verticle. You may have noticed that unlike the traditional JUnit @Before method, it receives a TestContext object. This object lets us control the asynchronous aspect of our test. For instance, when we deploy our verticle, it starts asynchronously, as do most Vert.x interactions. We cannot check anything until it gets started correctly. So, as a second argument of the deployVerticle method, we pass a result handler: context.asyncAssertSuccess(). It fails the test if the verticle does not start correctly. In addition, it waits until the verticle has completed its start sequence. Remember, in our verticle, we call fut.complete(). So it waits until this method is called, and in the case of failures, fails the test.
>
>Well, the tearDown (called after each test) method is straightforward, and just closes the vertx instance we created.
>
>Let’s now have a look at the test of our application: the testMyApplication method. The test emits a request to our application and checks the result. Emitting the request and receiving the response is asynchronous. So we need a way to control this. Like the setUp and tearDown methods, the test method receives a TestContext. From this object we create an async handle (async) that lets us notify the test framework when the test has completed (using async.complete()).
>
>So, once the async handle is created, we create an HTTP client and emit an HTTP request handled by our application with the getNow() method (getNow is just a shortcut for get(...).end()). The response is handled by a handler. In this function, we retrieve the response body by passing another function to the handler method. The body argument is the response body (as a buffer object). We check that the body contains"Hello" and declare the test complete.
>
>Let’s take a minute to mention the assertions. Unlike in traditional JUnit tests, it uses context.assert…. Indeed, if the assertion fails, it will interrupt the test immediately. So it’s important to use these assertions because of the asynchronous aspect of the Vert.x application and tests. However, Vert.x Unit provides hooks to let you use Hamcrest or AssertJ, as demonstrated in this example and this other example.


We can test the project using maven:

```text
$ mvn clean test
```

## Creating and running Uber Jar

We can now build the fat jar:

```text
$ mvn clean package
```

And run it:

```text
$ java -jar target/my-first-app-1.0-SNAPSHOT.jar 
Apr 11, 2018 10:24:57 PM io.vertx.core.impl.launcher.commands.VertxIsolatedDeployer
INFO: Succeeded in deploying verticle
```





















