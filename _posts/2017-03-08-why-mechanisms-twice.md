---
layout: post
title: Why mechanisms twice?
---

Why do we need to specify `configurable-sasl-server-factory` (or `configurable-http-server-factory`) in Elytron configuration to filter mechanisms, which we want to allow to use, when we have already defined them all in `sasl-authentication-factory` (`http-authentication-factory`)? It looks weird to have to write them twice:

{% highlight xml %}
<sasl-authentication-factory name="myFactory" sasl-server-factory="myConfigurable" security-domain="ManagementDomain">
    <mechanism-configuration>
        <mechanism mechanism-name="PLAIN"/>
    </mechanism-configuration>
</sasl-authentication-factory>
<configurable-sasl-server-factory name="myConfigurable" sasl-server-factory="global">
    <filters>
        <filter>
            <pattern-filter value="PLAIN"/>
        </filter>
    </filters>
</configurable-sasl-server-factory>
{% endhighlight %}

Sure, it look so, but the key is, the mechanism configuration **does not need to be based on mechanism names** only!

Look:

{% highlight xml %}
<sasl-authentication-factory name="myFactory" sasl-server-factory="configurable" security-domain="ManagementDomain">
    <mechanism-configuration>
        <mechanism host-name="localhost" protocol="https">
            <mechanism-realm realm-name="ApplicationRealm" />
        </mechanism>
    </mechanism-configuration>
</sasl-authentication-factory>
{% endhighlight %}

In this example was the mechanism configured by hostname and protocol - specified configuration can be used for any mechanism, when hostname will be `localhost` AND protocol will be `https`. Nobody cares about mechanism itself.

As you can see, it is not possible to filter provided mechanisms by `mechanism-configuration` section, because it does not need to contain all informations about them.

