# [Post 3] Some Rest with Vert.x

Based on [Some Rest with Vert.x](https://developers.redhat.com/blog/2018/03/22/eclipse-vert-x-application-configuration/) written by [Clement Escoffier](https://developers.redhat.com/blog/author/cescoffier/).

All the exercises are supossing that their root directory is `post-3`.


## Vert.x Web

Dealing with complex HTTP applications using only `Vert.x Core` becomes difficult, that's the main reason behind `Vert.x Web`. It eases a lot web application development without changing the philosophy.

In `post-1`, in `MyFirstVerticle` class we handled the HTTP response writing a handler the fly:

```java
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
```

But here in `post-2`, we will create an instance of `io.vertx.ext.web.Router` instance called `router`. It is responsible for dispatching the HTTP requests to the right handler based on the route of the URL:


```java
Router router = Router.router(vertx);

router.route("/").handler(rc -> {
    HttpServerResponse response = rc.response();
    response
        .putHeader("content-type", "text/html")
        .end("<h1>Hello from my first Vert.x 3 app</h1>");
});
```

So we can now use as handler the method `accept` of the object `router`. It instructs Vert.x to call the accept method of the router when it receives a request.

```java
vertx
    .createHttpServer()
    .requestHandler(router::accept)
    .listen(
        // Retrieve the port from the
        // configuration, default to 8080.
        config().getInteger("HTTP_PORT", 8080),
            result -> {
                if (result.succeeded()) {
                    fut.complete();
                } else {
                    fut.fail(result.cause());
                }
            }
        );
```

## Implementing Vert.x Web

Add the dependency below to `pom.xml` file:

```xml
<dependency>
    <groupId>io.vertx</groupId>
    <artifactId>vertx-web</artifactId>
    <version>${vertx.version}</version>
</dependency>
```

Modify `MyFirstVerticle` class to be alike the code below:

```java
package io.vertx.intro.first;

public class MyFirstVerticle extends AbstractVerticle {

    @Override
    public void start(Future fut) {
        // Create a router object.
        Router router = Router.router(vertx);

        // Bind "/" to our hello message - so we are
        // still compatible with out tests.
        router.route("/").handler(rc -> {
            HttpServerResponse response = rc.response();
            response
                    .putHeader("content-type", "text/html")
                    .end("<h1>Hello from my first Vert.x 3 app</h1>");
        });

        ConfigRetriever retriever = ConfigRetriever.create(vertx);
        retriever.getConfig(
                config -> {
                    if (config.failed()) {
                        fut.fail(config.cause());
                    } else {
                        // Create the HTTP server and pass the
                        // "accept" method to the request handler.
                        vertx
                                .createHttpServer()
                                .requestHandler(router::accept)
                                .listen(
                                        // Retrieve the port from the
                                        // configuration, default to 8080.
                                        config().getInteger("HTTP_PORT", 8080),
                                        result -> {
                                            if (result.succeeded()) {
                                                fut.complete();
                                            } else {
                                                fut.fail(result.cause());
                                            }
                                        }
                                );
                    }
                }
        );
    }
}
```

Start the application:

```text
$ mvn compile vertx:run
```

Access [localhost:8082](http://localhost:8082) to test the application.


## Exposing Static Resources

We will serve static resources, such as an `index.html` page.
 
Create directory for static resources:

```text
$ mkdir src/main/resources/assets
```

Create an `index.html` page in `src/main/resources/assets` with the content from [here](https://github.com/josebamartos/introduction-to-eclipse-vertx/blob/master/post-3/src/main/resources/assets/index.html).

Basically, the page is a simple CRUD UI to manage my not-yet-read articles. It was made in a generic way, so you can transpose it to your own stuff. The reading list is displayed in the main table. You can create a new reading list, edit one, or delete one. These actions are relying on a REST API (that we are going to implement) through AJAX calls. That’s all.

Add a new route definition to `MyFirstVerticle` class:

```java
router.route("/assets/*")
    .handler(StaticHandler.create("assets"));
```

`MyFirstVerticle` class should be alike the code below:

```java
package io.vertx.intro.first;

import io.vertx.config.ConfigRetriever;
import io.vertx.core.AbstractVerticle;
import io.vertx.core.Future;
import io.vertx.core.http.HttpServerResponse;
import io.vertx.ext.web.Router;
import io.vertx.ext.web.handler.StaticHandler;

public class MyFirstVerticle extends AbstractVerticle {

    @Override
    public void start(Future fut) {
        // Create a router object.
        Router router = Router.router(vertx);

        // Bind "/" to our hello message - so we are
        // still compatible with out tests.
        router.route("/").handler(rc -> {
            HttpServerResponse response = rc.response();
            response
                    .putHeader("content-type", "text/html")
                    .end("<h1>Hello from my first Vert.x 3 app</h1>");
        });
        // Serve static resources from the /assets directory
        router.route("/assets/*")
                .handler(StaticHandler.create("assets"));

        ConfigRetriever retriever = ConfigRetriever.create(vertx);
        retriever.getConfig(
                config -> {
                    if (config.failed()) {
                        fut.fail(config.cause());
                    } else {
                        // Create the HTTP server and pass the
                        // "accept" method to the request handler.
                        vertx
                                .createHttpServer()
                                .requestHandler(router::accept)
                                .listen(
                                        // Retrieve the port from the
                                        // configuration, default to 8080.
                                        config().getInteger("HTTP_PORT", 8080),
                                        result -> {
                                            if (result.succeeded()) {
                                                fut.complete();
                                            } else {
                                                fut.fail(result.cause());
                                            }
                                        }
                                );
                    }
                }
        );
    }
}
```


Start the application:

```text
$ mvn compile vertx:run
```

Access [localhost:8082/assets/index.html](http://localhost:8082/assets/index.html) to test the application.


## REST API with Vert.x Web

Vert.x Web makes the implementation of a REST API really easy, as it basically routes your URL to the right handler. The API is very simple, and will be structured as detailed in the table below:

| Method | Path              | Description                                        |
| ------ | ----------------- | -------------------------------------------------- |
| GET    | /api/articles     | Get all articles (getAll)                          |
| GET    | /api/articles/:id | Get the article with the corresponding id (getOne) |
| POST   | /api/articles     | Add a new article (addOne)                         |
| PUT    | /api/articles/:id | Update an article (updateOne)                      |
| DELETE | /api/articles/id  | Delete an article (deleteOne)                      |

This REST API manages articles, so we will create a very simple bean class representing an Article. This way we can easily perform marshal/unmarshaling operations for JSON using `Jackson` library, used by Vert.x.

Let's create the class in `src/main/java/io/vertx/intro/first/Article.java`:

```java
package io.vertx.intro.first;

import java.util.concurrent.atomic.AtomicInteger;

public class Article {

    private static final AtomicInteger COUNTER = new AtomicInteger();

    private final int id;

    private String title;

    private String url;

    public Article(String title, String url) {
        this.id = COUNTER.getAndIncrement();
        this.title = title;
        this.url = url;
    }

    public Article() {
        this.id = COUNTER.getAndIncrement();
    }

    public int getId() {
        return id;
    }

    public String getTitle() {
        return title;
    }

    public Article setTitle(String title) {
        this.title = title;
        return this;
    }

    public String getUrl() {
        return url;
    }

    public Article setUrl(String url) {
        this.url = url;
        return this;
    }
}
```

As we don’t really have a backend here, we're using just an in-memory map, so we must create some dummy data to work with. Create `readingList` (`LinkedHashMap`) object to store articles and define the `createSomeData()` method after `start()` method.

```java
// Store our readingList
private Map<Integer, Article> readingList = new LinkedHashMap<>();
// Create a readingList
private void createSomeData() {
    Article article1 = new Article(
        "Fallacies of distributed computing", 
        "https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing");
    readingList.put(article1.getId(), article1);
    Article article2 = new Article(
        "Reactive Manifesto", 
        "https://www.reactivemanifesto.org/");
    readingList.put(article2.getId(), article2);
}
```

Then, in the start method, call the createSomeData method:

```java
@Override
public void start(Future fut) {
    createSomeData();

    // Create a router object.
    Router router = Router.router(vertx);

    // Rest of the method
```

### List of articles (GET /api/articles)

Returns the list of acticles structured in a JSON Array.

In the start method, add this line just below the static handler line:

```java
router.get("/api/articles").handler(this::getAll);
```

This line instructs the router to handle the GET requests on `/api/articles` by calling the `getAll()` method. We could have inlined the handler code, but for clarity reasons let’s create another method:

```java
private void getAll(RoutingContext rc) {
    rc.response()
        .putHeader("content-type", "application/json; charset=utf-8")
        .end(Json.encodePrettily(readingList.values()));
}
```
Start the application:

```text
$ mvn compile vertx:run
```

Access [localhost:8082/api/articles](http://localhost:8082/api/articles) to get the response from `GET api/articles` service:

```text
$ curl -i http://localhost:8082/api/articles
HTTP/1.1 200 OK
content-type: application/json; charset=utf-8
Content-Length: 244

[ {
  "id" : 0,
  "title" : "Fallacies of distributed computing",
  "url" : "https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing"
}, {
  "id" : 1,
  "title" : "Reactive Manifesto",
  "url" : "https://www.reactivemanifesto.org/"
} ]
```

Access [localhost:8082/assets/index.html](http://localhost:8082/assets/index.html) to test the application. In this moment the two articles created through `createSomeData()` method would appear in the webpage.


### Add new article

Now we can retrieve the set of articles, let’s create a new one. Unlike the previous REST API endpoint, this one needs to read the request’s body. For performance reasons, it should be explicitly enabled. Don’t be scared; it’s just a handler.

In the start method, add these lines just below the line ending by getAll:

```java
router.route("/api/articles*").handler(BodyHandler.create());
router.post("/api/articles").handler(this::addOne);
```

  - The first line enables the reading of the request body for all routes under `/api/articles`. We could have enabled it globally with `router.route().handler(BodyHandler.create())`.
  - The second line maps `POST` requests on `/api/articles` to the addOne method.
  
Let’s create this method:

```test
private void addOne(RoutingContext rc) {
    Article article = rc.getBodyAsJson().mapTo(Article.class);
    readingList.put(article.getId(), article);
    rc.response()
        .setStatusCode(201)
        .putHeader("content-type", 
          "application/json; charset=utf-8")
        .end(Json.encodePrettily(article));
}
```

Start the application:

```text
$ mvn compile vertx:run
```

Access [localhost:8082/assets/index.html](http://localhost:8082/assets/index.html) to test the application. Enter the data such as: `Building Reactive Microservices in Java` as title and `https://developers.redhat.com/promotions/building-reactive-microservices-in-java/` as url. Click on  `Save`, and the article should appear in the list. When clicking `Save`, the method starts by creating an `Article` instance from the request body. Once created it adds it to the backend and returns the created article as `JSON`.

**Status 201?**

As you can see, we have set the response status to `201`. It means `CREATED` and is generally used in a REST API that creates an entity. By default `Vert.x Web` is setting the status to `200`, meaning `OK`.

### Delete article

Well, sometimes you take time to read articles, so we should be able to delete it from the list. In the start method, add this line:

```java
router.delete("/api/articles/:id").handler(this::deleteOne);
```

In the path, we define a parameter :id. So, when handling a matching request, Vert.x extracts the path segment corresponding to the parameter and lets us access it in the handler method. For instance, /api/articles/0 maps id to 0.

Let’s see how the parameter can be used in the handler method. Create the deleteOne method as follows:

```java
private void deleteOne(RoutingContext rc) {
    String id = rc.request().getParam("id");
    try {
        Integer idAsInteger = Integer.valueOf(id);
        readingList.remove(idAsInteger);
        rc.response().setStatusCode(204).end();
    } catch (NumberFormatException e) {
        rc.response().setStatusCode(400).end();
    }
}
```

The `path` parameter is retrieved using `rc.request().getParam("id")`. In a `try-catch` block we try to convert this path parameter as integer. If it fails (with a `NumberFormatException`), we write a `Bad Request - 400 response`. If it’s ok, we just remove the article from our backend.

**Status 204?**

As you can see, we have set the response status to `204 - NO CONTENT`. Responses to the HTTP Verb `DELETE` have generally no content.

### The other methods

We won’t explain `getOne` and `updateOne` as the implementations are straightforward and very similar. Their implementations are available on GitHub.

To finish the implementation, modify `MyFirstVerticle` class to be alike the code below:

```java
package io.vertx.intro.first;

public class MyFirstVerticle extends AbstractVerticle {

    // Store our readingList
    private Map<Integer, Article> readingList = new LinkedHashMap<>();

    @Override
    public void start(Future fut) {
        createSomeData();

        // Create a router object.
        Router router = Router.router(vertx);

        // Bind "/" to our hello message - so we are
        // still compatible with out tests.
        router.route("/").handler(rc -> {
            HttpServerResponse response = rc.response();
            response
                    .putHeader("content-type", "text/html")
                    .end("<h1>Hello from my first Vert.x 3 app</h1>");
        });
        // Serve static resources from the /assets directory
        router.route("/assets/*").handler(StaticHandler.create("assets"));
        router.get("/api/articles").handler(this::getAll);
        router.get("/api/articles/:id").handler(this::getOne);
        router.route("/api/articles*").handler(BodyHandler.create());
        router.post("/api/articles").handler(this::addOne);
        router.delete("/api/articles/:id").handler(this::deleteOne);
        router.put("/api/articles/:id").handler(this::updateOne);

        ConfigRetriever retriever = ConfigRetriever.create(vertx);
        retriever.getConfig(
                config -> {
                    if (config.failed()) {
                        fut.fail(config.cause());
                    } else {
                        // Create the HTTP server and pass the
                        // "accept" method to the request handler.
                        vertx
                                .createHttpServer()
                                .requestHandler(router::accept)
                                .listen(
                                        // Retrieve the port from the
                                        // configuration, default to 8080.
                                        config().getInteger("HTTP_PORT", 8080),
                                        result -> {
                                            if (result.succeeded()) {
                                                fut.complete();
                                            } else {
                                                fut.fail(result.cause());
                                            }
                                        }
                                );
                    }
                }
        );
    }

    private void createSomeData() {
        Article article1 = new Article(
            "Fallacies of distributed computing",
            "https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing");
        readingList.put(article1.getId(), article1);
        Article article2 = new Article(
            "Reactive Manifesto",
            "https://www.reactivemanifesto.org/");
        readingList.put(article2.getId(), article2);
    }

    private void getAll(RoutingContext rc) {
        rc.response()
            .putHeader("content-type", "application/json; charset=utf-8")
            .end(Json.encodePrettily(readingList.values()));
    }

    private void addOne(RoutingContext rc) {
        Article article = rc.getBodyAsJson().mapTo(Article.class);
        readingList.put(article.getId(), article);
        rc.response()
                .setStatusCode(201)
                .putHeader("content-type", "application/json; charset=utf-8")
                .end(Json.encodePrettily(article));
    }

    private void deleteOne(RoutingContext routingContext) {
        String id = routingContext.request().getParam("id");
        try {
            Integer idAsInteger = Integer.valueOf(id);
            readingList.remove(idAsInteger);
            routingContext.response().setStatusCode(204).end();
        } catch (NumberFormatException e) {
            routingContext.response().setStatusCode(400).end();
        }
    }


    private void getOne(RoutingContext routingContext) {
        String id = routingContext.request().getParam("id");
        try {
            Integer idAsInteger = Integer.valueOf(id);
            Article article = readingList.get(idAsInteger);
            if (article == null) {
                // Not found
                routingContext.response().setStatusCode(404).end();
            } else {
                routingContext.response()
                        .setStatusCode(200)
                        .putHeader("content-type", "application/json; charset=utf-8")
                        .end(Json.encodePrettily(article));
            }
        } catch (NumberFormatException e) {
            routingContext.response().setStatusCode(400).end();
        }
    }

    private void updateOne(RoutingContext routingContext) {
        String id = routingContext.request().getParam("id");
        try {
            Integer idAsInteger = Integer.valueOf(id);
            Article article = readingList.get(idAsInteger);
            if (article == null) {
                // Not found
                routingContext.response().setStatusCode(404).end();
            } else {
                JsonObject body = routingContext.getBodyAsJson();
                article.setTitle(body.getString("title")).setUrl(body.getString("url"));
                readingList.put(idAsInteger, article);
                routingContext.response()
                        .setStatusCode(200)
                        .putHeader("content-type", "application/json; charset=utf-8")
                        .end(Json.encodePrettily(article));
            }
        } catch (NumberFormatException e) {
            routingContext.response().setStatusCode(400).end();
        }

    }
}
```

Start the application:

```text
$ mvn compile vertx:run
```

Access [localhost:8082/assets/index.html](http://localhost:8082/assets/index.html) and test all the operations.

## Concurrency

Let’s talk a bit about concurrency. Obviously using an in-memory backend is not for a production setting, but it illustrates one of the key characteristics of `Vert.x`. We do read and write operations on this backend without using any synchronization constructs. Seasoned Java developers would clearly be mad about this.

However, `Vert.x` verticles are single threaded. It means that only one thread is accessing them, and always the same thread. So we don’t need synchronization because we can’t have concurrent access. That’s great, isn’t it? But how do we handle concurrent HTTP requests? Well, that’s also simple, using the very same thread every time. Everything we do is not blocking processing and responses to the request are fast. So, while we won’t process another request at the same time, it does not mean we can’t handle concurrent requests. They are just queued; but not for long. If you try to execute concurrent requests (with tools like `Gatling` or `wrk`) you will realize very good response times, thanks to this event loop mechanism.