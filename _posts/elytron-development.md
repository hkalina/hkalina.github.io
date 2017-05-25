---
layout: post
published: false
title: WildFly Elytron development
---

Fork [wildfly-elytron](https://github.com/wildfly-security/wildfly-elytron), [wildfly-core](https://github.com/wildfly-security-incubator/wildfly-core/) and [wildfly](https://github.com/wildfly-security-incubator/wildfly/) repos.

Clone that forks to your computer and add upstream (incubator) remotes:

{% highlight bash %}
git clone git@github.com:honza889/wildfly-elytron.git
cd wildfly-elytron
git remote add upstream git@github.com:wildfly-security/wildfly-elytron.git
cd ..
{% endhighlight %}

{% highlight bash %}
git clone git@github.com:honza889/wildfly-core.git
cd wildfly-core
git remote add incubator git@github.com:wildfly-security-incubator/wildfly-core.git
git remote add upstream git@github.com:wildfly/wildfly-core.git
cd ..
{% endhighlight %}

{% highlight bash %}
git clone git@github.com:honza889/wildfly.git
cd wildfly
git remote add incubator git@github.com:wildfly-security-incubator/wildfly.git
git remote add upstream git@github.com:wildfly/wildfly.git
cd ..
{% endhighlight %}

Build wildfly-elytron:

{% highlight bash %}
cd wildfly-elytron
mvn clean install [-DskipTests]
cd ..
{% endhighlight %}

Modify wildfly-core's pom.xml to reference wildfly-elytron you just built:

{% highlight bash %}
vers=$(sed -ne "s/[^<]*<version>\([^-<]*\)-SNAPSHOT</version>.*/\1/p" wildfly-elytron/pom.xml)



cd wildfly-elytron
mvn clean install [-DskipTests]
cd ..
{% endhighlight %}



