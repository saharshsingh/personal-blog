+++
author = "Saharsh Singh"
title = "Publishing Java Artifacts to Central Maven repository"
slug= "publish-to-maven"
date = "2018-01-01"
description = "In this blog post I go over uploading Java artifacts to the Central Maven repository"
tags = [
    "code",
    "howto",
    "java",
    "sdlc",
    "software",
    "tech",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/2018/01/01/publishing-java-artifacts-to-central-maven-repository/",
]
+++

In this blog post I go over uploading Java artifacts to the Central Maven repository, so that they can be used as dependencies by any developer in any Java project in the world (provided they have an internet connection and a Maven repository compatible build or dependency management tool).

<!--more-->

## Introduction

So you like Java. You are comfortable with it. You are productive with it. You might even consider yourself an expert in it. You have significantly contributed to multiple serious Java projects and have found yourself writing some really powerful, or at the very least reusable, Java code you have used across multiple projects. Being a successful software professional, you find yourself working in different private code bases owned by specific companies as you progress through your career. To bring in your libraries, you are either copy/pasting code or, worse, reimplementing the same thing over and over again.

Or, maybe, that’s not your issue. Instead, you have crafted some once in a lifetime Java code that can be leveraged by developers worldwide to build other amazing applications. You have uploaded your source code on Github or another open source hosting platform, and can’t wait for other developers to start discovering the magic you have bestowed upon them. However, for other developers to include your library in their projects, they have to download a raw JAR file and manually set it up as a dependency in their build system. Worse yet, they even have to build the JAR file from raw source code as you don’t have a publicly accessible hosting location that updates as your code updates.

Stop that! This article is here to make sure no one has to do the above ever again. So strap in as we figure out how you can leverage the Central Maven repository to publish your own Java project as a public library for anyone to include in their projects.

### Disclaimers

But, before we get started, some disclaimers:

1. This article is best for people who are comfortable using Maven to build Java projects. Maven is not the only choice for publishing your Java library to the world. Gradle and Ivy can probably do the job just fine. However, I won’t cover those tools here as I personally haven’t validated that.

1. I have used Sonatype’s Open Source Software Repository Hosting (OSSRH) to upload my artifacts to the Central Maven repository. If you want to learn how to push your artifact to the Central repository using some other specific hosting service or technology, you may not get much out of this article.

