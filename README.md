# Logbook

[![Build Status](https://img.shields.io/travis/zalando/logbook.svg)](https://travis-ci.org/zalando/logbook)
[![Coverage Status](https://img.shields.io/coveralls/zalando/logbook.svg)](https://coveralls.io/r/zalando/logbook)
[![Release](https://img.shields.io/github/release/zalando/logbook.svg)](https://github.com/zalando/logbook/releases)
[![Maven Central](https://img.shields.io/maven-central/v/org.zalando/logbook.svg)](https://maven-badges.herokuapp.com/maven-central/org.zalando/logbook)

*Logbook* is an extensible library to enable request and response logging for different client- and server-side technologies. It comes with a core module `logbook-core` and specific modules per framework, e.g. [`logbook-servlet`](#servlet) for Servlet 3.0 environment and [`logbook-spring`](#spring) for applications using Spring's `RestTemplate`.

## Dependency

```xml
<dependency>
    <groupId>org.zalando</groupId>
    <artifactId>logbook</artifactId>
    <version>${logbook.version}</version>
</dependency>
```
## Usage

All integrations require an instance of `Logbook` which holds all configuration and wires all necessary parts together. You can either create one using all the defaults:

```java
Logbook logbook = Logbook.create()
```

Or create a customized version using the `LogbookBuilder`:

```java
Logbook logbook = Logbook.builder()
    .formatter(new CustomHttpLogFormatter())
    .writer(new CustomHttpLogWriter())
    .build();
```

Logbook works in three phases:

1. [Obfuscation](#obfuscation),
2. [Formatting](#formatting) and
3. [Writing](#writing)

Each phase is represented by one or more interfaces that can be used for customization and every phase has a sensible default:

## Obfuscation

The goal of *Obfuscation* is to prevent certain sensitive parts of HTTP requests and responses to be logged. This usually includes the *Authorization* header but could also apply to certain plaintext query or form parameters, e.g. *password*.

Logbook differentiates between `Obfuscator` (for headers and query parameters) and `BodyObfuscator`. The default behaviour for all of them is to **not** obfuscate at all.

You can use predefined obfuscators:

```java
Logbook logbook = Logbook.builder()
    // will replace the Authorization header value with XXX
    .headerObfuscator(authorization())
    .build();
```

or create custom ones:

```java
Logbook logbook = Logbook.builder()
    .parameterObfuscator(obfuscate("password"::equals, "XXX"))
    .build();
```

or combine them:

```java
Logbook logbook = Logbook.builder()
    .headerObfuscator(compound(
        authorization(), 
        obfuscate("X-Secret"::equals, "XXX")))
    .build();
```

## Formatting

Formatting defines how requests and respones will be transformed to strings basically. Formatters do **not** specify where requests and responses are logged to, that's the work of writers.

Logbook comes with two different formatters by default - *HTTP* and *JSON*:

### HTTP Style

*HTTP* is the default formatting style, is provided by the `DefaultHttpLogFormatter` and is primarily designed to be used for local development and debugging. Since it's harder to read by machines this is usually not meant to be used in production.

#### Request

```http
GET /test HTTP/1.1
Accept: application/json
Host: localhost
Content-Type: text/plain

Hello world!
```

#### Response

```http
HTTP/1.1 200
Content-Type: application/json

{"value":"Hello world!"}
```

### JSON

*JSON* is an alternative formatting style, is provided by the `JsonHttpLogFormatter` and is primarily designed to be used for production since it's easily consumed by parsers and log consumers.

#### Request

```json
{
  "sender": "127.0.0.1",
  "method": "GET",
  "path": "/test",
  "headers": {
    "Accept": "application/json",
    "Content-Type": "text/plain"
  },
  "params": {
    "limit": "1000"
  },
  "body": "Hello world!"
}
```

#### Response

```json
{
  "status": 200,
  "headers": {
    "Content-Type": "text/plain"
  },
  "body": "Hello world!"
}
```

## Writing

Writing defines where formatted requests and responses are written to. Logback comes with two implementations:

### Logger

By default requests and respones are logged using a *slf4j* logger that uses the `org.zalando.logbook.Logbook` category and the log level `trace`. This can be customized though:

```java
Logbook logbook = Logbook.builder()
    .writer(new DefaultHttpLogWriter(
        LoggerFactory.getLogger("http.wire-log"), 
        Level.DEBUG))
    .build();
```

### Stream

An alternative implementation is logging requests and responses to a `PrintStream`, e.g. `System.out` or `System.err`. This is usually a bad choice for running in production, but might be use for short-term local development and/or investigations.

```java
Logbook logbook = Logbook.builder()
    .writer(new StreamHttpLogWriter(System.err))
    .build();
```

## Servlet

### Dependency

```xml
<dependency>
    <groupId>org.zalando</groupId>
    <artifactId>logbook-servlet</artifactId>
    <version>${logbook.version}</version>
</dependency>
```

You have to register the `LogbookFilter` as a `Filter` in your filter chain.

Either in your `web.xml` file:

```xml
<filter>
    <filter-name>LogbookFilter</filter-name>
    <filter-class>org.zalando.logbook.LogbookFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>LogbookFilter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>ASYNC</dispatcher>
    <dispatcher>ERROR</dispatcher>
</filter-mapping>
```
(Please note that the xml approach will use all the defaults and is **not** configurable.)

Or programmatically via the `ServletContext`:

```java
context.addFilter("LogbookFilter", new LogbookFilter(logbook))
    .addMappingForUrlPatterns(EnumSet.of(REQUEST, ASYNC, ERROR), true, "/*"); 
```

## Spring

### Dependency

```xml
<dependency>
    <groupId>org.zalando</groupId>
    <artifactId>logbook-spring</artifactId>
    <version>${logbook.version}</version>
</dependency>
```

The `logbook-spring` module contains a `ClientHttpRequestInterceptor` to be used with Spring's `RestTemplate`:

```java
RestTemplate template = new RestTemplate();
template.setInterceptors(asList(new LogbookClientHttpRequestInterceptor(logbook)));
```

## License

Copyright [2015] Zalando SE

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.