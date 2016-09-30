---
layout: blog
img: blogs/springboot_jdbi.png
category: Blog
title: Spring Boot + JDBI
tags: 
  - spring-boot
  - jdbi
  - micro services
description: |
---

I recently found myself architecting yet another microservices REST backend. Within the past few years I've written from-scratch backends in [Spring](https://projects.spring.io/spring-framework/), [Dropwizard](http://www.dropwizard.io/) and even in Python using [web.py](http://webpy.org/) but each has left me wanting more. 

## Spring

The grandaddy of modern microservices backends spring has great dynamic injection (DI) and a large extensive library of addons, but it feels boated and enterprisey. It's the workhorse for many successful rest backends but has a long startup time and just feels somewhat cumbersome to work in when compared to some of the newer frameworks.

It also features out of the box support for Hibernate as an ORM layer which (feel free to flame me in the comments) in my experience gets in the way as often as it doesn't. It's great for simple relational structures and is quick to bootstrap solutions but as soon as you're dealing with complex data structures or want specific types of queries (say for efficiency) it becomes quite cumbersome. 

## Dropwizard

A light-weight Jersey container which provides great support for [JDBI](http://jdbi.org/) as an ORM-light solution. I've found JDBI to be a simple way to interact with a DB, yes you have to bind your ResultSets to your model objects manually but I've found it a small price to pay for the great flexibility it provides. In my opinion it's a really nice solution that sits somewhere between JDBC and a full-blown ORM. 

What dropwizard doesn't do well is DI. There is support for Guice via extensions like [dropwizard-guice](https://github.com/HubSpot/dropwizard-guice) but I have found these to be clunky at best. My biggest issue with all of these solutions is that you end up with deperate DI graphs for each module. This isn't a problem for simple applications but I've experienced in complex applications that dropwizard-guice becomes quite unweildy and the seperate DI graphs end up having the same class being injected with a different instance/state which can start producing unintended results. 

I really just want out of the box DI that is easy to use and gets out of the way. This should be plumbing, not an application feature.

## web.py

Well, clearly one of these things is not like the other. I included web.py because I have written a fully-featured rest tier in it and if your religion is python then it's a good choice. If you're actually contemplating a Java-based framework then this obviously isn't your cup of tea.

## Spring Boot

Spring Boot is a lightweight version of Spring that is focussed on creation Spring-like applications that "just run". Whereas Spring can feel quite enterprisey, Spring Boot both feels and performs like a true first-class microservices framework. It has the same great support for DI as Spring and also features an extensive library of `spring-boot-starter-*` maven dependencies that you can use to add support for a great many things.

It still, however, uses hibernate as it's "native" ORM tier. Now if that's fine for you then go for it, Spring Boot out of the box is a great solution but for me I just find it too cumbersome. So what's a savvy software engineer to do?

# Spring Boot + JDBI

I wanted the best of both Spring Boot DI as well as a nice simple and consice ORM layer using JDBI so I created [spring-boot-jdbi-seed](https://github.com/dhagge/spring-boot-jdbi-seed). This project on github features a full rest tier (note: no security, that's for another blog post) as well as integration tests, checkstyle, findbugs, etc. But I want to focus on specifically what I had to do to get Spring Boot and JDBI integrated.

## build.gradle

Firstly you have to get rid of those hibernate libs

```gradle
configurations {
    all*.exclude group: "org.hibernate", module: "hibernate-entitymanager"
    all*.exclude group: "org.apache.tomcat", module: "tomcat-jdbc"
}
```

And then let's add the dependencies we need
```gradle
dependencies {
    compile (
            ...
            "org.springframework.boot:spring-boot-starter-jdbc",
            "joda-time:joda-time:2.8.1",
            "org.springframework.boot:spring-boot-starter-data-jpa",
            "org.jdbi:jdbi:2.71",
            ...
    )

    runtime(
            "mysql:mysql-connector-java:5.1.38",
            "com.zaxxer:HikariCP:2.4.3"
    )
}
```

We're adding joda-time to allow for DateTime objects to be serialized to/from the DB, because everyone uses Joda don't they?

## application.properties

Next I added the data access properties configuration to `application.properties`. Spring Boot nicely allows us to use the generic `spring.datasource.*` properties rather than any jdbi-specific properties.

```
spring.datasource.url=jdbc:mysql://localhost/services?createDatabaseIfNotExist=true&useSSL=false
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

Note that I used MySQL as the DB but that can be swapped out with whichever driver you want ot use of course.

## ServiceApplication

Next, set the jvm timezone default to UTC so that all `DateTime` objects will be set to UTC. This helps greatly serializing to/from the DB and not accidentally getting timezone conversions being applied when they shouldn't be. Secondly, we're registering JodaModule so that we can actually serialize the instances.

```java
@PostConstruct
public void postConstruct() {
    // set the JVM timezone to UTC
    TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
}

@Bean
public Module jodaModule() {
    return new JodaModule();
}
```

## PersistenceConfiguration

Now we have to create the Spring Boot configuration that will actually expose JDBI as well as map date/time objects. The full code can be found in [PersistenceConfiguration.java](https://github.com/dhagge/spring-boot-jdbi-seed/blob/master/src/main/java/com/myco/PersistenceConfiguration.java) but I'll draw out the key code here:

```java
@Configuration
public class PersistenceConfiguration {

    @Autowired
    DataSource dataSource;

    @Bean
    public DBI dbiBean() {
        DBI dbi = new DBI(dataSource);
        dbi.registerArgumentFactory(new DateTimeArgumentFactory());
        dbi.registerArgumentFactory(new LocalDateArgumentFactory());
        dbi.registerColumnMapper(new JodaDateTimeMapper());
        return dbi;
    }

    ...

    /**
     * DBI argument factory for converting joda DateTime to sql timestamp
     */
    public static class DateTimeArgumentFactory implements ArgumentFactory<DateTime> {

        @Override
        public boolean accepts(Class<?> expectedType, Object value, StatementContext ctx) {
            return value != null && DateTime.class.isAssignableFrom(value.getClass());
        }

        @Override
        public Argument build(Class<?> expectedType, final DateTime value, StatementContext ctx) {
            return new Argument() {
                @Override
                public void apply(int position, PreparedStatement statement, StatementContext ctx) throws SQLException {
                    long millis = value.withZone(DateTimeZone.UTC).getMillis();
                    statement.setTimestamp(position, new Timestamp(millis), getUtcCalendar());
                }
            };
        }
    }

    /**
     * A {@link ResultColumnMapper} to map Joda {@link DateTime} objects.
     */
    public static class JodaDateTimeMapper implements ResultColumnMapper<DateTime> {

        @Override
        public DateTime mapColumn(ResultSet rs, int columnNumber, StatementContext ctx) throws SQLException {
            final Timestamp timestamp = rs.getTimestamp(columnNumber, getUtcCalendar());
            if (timestamp == null) {
                return null;
            }
            return new DateTime(timestamp.getTime());
        }

        @Override
        public DateTime mapColumn(ResultSet rs, String columnLabel, StatementContext ctx) throws SQLException {
            final Timestamp timestamp = rs.getTimestamp(columnLabel, getUtcCalendar());
            if (timestamp == null) {
                return null;
            }
            return new DateTime(timestamp.getTime());
        }
    }

```

## Autowire and use JDBI

That's it, now you can autowire and use a `DBI` object to do everything that [JDBI](http://jdbi.org) allows you to do

```java
@RestController
@RequestMapping("/test")
@Slf4j
public class TestResource {

    @Autowired
    DBI dbi;

    @RequestMapping(method = RequestMethod.GET)
    public String get() {
        String name;
        Handle handle = dbi.open();
        try {
            name = handle.createQuery("select description from my_test")
                    .map(StringMapper.FIRST)
                    .first();
        } finally {
            handle.close();
        }
        return name;
    }
}
```

I personally tend to use the JDBI [SQL Object API](http://jdbi.org/sql_object_overview/) which I find a simple and convinient way to query data but you can read the docs and use what's right for you.

Full source code can be found at [spring-boot-jdbi-seed](https://github.com/dhagge/spring-boot-jdbi-seed), please fork and use at your leisure!