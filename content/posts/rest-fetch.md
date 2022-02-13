[//]: # (@formatter:off)
---
title: "My DRY implementation of GET REST endpoint"
date: 2022-01-05 
hideSummary: true
---
[//]: # (@formatter:on)

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
https://api.example.com/examples?filter=<some_condition>&include=<some_relations>
```

And let's briefly describe our entities:

```sql
CREATE TABLE person
(
    id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name           TEXT UNIQUE NOT NULL,
    fk_best_friend UUID REFERENCES person (id)
);
```

```sql
CREATE TABLE cat
(
    id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name      TEXT UNIQUE                 NOT NULL,
    dob       DATE                        NOT NULL,
    gender    TEXT                        NOT NULL,
    fk_owner  UUID REFERENCES person (id) NOT NULL,
    fk_father UUID REFERENCES cat (id),
    fk_mother UUID REFERENCES cat (id)
);
```

### Filtering

There is lots of different approaches how to achieve this goal but my personal choice is **RSQL** with spring-data
**Specification** and in particular this spring boot starter https://github.com/perplexhub/rsql-jpa-specification

In short library above allows converting string like **code=='demo';company.id>100** into spring-data **Specification**
which later we be evaluated and as a result we will get only entities which has field **code** as **demo** and theirs **
company** relation has **id** more than **100**

So in the end our controller will look like below:

```java
    //http://localhost:8080/cats?filter=owner.name==bob;name==scooter
    @GetMapping("/cats")
    public List<Cat> findAll(
        @RequestParam(value = "filter", required = false) String rsqlFilter
    ) {
        return repository.findAll(RSQLJPASupport.rsql(rsqlFilter));
    }
```

### Inclusion

Here is the most interesting part at least for me:) So we want a user define a scope of receiving data, and also we
don't want to write a lot of code, ideally as it is done in filtering part.

So first I looked at JSON:API implementations in Java to steal solution from there and the most ready for production was
project [elide](https://elide.io/), but I found that it has build in N+1 problem which is by design there and beside of
this it looks really not usual comparing to standard spring boot, you may find demo
app [here](https://github.com/spiashko/elide-demo), so I continued my research.

Obviously I started to look at GraphQL implementations trying to steal solution from them. While looking, I bumped
into [blaze persistence](https://persistence.blazebit.com/). I must say it is very nice lib which have solutions for
different data access problems. Actually thanks to this lib I found a solution for cursor pagination, but it is for
separate article. And even-though in the end this lib doesn't contain that magic class similar to RSQLJPASupport I found
this lib worth to mention.
[Here](https://github.com/spiashko/blaze-persistence-graphql-demo) is my demo project with Blaze Persistence GraphQL.

So in the end I decided to write my own solution which basically resulted in my simple `rest-fetch` project which solves
inclusion in a way described [here](https://github.com/spiashko/rest-fetch#rfetch) with all details.

### Collect all together

Long story short now we help of `rsql` and `rfetch` we can define our GET controller in super short way and give client
as much flexibility as possible and at the same time control performance.

If we want only one SQL:

```java
    @JsonView(View.Retrieve.class)
    @GetMapping("/cats")
    public List<Cat> findAll(
            @RequestParam(value = "filter", required = false) String rsqlFilter,
            @RequestParam(value = "include", required = false) String rfetchInclude
    ) {
        beforeRequestActionsExecutor.execute(rsqlFilter, rfetchInclude, Cat.class, View.Retrieve.class);

        Specification<Cat> fetchSpec = FetchAllInOneSpecTemplate.INSTANCE.toSpecification(RfetchSupport.compile(rfetchInclude, Cat.class));

        return repository.findAll(Specification.where(fetchSpec).and(RSQLJPASupport.rsql(rsqlFilter)));
    }
```

If we want to avoid cartesian product problem:

```java
    @JsonView(View.Retrieve.class)
    @GetMapping
    public List<Person> findAll(
            @RequestParam(value = "filter", required = false) String rsqlFilter,
            @RequestParam(value = "include", required = false) String rfetchInclude
    ) {
        beforeRequestActionsExecutor.execute(rsqlFilter, rfetchInclude, Person.class, View.Retrieve.class);

        return transactionTemplate.execute(s -> {
            List<Person> result = repository.findAll(RSQLJPASupport.rsql(rsqlFilter));
            fetchSmartTemplate.enrichList(RfetchSupport.compile(rfetchInclude, Person.class), result);
            return result;
        });
    }
```

Project link: [github](https://github.com/spiashko/rest-fetch)

### TODO

Add JSON:API integration

