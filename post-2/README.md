# [Post 2] Eclipse Vert.x Application Configuration

Based on [Eclipse Vert.x Application Configuration](https://developers.redhat.com/blog/2018/03/22/eclipse-vert-x-application-configuration/) written by [Clement Escoffier](https://developers.redhat.com/blog/author/cescoffier/).

All the exercises are supossing that their root directory is `post-2`.


## Making application configurable

In `post-1`, we set into `MyFirstVerticle` class port `8080` as default value:

```java
.listen(8080, result -> {
```

But here in `post-2`, we will use the `conf` object that extends `JsonObject`class. This object is created from a JSON configuration file where the port number is set into the parameter `HTTP_PORT`. We also manually set `8080` as default value.

```java
.listen(config().getInteger("HTTP_PORT", 8080), result -> {
```

The first step is to modify the io.vertx.intro.first.MyFirstVerticle class to not bind to the port 8080, but to read it from the configuration:

```java
package io.vertx.intro.first;

public class MyFirstVerticle extends AbstractVerticle {
    
    @Override
    public void start(Future<Void> fut) {
        vertx
            .createHttpServer()
            .requestHandler(r ->
                r.response().end("<h1>Hello from my first Vert.x application</h1>"))
            .listen(config().getInteger("HTTP_PORT", 8080), result -> {
                if (result.succeeded()) {
                    fut.complete();
                } else {
                    fut.fail(result.cause());
                }
            });
    }
}
```

We can now build and run the application without setting a custom port and it will use 8080:

```text
$ mvn clean package
$ java -jar target/my-first-app-1.0-SNAPSHOT.jar
```

`AbstractVerticleClass` provides the method `config()` which returns the configuration as instance of `JSONObject`. We didn't load any configuration field yet, so `config()` function will only return null references. The application is working because we set `8080` as default value for `config.getInteger()` method, if we didn't, we would get a `NullPointerException` at application startup.


## API-based Configuration:

Previously we parameterized the code the be able to read the port from configuration. Now we will learn how to run tests to run `MyFirstVerticle` using a different port for testing purproses.

1. First we will learn how to ser a custom port to run the tests.
2. After that we will learn how to get a unused port to run the tests.


### Custom port for tests

In `post-1`, we deployed `MyFirstVerticle` for testing through `MyFirstVerticleTest` class port `8080` as default value:

```java
vertx.deployVerticle(MyFirstVerticle.class.getName(), context.asyncAssertSuccess());
```

Here in `post-2`, we will create an `options` (`DeploymentOptions`) object and send it to `deployVerticle()` for configuring `MyFirstVerticle` class.


```java
private int port = 8081;

DeploymentOptions options = new DeploymentOptions()
    .setConfig(new JsonObject().put("HTTP_PORT", port));

vertx.deployVerticle(MyFirstVerticle.class.getName(), options, context.asyncAssertSuccess());
```

We also have to substitute in `vertx.createHttpClient().getNow()` method the hard-coded por number `8080` with `port` variable. With these modifications, the code of `MyFirstVerticleTest` will be alike the code below:

```java
package io.vertx.intro.first;

@RunWith(VertxUnitRunner.class)
public class MyFirstVerticleTest {

    private Vertx vertx;
    private int port = 8081;

    @Before
    public void setUp(TestContext context) {
        vertx = Vertx.vertx();

        // Create deployment options with the chosen port
        DeploymentOptions options = new DeploymentOptions()
            .setConfig(new JsonObject().put("HTTP_PORT", port));

        // Deploy the verticle with the deployment options
        vertx.deployVerticle(MyFirstVerticle.class.getName(), options, context.asyncAssertSuccess());
    }

    @After
    public void tearDown(TestContext context) {
        vertx.close(context.asyncAssertSuccess());
    }

    @Test
    public void testMyApplication(TestContext context) {
        final Async async = context.async();

        vertx.createHttpClient().getNow(port, "localhost", "/",
            response ->
                response.handler(body -> {
                    context.assertTrue(body.toString().contains("Hello"));
                    async.complete();
                }));
    }
}
```

We can test it now:

```text
$ mvn clean test
```

### Random port for tests

What happens when the port 8081 is also used? To avoid this, we will open a server socket that would pick a random port, retrieve it and close the socket. This port will be used when launching tests against `MyFirstVerticleTest`.

In `MyFirstVerticleTest` class, before creating `DeploymentOptions` object, we modify the value of `port` variable:


```java
ServerSocket socket = new ServerSocket(0);
port = socket.getLocalPort();
socket.close();

DeploymentOptions options = new DeploymentOptions()
    .setConfig(new JsonObject().put("HTTP_PORT", port));
```

The code of `MyFirstVerticleTest` will be alike the code below:

```java
package io.vertx.intro.first;

import io.vertx.core.DeploymentOptions;
import io.vertx.core.Vertx;
import io.vertx.core.json.JsonObject;
import io.vertx.ext.unit.Async;
import io.vertx.ext.unit.TestContext;
import io.vertx.ext.unit.junit.VertxUnitRunner;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;

import java.io.IOException;
import java.net.ServerSocket;

@RunWith(VertxUnitRunner.class)
public class MyFirstVerticleTest {

    private Vertx vertx;
    private int port = 8081;

    @Before
    public void setUp(TestContext context) throws IOException {
        vertx = Vertx.vertx();

        // Pick an available and random
        ServerSocket socket = new ServerSocket(0);
        port = socket.getLocalPort();
        socket.close();

        // Create deployment options with the chosen port
        DeploymentOptions options = new DeploymentOptions()
            .setConfig(new JsonObject().put("HTTP_PORT", port));

        // Deploy the verticle with the deployment options
        vertx.deployVerticle(MyFirstVerticle.class.getName(), options, context.asyncAssertSuccess());
    }

    @After
    public void tearDown(TestContext context) {
        vertx.close(context.asyncAssertSuccess());
    }

    @Test
    public void testMyApplication(TestContext context) {
        final Async async = context.async();

        vertx.createHttpClient().getNow(port, "localhost", "/",
            response ->
                response.handler(body -> {
                    context.assertTrue(body.toString().contains("Hello"));
                    async.complete();
                }));
    }
}
```

We can test it now:

```text
$ mvn clean test
```

##  External configuration

Random ports may have sense for testing but can't be used for production. Now we will create the JSON configuration file at `src/main/conf/my-application-conf.json` with the content below:

```json
{
  "HTTP_PORT" : 8082
}
```

We can build the fat jar and launch it specifying the embedded configuration file:

```text
$ mvn clean package
$ java -jar target/my-first-app-1.0-SNAPSHOT.jar -conf src/main/conf/my-application-conf.json
```

Access [localhost:8082](http://localhost:8082) to test the application.

The fat jar is using the `Launcher` class (provided by Vert.x) to launch the application. This class is reading the `-conf` parameter to create the `JsonObject` that will be accesed via `config()` method.

## Twelve-factor app methodology

Twelve-factor app recommends that the application read environment variables as the configuration. We will learn how to use `vertx-config` module to handle with this.

Add the following dependency to `pom.xml` file:

```xml
<dependency>
    <groupId>io.vertx</groupId>
    <artifactId>vertx-config</artifactId>
    <version>${vertx.version}</version>
</dependency>
```

An we will update `start` method to:

```java
package io.vertx.intro.first;

import io.vertx.config.ConfigRetriever;
import io.vertx.core.AbstractVerticle;
import io.vertx.core.Future;

public class MyFirstVerticle extends AbstractVerticle {

    @Override
    public void start(Future<Void> fut) {
        ConfigRetriever retriever = ConfigRetriever.create(vertx);
        retriever.getConfig(
            config -> {
                if (config.failed()) {
                    fut.fail(config.cause());
                } else {
                    vertx
                        .createHttpServer()
                        .requestHandler(r ->
                            r.response().end("<h1>Hello from my first Vert.x application</h1>"))
                                .listen(config.result().getInteger("HTTP_PORT", 8080), result -> {
                                    if (result.succeeded()) {
                                        fut.complete();
                                    } else {
                                        fut.fail(result.cause());
                                    }
                                });
                    }
                }
        );
    }
}
```

The `vertx-config` module provides the `ConfigRetriever`. This object is responsible for retrieving the different configuration chunks and computing the final configuration. Since this process is asynchronous, the result is passed to a handler that executes the rest of the startup logic.

With this in place, the port is now chosen from 3 different locations:

  - The configuration file, as seen previously (using -conf).
  - System properties. For instance, launch the application with -DHTTP_PORT=8081 to use the port 8081.
  - Environment properties. For instance, launch the application with:

```text
$ mvn clean package
$ export HTTP_PORT=8083
$ java -jar target/my-first-app-1.0-SNAPSHOT.jar
```

Access [localhost:8083](http://localhost:8083) to test the application.
