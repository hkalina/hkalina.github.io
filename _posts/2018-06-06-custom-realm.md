---
layout: post
title: Creating custom security realm for WildFly Elytron
---

This tutorial describes creating custom security realm and its usage in WildFly Elytron subsystem.

To create custom Elytron component you need to create WildFly module containing JAR with class extending interface `SecurityRealm` available from Maven in package `org.wildfly.security.wildfly-elytron`.
This JAR will have to be placed in `modules` directory of your application server together with module descriptor `module.xml`.

## Building the JAR

For simplification this tutorial will assume you use Maven to build your security realm JAR.
There is no special requirement for the JAR - plain JAR archive with class files of your realm needs to be generated.
To have Elytron interfaces available for the compilation, dependency on `wildfly-elytron` will be declared in `pom.xml`:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>my.package</groupId>
    <artifactId>myrealm</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.wildfly.security</groupId>
            <artifactId>wildfly-elytron</artifactId>
            <version>1.2.3.Final</version>
        </dependency>
    </dependencies>

</project>
{% endhighlight %}

As usual in Maven, Java sources will be expected in `src/main/java` subdirectory and a built can be done using `mvn package` command.

## Implementing security realm class

Lets create custom implementation of the security realm - class `MyRealm` extending `SecurityRealm` interface:

{% highlight java %}
package my.package;

import org.wildfly.security.auth.server.SecurityRealm;

public class MyRealm implements SecurityRealm {

}
{% endhighlight %}

Methods required by `SecurityRealm` interface will have to be defined.

Method `getCredentialAcquireSupport` allows to detect, whether the security realm allows to acquire given type of credential.
Our testing security realm will not allow to obtain hash of the password, so we will return `UNSUPPORTED` for any type:

{% highlight java %}
    public SupportLevel getCredentialAcquireSupport(Class<? extends Credential> credentialType,
            String algorithmName, AlgorithmParameterSpec parameterSpec) throws RealmUnavailableException {
        return SupportLevel.UNSUPPORTED;
    }
{% endhighlight %}

Our security realm will, on the other hand, support password verification - when clear password will be given, the realm will be able to say whether it is correct or not. But only for identities it knows - and we are asked for status for any identity - because of that we will not return `SUPPORTED`, but `POSSIBLY_SUPPORTED` only:

{% highlight java %}
    public SupportLevel getEvidenceVerifySupport(Class<? extends Evidence> evidenceType, String algorithmName)
            throws RealmUnavailableException {
        return PasswordGuessEvidence.class.isAssignableFrom(evidenceType) ? SupportLevel.POSSIBLY_SUPPORTED : SupportLevel.UNSUPPORTED;
    }
{% endhighlight %}

Finally, the main method of security realm will allow to obtain object of an identity.
If the identity for given principal does not exists, we can return predefined `NON_EXISTENT` identity.
Otherwise we need to return object of our new identity class:

{% highlight java %}
    public RealmIdentity getRealmIdentity(final Principal principal) throws RealmUnavailableException {
        if (myStorage.exists(principal.getName())) {
            return new RealmIdentity() {
                
            };
        } else {
            return RealmIdentity.NON_EXISTENT;
        }
    }
{% endhighlight %}

You will have to define methods of `RealmIdentity` too. The trivial ones first:

{% highlight java %}
                public Principal getRealmIdentityPrincipal() {
                    return principal;
                }
                public boolean exists() throws RealmUnavailableException {
                    return true;
                }
{% endhighlight %}

As we have already said on realm level, we will not support credential acquisition (obtaining):

{% highlight java %}
                public SupportLevel getCredentialAcquireSupport(Class<? extends Credential> credentialType,
                        String algorithmName, AlgorithmParameterSpec parameterSpec) throws RealmUnavailableException {
                    return SupportLevel.UNSUPPORTED;
                }

                public <C extends Credential> C getCredential(Class<C> credentialType) throws RealmUnavailableException {
                    return null;
                }
{% endhighlight %}

But we will support verification of clear passwords (`PasswordGuessEvidence`s). This time we already know the user exists, so we will return `SUPPORTED`:

