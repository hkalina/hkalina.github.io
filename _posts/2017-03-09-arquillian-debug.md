---
layout: post
title: Debugging WildFly Arquillian test
---

When the worst comes in WildFly development, it is time to debug Arquillian test.
In this article I want to compile instructios how to do it.

The worst thing on it is, you have to debug two JVMs in two debuggers in one time:

## JVM of the test

To debug it run test in maven with following param:

{% highlight text %}
-Dmaven.surefire.debug="-Xdebug -Xrunjdwp:transport=dt_socket,server=n,address=5005"
{% endhighlight %}

For example:

{% highlight text %}
mvn clean install -Dtest=org.jboss.as.test.integration.web.formauth.FormAuthUnitTestCase#testPostDataFormAuth -Dmaven.surefire.debug="-Xdebug -Xrunjdwp:transport=dt_socket,server=n,address=5005"
{% endhighlight %}

This will cause the test will wait for debugger, which is supposed to listen on port 5005.
In IDEA we will setup this Remote debug in test project by folowing way:

**Debug - Edit configuration - Add new configuration - Remote - Socket, Listen, Port: 5005**

At first we should press Debug in IDEA and start the maven test in CLI after.

## JVM of WildFly

To debug JVM of WildFly use following param:

{% highlight text %}
"-Djvm.args.other=-server -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=8787"
{% endhighlight %}

Or you can modify **arquilian.xml** (probably in `src/test/config/arq/arquilian.xml`) - add `-agentlib` param:

{% highlight text %}
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8787
{% endhighlight %}

To the **javaVmArguments**:

{% highlight xml %}
<property name="javaVmArguments">${server.jvm.args} -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8787 -Djboss.inst=${basedir}/target/jbossas -Dtest.bind.address=${node0} -Djava.security.policy==${basedir}/target/test-classes/security.policy</property>
{% endhighlight %}

Now just add debug configuration into IDEA by similar way, but to the project of tested component:

**Debug - Edit configuration - Add new configuration - Remote - Socket, Attach, Port: 8787**

In this case we can start Debug only when JVM is already running - when the test is running. You can:

* press Debug buttom randomly **after** `T E S T S` headline appears in maven output (stupid but fast and often works :)
* to debug JVM of the test and connect second debugger when the test will be waiting on breakpoint

## JVM of JBoss CLI

Sometime you can also need to debug JBoss CLI client. That require to uncomment following near the end of `bin/jboss-cli.sh`:

{% highlight text %}
JAVA_OPTS="$JAVA_OPTS -agentlib:jdwp=transport=dt_socket,address=8787,server=y,suspend=n"
{% endhighlight %}

Just note, don't forgot to change port 8787 to samething else, if you want to debug CLI and server at one time.

## How to run one concrete test using maven

To run one concrete test from WildFly testsuite you can use following maven parameters:

{% highlight text %}
mvn clean install -DallTests -Dmaven.test.failure.ignore=true -pl testsuite/integration/basic -Dtest=org.jboss.as.test.integration.naming.ldap.LdapUrlInSearchBaseTestCase
{% endhighlight %}

Run this in wildfly directory (where you cloned git repository), use `-pl` to specify testsuite directory and use `-Dtest` to specify test in testsuite. Use `-Delytron` if you want to test it.
**Do not remove** `-DallTests`, even through you want to run only one test - it is necessary to run any non-smoke test.

