---
title: "My DRY implementation of GET rest-point"
date: 2022-01-05 draft: true
---

### Introduction

For long time I was interested in finding the most simple, small, flexible and the same time efficient implementation of
GET rest-points. So in this article I will try to share my findings and hope you discover something interesting for you.
And to make it easier to explain and understand let's design and implement simple application with a couple of entities.

### Design application

Assume we need to develop simple application to serve data about **people** and theirs **cats**. And this application
should have features:

| Features   | Request params    |
|------------|-------------------|
| filtering  | filter            |
| pagination | limit;offset;sort |
| inclusion  | include           |

**note:** by inclusion means to allow requester define scope of receiving data (like GraphQL)

Considering requirements above our application should be able to handle such GET requests as below

```
https://api.example.com/examples?filter=<some_condition>limit=<x>&offset=<x>&sort=<sort_condition>&include=<some_fields>
```

And let's briefly describe our entities:

rest-persistence.png

### Filtering

There is lots of different approaches how to achieve this goal but my personal choice is `RSQL` with spring-data
`Specification` and in particular this spring boot starter https://github.com/perplexhub/rsql-jpa-specification

In short library above allows converting string like `code=='demo';company.id>100` into spring-data `Specification`
which later we be evaluated and as a result we will get only entities which has field `code` as `demo` and
theirs `company` relation has `id` more than `100`

To make it even more convenient we can create custom annotation which will be used in next way:

```
    #http://localhost:8080/cats?filter=owner.name==bob;name==scooter
    #expression inside filter param mapped into rsqlSpec argument
    @GetMapping("/cats")
    public List<Cat> findAll(
            @RsqlSpec Specification<Cat> rsqlSpec
    ) {
        List<Cat> result = searchService.findAll(rsqlSpec);
        return result;
    }
```

### Pagination

Here everything is super easy as spring data has

### Collect all together

### Conclusion
