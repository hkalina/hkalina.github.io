---
layout: post
title: SSL key switch without server restart
---

The upcomming WildFly 11 (from 11.0.0.Beta1) using Elytron security framework to secure HTTPS connections supports key and certificate exchange without application server restart. This blogspot describes how to use Elytron for simple HTTPS configuration and how to make mentioned SSL certificate and key exchange.

Lets suppose you have already your SSL key and certificate prepared in keystore file in `standalone/configuration` directory. For testing purposes you can generate self-signed certificate:

{% highlight text %}
keytool -genkeypair -alias alias1 -keyalg RSA -keysize 1024 -validity 365 -keystore standalone/configuration/keystore.jks -dname "CN=alias1" -storetype JKS -storepass secret1 -keypass secret2
{% endhighlight %}

To be able to test certificate/key switching lets generate second testing keystore too:

{% highlight text %}
keytool -genkeypair -alias alias2 -keyalg RSA -keysize 1024 -validity 365 -keystore standalone/configuration/keystore2.jks -dname "CN=alias2" -storetype JKS -storepass secret1 -keypass secret2
{% endhighlight %}

Now you can register generated keystore in Elytron and use it to secure HTTPS connections.

## Elytron configuration

At first the keystore file needs to be registered in Elytron subsystem:

{% highlight text %}
/subsystem=elytron/key-store=httpsKS:add(path=keystore.jks,relative-to=jboss.server.config.dir,credential-reference={clear-text=secret1},type=JKS)
{% endhighlight %}

Now you can configure appropriate key manager, which will use the keystore to provide key into the SSL context:

{% highlight text %}
/subsystem=elytron/key-manager=httpsKM:add(key-store=httpsKS,algorithm="SunX509",credential-reference={clear-text=secret2})
{% endhighlight %}

The next step will to configure server SSL context, which will be used to establish secured connections:

{% highlight text %}
/subsystem=elytron/server-ssl-context=httpsSSC:add(key-manager=httpsKM,protocols=["TLSv1.2"])
{% endhighlight %}

## Management configuration

For using created SSL context to secure management inteface you need to set it into the HTTP management interface resource:

{% highlight text %}
/core-service=management/management-interface=http-interface:write-attribute(name=ssl-context, value=httpsSSC)
/core-service=management/management-interface=http-interface:write-attribute(name=secure-socket-binding, value=management-https)
reload
{% endhighlight %}

## Undertow configuration

For using created SSL context to secure regular HTTPS, providing deployed applications, you need to replace legacy security domain by new SSL context in Undertow HTTPS listener:

{% highlight text %}
batch
/subsystem=undertow/server=default-server/https-listener=https:write-attribute(name=ssl-context, value=httpsSSC)
/subsystem=undertow/server=default-server/https-listener=https:undefine-attribute(name=security-realm)
run-batch
reload
{% endhighlight %}

Now, when you navigate to the web published by application server, you will see it secured by your first certificate:

![Certificate of alias1 visible by web browser](/images/firefox-alias1.png "Certificate of alias1 visible by web browser")

## Certificate and key exchange

Finally, to exchange certificate and key without server restart follow steps:

Exchange keystore file on the filesystem:

{% highlight text %}
mv standalone/configuration/keystore.jks standalone/configuration/keystore1.jks
mv standalone/configuration/keystore2.jks standalone/configuration/keystore.jks
{% endhighlight %}

Reload Elytron key-store resource:

{% highlight text %}
/subsystem=elytron/key-store=httpsKS:load()
{% endhighlight %}

Optionally check content of loaded key-store:

{% highlight text %}
/subsystem=elytron/key-store=httpsKS:read-aliases
{
    "outcome" => "success",
    "result" => ["alias2"]
}
/subsystem=elytron/key-store=httpsKS:read-alias(alias=alias2)
{
    "outcome" => "success",
    "result" => {
        "alias" => "alias2",
        "entry-type" => "PrivateKeyEntry",
        "creation-date" => "2017-10-13T11:30:27.270+0200",
        "certificate-chain" => [{
            "type" => "X.509",
            "algorithm" => "RSA",
            "format" => "X.509",
            "public-key" => "...",
            "sha-1-digest" => "5f:f8:eb:4f:83:25:80:8a:67:de:c6:bf:7d:b9:35:80:95:aa:6d:38",
            "sha-256-digest" => "a4:b4:ac:00:1a:89:d1:5c:eb:c1:5b:86:d0:f1:95:f5:c1:b9:dd:...",
            "encoded" => "...",
            "subject" => "CN=alias2",
            "issuer" => "CN=alias2",
            "not-before" => "2017-10-13T11:30:27.000+0200",
            "not-after" => "2018-10-13T11:30:27.000+0200",
            "serial-number" => "06:b2:f9:7a",
            "signature-algorithm" => "SHA256withRSA",
            "signature" => "...",
            "version" => "v3"
        }]
    }
}
{% endhighlight %}

And when you are sure that correct key-store is on place, you can apply changes by reinitializing key manager resource:

{% highlight text %}
/subsystem=elytron/key-manager=httpsKM:init()
{% endhighlight %}

From this moment the new certificate/key should be used for any new SSL connection. When you check the certificate in web browser, you should see that the new certificate is used:
![Certificate of alias2 visible by web browser](/images/firefox-alias2.png "Certificate of alias2 visible by web browser")

