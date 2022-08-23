+++
author = "Saharsh Singh"
title = "Builder Pattern and Fluent Interfaces"
slug= "builder-pattern-and-fluent-interfaces"
date = "2018-02-12"
description = "I love fluent interfaces, and I love the builder pattern. In this post, I’ll use a Java library I recently open sourced to talk about both."
tags = [
    "code",
    "design patterns",
    "java",
    "object-oriented programming",
    "software",
    "tech",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/2018/02/12/builder-pattern-and-fluent-interfaces/",
]
+++

I love fluent interfaces, and I love the builder pattern. In this post, I’ll use a Java library I recently open sourced to talk about both.

<!--more-->

## Introduction

It’s often repeated among experienced software professionals that code is read FAR more than it is written. After spending a decade in the industry working for various companies as a software developer, I can assure you that the statement is not only true, it’s almost an understatement. Very rarely do developers treat reading code as a leisurely activity for the beach or a healthy pleasure before bedtime. No, developers mostly find themselves reading code because they need to use it to do something. More frequently, the code they are reading is broken or needs to be updated, and they are the one accountable for making sure that happens. This leads to another often repeated quote among software developers: “Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live.” What does this have to do with fluent interfaces and the builder pattern? I’m glad you asked.

As objects grow in size and complexity, configuring them, executing methods on them, and passing them as arguments to other object methods quickly leads to cumbersome syntax that’s tough to read and maintain. There are actually two distinct concerns at play here:

1. Object construction and configuration (addressed by the Builder pattern)
2. Code self documenting its domain level implications (addressed by Fluent interfaces)

## Builder Pattern

First we have object construction and configuration. Typically objects are either created via constructors alone, or via a mix of constructors and setter methods. When you have classes with optional fields, neither strategy is pretty. Setters mean your objects are mutable. This may be fine for data objects that will only ever exist at functional scope, but dangerous for pretty much anything else. Immutability is a powerful property for non-data objects to have. I mostly write web services, and it has served me well to create long term objects that process data (e.g. web request handlers and their business logic delegates) as immutable objects that receive their dependencies via constructors. After instantiation, they are only passed data objects (e.g. deserialized web requests) at functional scope, and their instance state rarely changes. Coding like this makes your code inherently thread-safe and mostly leads to modular code that is easy to test and evolve. By allowing consumers of your class to mutate its internal state via setters (i.e. your class’s official contract), your class gives up control over its internal state. This can break encapsulation, create unintended abstraction leaks, and lead way to allowing consumers of your class to create unnecessary coupling between their abstractions and your class. All of these basically limit your options as to what assumptions you can make when evolving your class’s internal implementation or making guarantees about its functionality.

Handling optional fields via constructors has its own pains. You can either expose one constructor that makes optional fields nullable, or declare a separate constructor for each subset of field combinations. First approach leads to bad readability in languages that only do positional parameters. It’s hard to tell by just reading constructor invocations which specific field each ‘null’ is referring to. And as I mentioned earlier, bad readability typically translates to bad maintainability. Not to mention, if you need to introduce additional fields in the future, you have no option but to introduce breaking changes in your class’s consumers. Multiple constructors, on the other hand, can quickly become a maintenance nightmare for the class maintainer. This is where the builder pattern comes into play.

Builder pattern combines the convenience of setters with the safety and simplicity of immutability. Typically you create a class with one constructor and immutable state, and a companion builder class with a method for each field that can be provided to the constructor. Each of the field setter methods in the builder returns the instance of builder back to enable method chaining when setting fields. Finally the builder class has a ‘build’ method that calls the target class’s constructor, providing user provided values where possible and using some default value or ‘null’ for others. You can also include validation logic in each of your field setter methods as well as the final build method to ensure target object is only created with valid state.

Let’s look at an example. Let’s say we write a library that is a proxy to ElasticSearch. It exposes an interface, `SearchProxy`, that defines the contract for this library.

{{< highlight java "linenos=table" >}}
package org.saharsh.search;
 
public interface SearchProxy {
    SearchResult search(SearchRequest search);
}
{{< /highlight >}}

