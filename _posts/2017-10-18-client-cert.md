---
layout: post
title: CLIENT-CERT authentication with Elytron
---

This blogspot describes how to use Elytron for two-way (client certificate) SSL authentication. This is draft which requires to have [patch #1018](https://github.com/wildfly-security/wildfly-elytron/pull/1018) merged.

Lets suppose you have already your SSL key and certificate prepared in keystore file in `standalone/configuration` directory. For testing purposes you can generate self-signed certificate:

{% highlight text %}
keytool -genkeypair -alias alias1 -keyalg RSA -keysize 1024 -validity 365 -keystore standalone/configuration/keystore.jks -dname "CN=alias1" -storetype JKS -storepass secret1 -keypass secret2
{% endhighlight %}

Generate also client-side certificate as self-signed:

{% highlight text %}
keytool -genkeypair -alias alias2 -keyalg RSA -keysize 1024 -validity 365 -keystore standalone/configuration/truststore.jks -dname "CN=alias1" -storetype JKS -storepass secret3 -keypass secret4
{% endhighlight %}

Re-export private key into PKCS12 to be able to import it into firefox:

{% highlight text %}
keytool -importkeystore -srcstoretype JKS -deststoretype PKCS12 -srcalias alias2 -destalias alias3 -srckeystore standalone/configuration/truststore.jks -destkeystore standalone/configuration/truststore.p12 -srcstorepass secret3 -srckeypass secret4 -deststorepass secret5 -destkeypass secret5
{% endhighlight %}

Just note that truststore should contain certificate/public key only in production deployment.

## SSL Elytron configuration

When keystore is on place, we can configure `keystore` and `key-manager` to be used by new `server-ssl-context`.
In comparison with simple one-way SSL we need to configure `trust-manager` too and to set `want-client-auth` or `need-client-auth` later.

{% highlight text %}
/subsystem=elytron/key-store=httpsKS:add(path=keystore.jks,relative-to=jboss.server.config.dir,credential-reference={clear-text=secret1},type=JKS,required=true)
/subsystem=elytron/key-store=httpsTS:add(path=truststore.jks,relative-to=jboss.server.config.dir,credential-reference={clear-text=secret3},type=JKS,required=true)
/subsystem=elytron/key-manager=httpsKM:add(key-store=httpsKS,credential-reference={clear-text=secret2})
/subsystem=elytron/trust-manager=httpsTM:add(key-store=httpsTS)
/subsystem=elytron/server-ssl-context=httpsSSC:add(key-manager=httpsKM,trust-manager=httpsTM,protocols=["TLSv1","TLSv1.2"])
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

## Management configuration

For using created SSL context to secure management inteface you need to set it into the HTTP management interface resource:

{% highlight text %}
/core-service=management/management-interface=http-interface:write-attribute(name=ssl-context, value=httpsSSC)
/core-service=management/management-interface=http-interface:write-attribute(name=secure-socket-binding, value=management-https)
reload
{% endhighlight %}

I don't recommand to set this before you will have the SSL configuration and authentication tested sucessfuly in application scope - access to the management console can be disabled by following steps.

## Extending to two-way SSL

At first we need to add user source, which will be used to obtain roles of users logged using certificate.
For now lets add user `test` into default application properties realm - this user will be used for all users autheticated by certificate:

**application-users.properties**:
{% highlight text %}
test=
{% endhighlight %}

**application-roles.properties**:
{% highlight text %}
test=User
{% endhighlight %}

Now we can active `want-client-auth` flag (or `need-client-auth` if you don't want to allow SSL conection without valid client certificate) of the `server-ssl-context`:

{% highlight text %}
/subsystem=elytron/server-ssl-context=httpsSSC:write-attribute(name=want-client-auth, value=true)
{% endhighlight %}

To use user `test` for any user certificate we need to define constant principal transformer:

{% highlight text %}
/subsystem=elytron/constant-principal-transformer=toTest:add(constant=test)
{% endhighlight %}

Before we add HTTP authentication factory, we need to add configuring factory, which will disable certificate verification against the security realm:

{% highlight text %}
/subsystem=elytron/configurable-http-server-mechanism-factory=configuredCert:add(http-server-mechanism-factory=global, properties={org.wildfly.security.http.skip-certificate-verification=true})
{% endhighlight %}

To provide username to the deployed application now we add HTTP authentication factory:

{% highlight text %}
/subsystem=elytron/http-authentication-factory=client-cert-digest:add(http-server-mechanism-factory=configuredCert, security-domain=ApplicationDomain, mechanism-configurations=[{mechanism-name=CLIENT_CERT,final-principal-transformer=toTest}])
{% endhighlight %}

And to define Undertow security domain, which will use it:

{% highlight text %}
/subsystem=undertow/application-security-domain=other:add(http-authentication-factory=client-cert-digest)
{% endhighlight %}

Now should be logging into application using certificate possible - user with valid certificate will get roles of user `test` and the application will get as user principal the certificate DN.

By similar way you can extract some part of the certificate DN and use it to rewrite user principal to user name with appropriate roles in used security realm.

## Troubleshotting

To enable Elytron traces logging:

{% highlight text %}
/subsystem=logging/logger=org.wildfly.security:add(level=TRACE)
{% endhighlight %}

To check keystores availability:

{% highlight text %}
/subsystem=elytron/key-store=httpsKS:read-aliases
/subsystem=elytron/key-store=httpsTS:read-aliases
{% endhighlight %}