{% highlight java %}
                public SupportLevel getEvidenceVerifySupport(Class<? extends Evidence> evidenceType, String algorithmName)
                        throws RealmUnavailableException {
                    return PasswordGuessEvidence.class.isAssignableFrom(evidenceType) ? SupportLevel.SUPPORTED : SupportLevel.UNSUPPORTED;
                }

                public boolean verifyEvidence(Evidence evidence) throws RealmUnavailableException {
                    if (evidence instanceof PasswordGuessEvidence) {
                        PasswordGuessEvidence guess = (PasswordGuessEvidence) evidence;
                        try {
                            return myStorage.verifyPassword(principal.getName(), guess.getGuess());
                        } finally {
                            guess.destroy();
                        }
                    }
                    return false;
                }
{% endhighlight %}

Now, when we have successfuly implemented neccessary classes, we can build the JAR using `mvn package` command.

## Module descriptor

To be the realm available for the Elytron subsystem, we need to place it into WildFly module.
WildFly module is a directory in `modules` directory of your application server, where you place your JARs and a module descriptor describing them and defined classes will become available to the application server.

Lets say you have choosen path `my.package.myrealm` (it is recommanded to be corresponding with your Maven package) - you should place your JARs into `modules/my/package/myrealm/main` together with `module.xml` descriptor like this:

{% highlight xml %}
<?xml version='1.0' encoding='UTF-8'?>
<module xmlns="urn:jboss:module:1.1" name="my.package.myrealm">

    <resources>
        <resource-root path="myrealm-1.0-SNAPSHOT.jar"/>
    </resources>

    <dependencies>
        <module name="org.wildfly.security.elytron"/>
    </dependencies>

</module>
{% endhighlight %}

For more information about possible content of a module descriptor check [jboss-modules documentation](https://jboss-modules.github.io/jboss-modules/manual/#descriptors).

## Referencing from Elytron

When the module is on place, you can create `custom-security-realm` in WildFly CLI:

{% highlight text %}
/subsystem=elytron/custom-realm=MyRealm:add(module=my.package.myrealm, class-name=my.package.MyRealm)
{% endhighlight %}

In this moment you can start to use your realm like any other realm in WildFly, like `properties-realm` - for example start to use it as default security realm of `ApplicationDomain`:

{% highlight text %}
/subsystem=elytron/security-domain=ApplicationDomain:list-add(name=realms, index=0, value={realm=MyRealm3})
/subsystem=elytron/security-domain=ApplicationDomain:write-attribute(name=default-realm, value=MyRealm3)
reload
{% endhighlight %}

From now, if you has already configured your server to secure your applications using Elytron, your security realm should be used to authenticate requests to your applications.

## Custom component configuration

If you want to provide some configuration to your realm from `standalone.xml`, you need to implement method `initialize()` and interface `Configurable`. (In later versions of WildFly should be need of `Configurable` interface [removed](https://issues.jboss.org/browse/WFCORE-3877).)

{% highlight java %}
import org.wildfly.extension.elytron.Configurable;

public class MyRealm implements SecurityRealm, Configurable {

    private String myAttribute;

    // receiving configuration from subsystem
    public void initialize(Map<String, String> configuration) {
        myAttribute = configuration.get("myAttribute");
    }
{% endhighlight %}

To have interface `Configurable` available you need to add Maven dependency on Elytron subsystem:

{% highlight xml %}
        <dependency>
            <groupId>org.wildfly.core</groupId>
            <artifactId>wildfly-elytron-integration</artifactId>
            <version>4.0.0.Final</version>
        </dependency>
{% endhighlight %}

When this requirement is fullfiled, you can provide configuration while creating the realm:

{% highlight text %}
/subsystem=elytron/custom-realm=myRealm:add(
    module=my.package.myrealm,
    class-name=my.package.MyRealm
    configuration={ myAttribute="myValue", otherAttribute="test" })
{% endhighlight %}

After realm construction using non-parametrized constructor, method `initialize()` will be invoked with provided configuration.

For full example of custom security realm check [elytron-examples](https://github.com/wildfly-security-incubator/elytron-examples/tree/master/custom-realm).