We provide a package private implementation class for this interface that depends on [io.searchbox.client.JestClient](https://github.com/searchbox-io/Jest/tree/master/jest) for interaction with ElasticSearch. It also optionally depends on Dropwizard’s [com.codahale.metrics.MetricRegistry](https://github.com/dropwizard/metrics) for recording metrics.

{{< highlight java "linenos=table" >}}
package org.saharsh.search;

import com.codahale.metrics.MetricRegistry;
import io.searchbox.client.JestClient;

// intentionally package private
class SearchProxyImpl {

    private final JestClient jestClient;
    private final MetricRegistry metricRegistry;

    SearchProxyImpl(JestClient jestClient, MetricRegistry metricRegistry) {
        this.jestClient = jestClient;
        this.metricRegistry = metricRegistry;
    }

    @Override
    public SearchResult search(SearchRequest search) {
        // search implementation goes here
    }
}
{{< /highlight >}}

Finally, as part of the builder pattern, we also provide a builder implementation to consumers of this library for instantiating the Search proxy.

{{< highlight java "linenos=table" >}}
package org.saharsh.search;

import com.codahale.metrics.MetricRegistry;

import io.searchbox.client.JestClient;
import io.searchbox.client.JestClientFactory;
import io.searchbox.client.config.HttpClientConfig;

public class SearchProxyBuilder {

    private String scheme = "http";
    private String host = "localhost";
    private int port = 9200;
    private MetricRegistry metricRegistry = null;

    /**
     * @param scheme
     *            ElasticSearch URL scheme. 'http' or 'https'. (Default: http)
     * @return builder instance
     */
    public SearchProxyBuilder esScheme(String scheme) {
        if("http".equals(scheme) || "https".equals(scheme)) {
            this.scheme = scheme;
            return this;
        }

        throw new IllegalArgumentException("Invalid ElasticSearch URL scheme: " + scheme);
    }

    /**
     * @param host
     *            ElasticSearch host (Default: localhost)
     * @return builder instance
     */
    public SearchProxyBuilder esHost(String host) {
        this.host = host;
        return this;
    }

    /**
     * @param port
     *            ElasticSearch port (Default: 9200)
     * @return builder instance
     */
    public SearchProxyBuilder esPort(int port) {
        this.port = port;
        return this;
    }

    /**
     * @param registry
     *            (optional) Metrics logging disabled if not provided
     * @return builder instance
     */
    public SearchProxyBuilder metricRegistry(MetricRegistry registry) {
        metricRegistry = registry;
        return this;
    }

    /** @return SearchProxy instance configured according to builder state */
    public SearchProxy build() {

        // build JestClient
        JestClientFactory factory = new JestClientFactory();
        String url = scheme + "://" + host + ":" + port;
        factory.setHttpClientConfig(new HttpClientConfig.Builder(url).defaultSchemeForDiscoveredNodes(scheme).build());
        JestClient jestClient = factory.getObject();

        // build Search Proxy
        return new SearchProxyImpl(jestClient, metricRegistry);
    }
}
{{< /highlight >}}

`SeachProxyBuilder` is mutable, does not guarantee thread safety, and that is completely fine. Instances of the builder are meant to be ephemeral and used by a single thread, typically at application startup. Once `build` is called on the builder, an immutable object is returned. It is this object that makes guarantees about the library’s core functionality and thread safety. Neither the interface nor the concrete implementation expose any public methods that consumers can use to directly manipulate internal state. All exchange of information is encapsulated within the message or data objects, `SearchRequest` and `SearchResult`. I hope you can see how this can go a long way in helping developers write modular code that can be easily tested and refactored.

## Fluent Interfaces

In the builder example above, each field setter method returns the builder instance. To be strict, this is not a requirement of the builder pattern. The builder pattern is primarily concerned with configuring an object via a separate ephemeral object. I could have easily used the `void` return type for each setter method. Of course then I wouldn’t be able to chain the method invocations together for a one liner, and the builder would lose its **fluency**. This leads us to our second concept for the day, **fluent interfaces**.

You’ll notice that way back in [Introduction](#introduction), the second concern is NOT **enable method chaining**. Even though method chaining is at the heart of them, fluent interfaces are about more than just leveraging method chaining to create one liners. The main motivation behind fluent interfaces is to produce code that reads more like prose and therefore, if implemented correctly, is self documenting and explicitly states its domain level implications. Essentially, libraries or modules that are written to be fluent end up delivering domain specific languages just from how they are consumed in code. Martin Fowler is one of the people credited with coining the term, and his [excellent blog article](https://martinfowler.com/bliki/FluentInterface.html) on the matter drives the point home much better than I probably can.

I chose to pair the builder pattern and fluent interfaces in one article, because both go hand in hand. First of all, the core values both are trying to deliver are  readability and maintainability. Although they are dedicated to solving slightly different concerns, both rely heavily on each other to accomplish their goals. Almost all builder implementations you’ll come across today are fluent. On the other hand, implementations of any significant fluent library or module rely heavily on builder classes internally. To see a good example of this at play, check out my recent open source project, [Fluent JDBC](https://github.com/saharshsingh/fluent-jdbc).
