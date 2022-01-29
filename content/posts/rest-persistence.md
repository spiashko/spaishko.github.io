---
title: "My DRY implementation of GET REST endpoint"
date: 2022-01-05
hideSummary: true
---

### Introduction

For long time I was interested in finding the most simple, small, flexible and the same time efficient implementation of
GET rest-points. So in this article I will try to share my findings and hope you discover something interesting for you.
And to make it easier to explain and understand let's design and implement simple application with a couple of entities.

### Design application

Assume we need to develop simple application to serve data about **people** and theirs **cats**. And this application
should have basic features as filtering and inclusion.

**note:** by inclusion means to allow requester define scope of receiving data (like GraphQL or in REST
world [JSON:API](https://jsonapi.org/))

Considering requirements above our application should be able to handle such GET requests as below

```
https://api.example.com/examples?filter=<some_condition>&include=<some_fields>
```

And let's briefly describe our entities:

![ERD](https://spiashko.github.io/rest-persistence.png)

### Filtering

There is lots of different approaches how to achieve this goal but my personal choice is **RSQL** with spring-data
**Specification** and in particular this spring boot starter https://github.com/perplexhub/rsql-jpa-specification

In short library above allows converting string like **code=='demo';company.id>100** into spring-data **Specification**
which later we be evaluated and as a result we will get only entities which has field **code** as **demo** and theirs **
company** relation has **id** more than **100**

To make it even more convenient we can create custom annotation which will be used in next way:

[//]: # (@formatter:off)
```java
    //http://localhost:8080/cats?filter=owner.name==bob;name==scooter
    //expression inside filter param mapped into rsqlSpec argument
    @GetMapping("/cats")
    public List<Cat> findAll(
            @RsqlSpec Specification<Cat> rsqlSpec
    ) {
        List<Cat> result = searchService.findAll(rsqlSpec);
        return result;
    }
```
[//]: # (@formatter:on)

### Inclusion

Here is the most interesting part at least for me:) So we want a user define a scope of receiving data, and also we
don't want to write a lot of code, ideally just one annotation as it is done in filtering part.

So first I looked at JSON:API implementations in Java to steal solution from there and the most ready for production was
project [elide](https://elide.io/), but I found that it has build in N+1 problem which is by design there and beside of
this it looks really not usual comparing to standard spring boot, you may find demo
app [here](https://github.com/spiashko/elide-demo), so I continued my research.

Obviously I started to look at GraphQL implementations trying to steal solution from them. While looking, I bumped
into [blaze persistence](https://persistence.blazebit.com/). I must say it is very nice lib which have solutions for
different data access problems. Actually thanks to this lib I found a solution for cursor pagination, but it is for
separate article. And even-though in the end this lib doesn't contain that magic annotation I found it worth to mention.
[Here](https://github.com/spiashko/blaze-persistence-graphql-demo) is my demo project with Blaze Persistence GraphQL.

After some time I
found [this article about GraphQL](https://piotrminkowski.com/2020/07/31/an-advanced-guide-to-graphql-with-spring-boot/)
from Piotr Minkowski and the most interesting part is this code snippet:

[//]: # (@formatter:off)
```java
private Specification<Department> fetchEmployees() {
   return (Specification<Department>) (root, query, builder) -> {
      Fetch<Department, Employee> f = root.fetch("employees", JoinType.LEFT);
      Join<Department, Employee> join = (Join<Department, Employee>) f;
      return join.getOn();
   };
}
```
[//]: # (@formatter:on)

So the key point is the fact that we can make fetch join right inside Specification. And knowing that fact writing
custom solution is just piece of cake and here it is:

[//]: # (@formatter:off)
```java
public class RfetchSupport {

    public Specification<Object> toSpecification(List<String> includedPaths) {
        Specification<Object> rfetchSpec = includedPaths.stream()
                .map(this::buildSpec)
                .reduce(Specification.where(null),
                        Specification::and);
        return rfetchSpec;
    }

    private Specification<Object> buildSpec(String attributePath) {
        return (root, query, builder) -> {
            PropertyPath path = PropertyPath.from(attributePath, root.getJavaType());
            FetchParent<Object, Object> f = traversePath(root, path);
            Join<Object, Object> join = (Join<Object, Object>) f;

            query.distinct(true);

            return join.getOn();
        };
    }

    private FetchParent<Object, Object> traversePath(FetchParent<?, ?> root, PropertyPath path) {
        FetchParent<Object, Object> result = root.fetch(path.getSegment(), JoinType.LEFT);
        return path.hasNext() ? traversePath(result, Objects.requireNonNull(path.next())) : result;
    }

}
```
[//]: # (@formatter:on)

And obviously we can create custom annotation to make our controller looks like below

[//]: # (@formatter:off)
```java
    //http://localhost:8080/cats?include=father;mother
    @GetMapping("/cats")
    public List<Cat> findAll(
            @RfetchSpec Specification<Cat> rFetchSpec
    ) {
        List<Cat> result = searchService.findAll(rFetchSpec);
        return result;
    }
```
[//]: # (@formatter:on)

### Collect all together

Long story short now we can define our GET controller in super short way and give client as much flexibility as
possible.

[//]: # (@formatter:off)
```java
    //http://localhost:8080//cats?filter=owner.name==bob&include=father;mother
    @GetMapping("/cats")
    public List<Cat> findAll(
            @RfetchSpec Specification<Cat> rFetchSpec,
            @RsqlSpec Specification<Cat> rSqlSpec
    ) {
        val spec = Stream.of(rFetchSpec, rSqlSpec)
                .reduce(Specification.where(null),
                        Specification::and);
        List<Cat> result = searchService.findAll(spec);
        return result;
    }
```
[//]: # (@formatter:on)

You may find all code in my [github](https://github.com/spiashko/rest-persistence)

### TODO

Create annotation for cursor pagination

