# Introduction to Eclipse Vert.x

> I am using this repository as a personal workspace to complete the training proposed by [Clement Escoffier](https://developers.redhat.com/blog/author/cescoffier/) in the _Introduction to Eclipse Vert.x_ blog post series. I didn't fork the [original repository](https://github.com/redhat-developer/introduction-to-eclipse-vertx) because i want to develop the complete project from scratch.



This repository contains the code developed in the _Introduction to Eclipse Vert.x_ blog post series. Posts composing the series are:

1. [My First Vert.x Application](https://developers.redhat.com/blog/2018/03/13/eclipse-vertx-first-application/)
2. [Eclipse Vert.x Application Configuration](https://developers.redhat.com/blog/2018/03/22/eclipse-vert-x-application-configuration/)
3. [Some REST with Vert.x](https://developers.redhat.com/blog/2018/03/29/rest-vert-x/)
4. [Asynchronous data access with Vert.x](https://developers.redhat.com/blog/2018/04/09/accessing-data-reactive-way/)
5. Using Reactive eXtensions with Vert.x - to be published
6. Deploying Vert.x applications on Kubernetes and OpenShift - to be published

The code developed for each post is contained in the corresponding folder. Except mentionned otherwise in the blog post, the application can be:

* build using: `mvn package`
* run locally using `mvn compile vertx:run`