1. Pretty much all the information provided in this article is available via the official web sites of [Maven](https://maven.apache.org/guides/mini/guide-central-repository-upload.html) and [Sonatype](http://central.sonatype.org/pages/ossrh-guide.html). However, I had a hard time finding all the relevant information captured step by step in one convenient page. Therefore, I believe this post can create value by compiling all the necessary information in one place.

## Crash Course in Dependency Management

Ok, now some background information to get us on the same page before starting. Feel free to skip this section and jump right to Publish to Central Repository if you are confident you know what the terms **software libraries**, **dependency management**, **Maven**, and **Central Maven Repository** mean, and why publishing your library to the Central Maven Repository is a great idea.

### Applications vs Libraries

This article is about sharing libraries. They are different from applications. Applications are something the end user (not the developer) directly consumes for some business or personal value. They are web pages, mobile applications, programs installed on your computer, among others. Developers typically share applications in different ways depending on the target platform. Web applications are shared by having end users access a URL through their browser. Mobile applications are typically downloaded and installed using the specific mobile operating system’s approved app store. Similarly, end users consume desktop applications by either directly executing the application or its installer, a file they either downloaded from the internet or received via some other portable memory device.

Libraries are different. Libraries are code modules developers can leverage when writing code of their own. This way they don’t have to keep writing new code to solve the same problems over and over again. Some of the most widely used libraries typically come bundled in with the language. For example, Java’s **rt.jar** contains, other than classes that make up the Java programming language itself, classes that implement the most fundamental data structures and algorithms, operations on the local filesystem, abstractions that make it easy to work with common network protocols like TCP/IP and UDP, and more. You don’t have to look too hard to find **rt.jar** as it comes bundled inside any JDK installation. However, **rt.jar** is not complete by any means. Vast amounts of library code has been written, packaged as JAR files or similar modules, and distributed to developers across the world to make Java more feature rich and powerful. The immensely popular and ubiquitous [Apache Commons library](http://commons.apache.org/) is a great example. However, this article is not a survey of the best Java libraries. For that check out awesome articles on other blogs like [Code Like a Girl](https://code.likeagirl.io/common-libraries-in-java-and-where-to-find-them-7d72477dc96b) or [DiscoverSDK](http://www.discoversdk.com/blog/15-java-libraries-every-developer-should-know). Better yet, check out JavaWorld’s slightly dated [list](https://www.javaworld.com/article/2924315/open-source-tools/javas-top-20-the-most-used-java-libraries-on-github.html) of most used Java libraries acrosss Github projects.

### Sharing Libraries

No, this article is about creating and sharing your own libraries. More precisely, this is an article that will help you share Java libraries you have already created. Creating the libraries, or in other words writing the code – the fun part – is up to you. I will make sure sharing it – the boring part – is explained in enough detail here so you don’t have to spend much time thinking about it. Sharing software libraries has two parts:

1. Library creator(s) must make the library accessible to the target audience.

1. Members of the target audience must incorporate the right version of the library in their projects’ builds

The most basic way you can share a Java library is by distributing your code as either raw source files or compiled bytecode (`.class` files) packaged as a JAR file or a similar module. To make the library files accessible, you can either send them directly to the interested developers or you can host them over the network at a location that interested parties can download from using HTTP, FTP, or another similar protocol. The developers using your library can then compile the source code if needed, and place the compiled bytecode on their project’s classpath. In fact, years ago, this was business as usual. I have worked on multiple projects in different companies where each Java project had a version controlled `lib` folder that contained that project’s external dependencies as JAR files. It was up to the project’s developers to periodically locate updates to those libraries, bring them into the `lib` folder, and commit the update to the version control system. Yeah.. not fun! Worse, the complexity and tediousness of this process exacerbates rapidly when transitive dependencies (libraries needed by libraries that you need) are introduced into the mix. Transitive dependencies are unavoidable on most Java projects. Naturally, sharing and consuming libraries in the very manual way described above does not scale well. The need for automated package and dependency management makes itself apparent in any serious software development endeavor. In fact, the rapid pace at which open source software has matured and evolved over the years would be not be possible without automated package and dependency management.

### Dependency Management in Java

Dependency management solutions (as well as package managers) have been created in almost every mainstream language. In Java, [Maven](https://maven.apache.org/) and [Gradle](https://gradle.org/) have become the most popular options for build, dependency, and package management. Gradle vs Maven can be a hot debate, although one that Gradle seems to be winning more and more. It hasn’t won it for me yet. To me, typical Gradle and Maven build configurations are largely the same. Also, I prefer to keep my build files as simple and devoid of custom code as possible. Having said that, I do think Gradle syntax is a LOT cleaner. Also, if you do need customization in your builds, Gradle is your build tool. Regardless, as far as the scope of this article is concerned, Gradle vs Maven debate is besides the point. Both tools search [Maven’s Central Repository](http://search.maven.org/) to resolve a project’s declared dependencies. Therefore, if your goal is to release a Java library with the goal of public adoption, then releasing it to Maven’s Central Repository is the way to go. This ensures all Maven and Gradle users – as well as users of other JVM based build tools like Ivy (distant 3rd in Java world), SBT (Scala), Grape (Groovy), and Leiningen (Clojure) – can transparently declare the appropriate version of your artifact as a dependency of their projects.

Final piece of terminology to demystify here is the **Central repository**. Well, before we get there, let’s just address what repository means in this context. If you’ve been using Maven to build your Java projects (and this article assumes you have), then you are somewhat familiar with `package`, `install`, and maybe even `deploy` build goals. The `package` goal is the next step in the build cycle after your project is compiling and passing its tests. The `package` goal, by default, creates a JAR file that contains all your production `.class` files and any other production classpath files and places it in the project’s `target` folder. The `install` goal takes that JAR file and copies it over into your local Maven repository, located at `.m2/repository` in your home directory by default.  Now you can have your other Java projects on your computer declare this JAR file as a dependency in their `pom.xml` files. The local repository is what Maven searches first to resolve dependencies declared in your `pom.xml` files. Whatever dependencies Maven can’t find in the local `.m2/repository` folder, it looks for in the configured remote repositories. While you can configure your own private remote repositories (typically hosted using Nexus or Artifactory), by default Maven will search the Central Maven repository for dependencies it can’t find locally.

Think of the Central repository as a bigger `.m2/repository` that is accessed over the Internet rather than the local filesystem and is shared by everyone in the world. By running install, as described above, you make a stand alone library available to other projects on the same computer. However, that does not enable other developers using separate computers to start using your library. To do that you have to use the `deploy` goal to upload your built artifacts, i.e. JAR and POM files, to a remote repository to which all relevant developers have access. A privately hosted Maven repository (typically using Nexus or Artifactory) allows developers in an organization to easily share libraries. To share your libraries with the world, on the other hand, you must get your library artifacts inside the Central Repository. How can we do that? We are ready to find out.

## Publish to Central Repository

Alright, let’s get down to business. Following is a handy checklist for publishing artifacts to the Central Maven repository. We’ll explore each step in detail below.

1. Publicly host your project’s source code
1. Retrieve an OSSRH Group ID
1. Setup GPG/PGP for JAR signing
1. Prepare your project’s pom.xml and your settings.xml for release
1. Perform release using Maven’s release plugin

Finally, I’ll build an example `pom.xml` as I go along the steps below. I am essentially re-building the [`pom.xml` of my master-pom project](https://github.com/saharshsingh/master-pom/blob/master/pom.xml) that I use as the Maven parent for all my open source Java libraries. You’ll notice the `<packaging>pom</packaging>` element in this POM file which tells Maven the intended deliverable of this project is a simple POM file. If you have a traditional Java project whose intended deliverable is a JAR file, then just simply omit this line.

### Hosting Your Project

The Central Repository does not support direct uploads. Instead your artifact must be part of an approved repository hosting location. The easiest location, and also the one recommended by Central Repository itself, is [Open Source Software Repository Hosting (OSSRH)](http://central.sonatype.org/pages/ossrh-guide.html). That’s what I recommend you use as well. Hosting with OSSRH has its caveats though. They are by definition a repository for open source projects and have strict requirements around that. You will need to host your source code in a publicly accessible location (typically Github or similar service), and provide that location in your artifact’s `pom.xml` file via the `<scm />` tag. Therefore, if you haven’t already done so, create a Github account and upload your project’s source code as a public repository in your account.

NOTE: OSSRH folks may make an exception for you if you insist upon keeping your source code private and still using their repository to host your built artifacts. As far as I know, this is done on [a case by case basis](https://issues.sonatype.org/browse/MVNCENTRAL-692) and is not guaranteed.

### OSSRH Group ID

Maven artifacts must be uniquely identifiable using two pieces of information: **Group ID** and **Artifact ID**. Artifact ID is just some arbitrary name you’ll choose as the name of your project. Group ID, on the other hand, is more of a namespace identifier, and something that’s typically assigned to multiple related projects. For public projects, you will need to have a Group ID that’s unique. Also, OSSRH requires that you actually have the right to use your Group ID based on domain name ownership. For example, I use `org.saharsh` as the Group ID for my open source projects. I am able to do this since I own the saharsh.org domain and registered my OSSRH JIRA account using an email address from this domain. If you don’t want to purchase a domain, then you can use a group ID derived from your project hosting. Given my Github account is [`saharshsingh`](https://github.com/saharshsingh), I also have the option to use `com.github.saharshsingh` or `io.github.saharshsingh` as a Group ID. See [official OSSRH documentation](http://central.sonatype.org/pages/choosing-your-coordinates.html) on this for more details.

Once you have decided what group ID you wish to use, go ahead and create a Sonatype JIRA ticket to get this group ID registered with OSSRH. First you will need to [sign up for an account](https://issues.sonatype.org/secure/Signup!default.jspa). Then you can [create an issue](https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134) using your new JIRA account to register your Group ID with OSSRH. You can look at the [JIRA issue I created](https://issues.sonatype.org/browse/OSSRH-36012) to register `org.saharsh` for an example. It typically takes just an hour or two, but can take up to two business days, for OSSRH team to respond to your request. Once your request meets all their requirements, you can start uploading artifacts to their snapshot or staging repositories using the credentials you provided when creating your JIRA account.

### Setup GPG/PGP

Before you can start setting up your POM for release, there is one other dependency you need to square away. The Central Repository requires that all artifacts uploaded are signed using GPG/PGP encryption. To be able to accomplish this, you will need to setup proper tooling on the computer that you wish to use to release your library.  GnuPGP is a great resource for this, and it is officially recommended by OSSRH. [Download](https://www.gnupg.org/download/) and install the appropriate version depending on the OS your computer is using. Once you have the tooling installed, create a new key pair and take note of the `keyname` and `passphrase` you chose. You’ll be including these in your `settings.xml` below.

### Setup Basic `pom.xml` Information

Alright, let’s setup your project’s pom.xml so that it is configured to release to Central Repository. First of all, make sure the basic identifying information is correct. Use the following example as guidance.

{{< highlight xml "linenos=table" >}}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

    <groupId>org.saharsh</groupId>
    <artifactId>master-pom</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>Master POM</name>
    <description>Master POM used by all org.saharsh projects</description>
    <url>https://github.com/saharshsingh/master-pom</url>

    <scm>
        <url>https://github.com/saharshsingh/master-pom</url>
        <connection>scm:git:git://github.com/saharshsingh/master-pom.git</connection>
        <developerConnection>scm:git:git@github.com:saharshsingh/master-pom.git</developerConnection>
        <tag>HEAD</tag>
    </scm>

    <developers>
        <developer>
            <email>singh@saharsh.org</email>
            <name>Saharsh Singh</name>
            <url>http://saharsh.org</url>
            <id>saharshsingh</id>
        </developer>
    </developers>

    <licenses>
        <license>
            <name>Apache License, Version 2.0</name>
            <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
            <distribution>repo</distribution>
        </license>
    </licenses>
</project>
{{< /highlight >}}

Pay special attention to the following:

* **Group ID**: Make sure you are using the Group ID you registered with OSSRH earlier.

* **Version**: Make sure this is in the `#.#-SNAPSHOT` format. The `-SNAPSHOT` part is a Maven convention. Snapshot deploys change the `-SNAPSHOT` into `-[TIMESTAMP]` before uploading the artifact, where `[TIMESTAMP]` is the time at deploy. Releases will completely drop `-SNAPSHOT` before uploading artifacts.

* **Packaging**: As I mentioned earlier, my project is a POM project that is used as a Maven parent by my open source projects. Your project might be producing a JAR file. In that case, omit this element. Maven uses JAR as the default packaging.

* **Name, Description, URL**: Central Repository is particular about these elements. Make sure you are choosing reasonable values for these fields. Make sure the URL field is a real URL that points to your project’s source code. Your project’s Github URL is perfect for this.

* **SCM**: Again, this is required due to OSSRH being a repository for open source projects.

* **Developer**: Yes, Central Repository requires you provide at least one developer’s information. It’s OK to use your Github URLs here if none of the developers on your team have or want to share another web presence.

* **License**: Include your project’s license information here. [Apache](https://opensource.org/licenses/Apache-2.0) or [MIT](https://opensource.org/licenses/MIT) licenses are popular examples of licenses used in open source projects.

### Configure OSSRH Repository

Next, you need to declare the OSSRH snapshot repository as the target repository for your project’s snaphost deploys. This is not the same as releasing a version of your project. This just means when you run `mvn deploy`, the built artifacts are uploaded to the OSSRH snapshot repository. In your `pom.xml`, add the following:

{{< highlight xml "linenos=table" >}}
    <distributionManagement>
        <snapshotRepository>
            <id>sonatype-nexus</id>
            <name>Sonatype Nexus Snapshots</name>
            <url>https://oss.sonatype.org/content/repositories/snapshots</url>
        </snapshotRepository>
    </distributionManagement>
{{< /highlight >}}

Now you need to tell Maven your OSSRH credentials so it can succesfully authenticate before deploying. It’s good practice to put this information in your local `settings.xml` file for security reasons. The Maven `settings.xml` file is located in the `.m2` folder in your home directory. If it’s not there, create a new one, and add the following information:

{{< highlight xml "linenos=table" >}}
<settings>
    <servers>
        <server>
            <id>sonatype-nexus</id>
            <username>[your OSSRH username]</username>
            <password>[your OSSRH password]</password>
        </server>
    </servers>
</settings>
{{< /highlight >}}

Note that the `<id>sonatype-nexus</id>` tag is common between the `pom.xml` and the `settings.xml` files. Now when you do `mvn deploy` on your project, you should see your built artifacts get uploaded to the OSSRH snapshot repository. The configuration we provided in `settings.xml` will help with releasing as well.

### Configure Maven Release Plugin

Now we are at the final bit of configuration in your project before you are ready to release. There are multiple ways to release your project. I recommend Maven’s release plugin as it’s Maven’s recommended approach. Once you are setup, you just run simple Maven goals and the artifact will be released and your Git repository will get the appropriate commit tags – very convenient.

Configure the following build plugins in your pom.xml:

{{< highlight xml "linenos=table" >}}
    <build>
        <plugins>
            <plugin>
                <groupId>org.sonatype.plugins</groupId>
                <artifactId>nexus-staging-maven-plugin</artifactId>
                <version>1.6.8</version>
                <extensions>true</extensions>
                <configuration>
                    <serverId>sonatype-nexus</serverId>
                    <nexusUrl>https://oss.sonatype.org/</nexusUrl>
                    <autoReleaseAfterClose>true</autoReleaseAfterClose>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-release-plugin</artifactId>
                <version>2.5.3</version>
                <configuration>
                    <autoVersionSubmodules>true</autoVersionSubmodules>
                    <useReleaseProfile>false</useReleaseProfile>
                    <releaseProfiles>release</releaseProfiles>
                    <goals>deploy</goals>
                </configuration>
            </plugin>
        </plugins>
    </build>
{{< /highlight >}}

I have used the latest versions of these plugins as of December 2017. Feel free to use the latest versions. Careful readers will also notice the `<releaseProfiles>release</releaseProfiles>` tag. This tag is included as we need to add more to our `pom.xml`. Central repository has more requirements for your artifact. For release uploads, you must include Javadoc and Sources JAR files, and you must sign all your JAR files with GPG/PGP encryption. I alluded to the encryption requirement above when I had you setup GnuPGP tooling on your computer. Now we use it. Add the following to your `pom.xml`:

{{< highlight xml "linenos=table" >}}
    <profiles>
        <profile>
            <id>release</id>
            <activation>
                <property>
                    <name>performRelease</name>
                    <value>true</value>
                </property>
            </activation>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-source-plugin</artifactId>
                        <version>3.0.1</version>
                        <executions>
                            <execution>
                                <id>attach-sources</id>
                                <goals>
                                    <goal>jar-no-fork</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-javadoc-plugin</artifactId>
                        <version>2.10.4</version>
                        <executions>
                            <execution>
                                <id>attach-javadocs</id>
                                <goals>
                                    <goal>jar</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-gpg-plugin</artifactId>
                        <version>1.6</version>
                        <executions>
                            <execution>
                                <id>sign-artifacts</id>
                                <phase>verify</phase>
                                <goals>
                                    <goal>sign</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
{{< /highlight >}}

Again, use the latest versions of the above plugins if the version numbers I have provided above have now become outdated. Note that the `<activation />` tag makes sure these plugins only execute during release. The `performRelease` property is set to `true` by the Maven Release plugin during execution.

We have one last thing to configure before you are ready to perform the release. The Maven GPG plugin needs to be configured to use the key pair you created above. As with the OSSRH credentials, this information belongs in your `settings.xml`.

{{< highlight xml "linenos=table" >}}
    <profiles>
        <profile>
            <id>release-sign-artifacts</id>
            <activation>
                <property>
                    <name>performRelease</name>
                    <value>true</value>
                </property>
            </activation>
            <properties>
                <gpg.keyname>[Your GPG key name]</gpg.keyname>
                <gpg.passphrase>[Your GPG key passphrase]</gpg.passphrase>
                <gpg.defaultKeyring>false</gpg.defaultKeyring>
                <gpg.useagent>true</gpg.useagent>
                <gpg.lockMode>never</gpg.lockMode>
                <gpg.homedir>[Path to .gnupg folder]</gpg.homedir>
                <gpg.publicKeyring>[Path to pubring.gpg]</gpg.publicKeyring>
                <gpg.secretKeyring>[Path to secring.gpg]</gpg.secretKeyring>
            </properties>
        </profile>
    </profiles>
{{< /highlight >}}

Take note of the following:

* gpg.keyname: This is the key name you chose earlier
* gpg.passphrase: This is the key passphrase you chose earlier
* gpg.homedir: This should be the full path to the `.gnupg` folder in your home directory
* gpg.publicKeyring: Full path to the public key ring file in the `.gnupg` folder that contains your key. Typically `pubring.gpg`
* gpg.secretKeyring: Full path to the secret key ring file in the `.gnupg` folder that contains your key. Typically `secring.gpg`

### Release!

Releasing your project at this point is straightforward. Simply use the Maven release plugin that you have already configured in your POM file. There are three Maven goals to run.

* `mvn release:prepare` – This goal will prepare your project for release. You will choose the release version as well as the Git tag to apply as part of the release

* `mvn release:perform` – This goal will execute the actual release. Your artifacts will be uploaded to OSSRH’ staging repository and released to Central Repository (except your very first release). On successful release, the Git tag you specified will also get created and committed into your local Git repository.

* `mvn release:clean` – This will delete any temporary files the release process created.

And that’s it! On your very first release, you will need to let the OSSRH team know that your release attempt went smoothly via the Group ID JIRA ticket you created earlier. After this notification they will turn on sync to Central Release. Boom! Now your libraries are available for public consumption.
