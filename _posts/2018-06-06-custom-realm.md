---
layout: post
title: Creating custom security realm for WildFly Elytron
---

This tutorial describes creating custom security realm and its usage in WildFly Elytron subsystem.

To create custom Elytron component you need to create WildFly module containing JAR with class extending interface `SecurityRealm` available from maven in package `wildfly-elytron`.
This JAR will have to be placed into `modules` directory of your application server together with module descriptor `module.xml`.

## Building the JAR

For simplification will this tutorial assume you use Maven to build your security realm JAR.
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

Java sources will be expected in `src/main/java` subdirectory and the built can be done using `mvn package` command.

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

Our security realm will, on the other hand, support password verification - when clear password will be given, the realm will be able to say whether it is correct or not. But only for identities it knows - and we are asked for status for any identity - because of that we will return `POSSIBLY_SUPPORTED`:

{% highlight java %}
    public SupportLevel getEvidenceVerifySupport(Class<? extends Evidence> evidenceType, String algorithmName)
            throws RealmUnavailableException {
        return PasswordGuessEvidence.class.isAssignableFrom(evidenceType) ? SupportLevel.POSSIBLY_SUPPORTED : SupportLevel.UNSUPPORTED;
    }
{% endhighlight %}

Finally, the main method of security realm will allow to obtain object of an identity.
If the identity for given principal does not exists, we can return predefined `NON_EXISTENT` identity.
Otherwise we need to return object of our own identity class:

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

But we will support verification of clear passwords (`PasswordGuessEvidence`s). This time we already know the user exists, so we will return `SUPPORTED`.

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

To be the realm available for the Elytron subsystem we need to place it into WildFly module.
WildFly module is a directory in `modules` directory of your application server, where are placed your JARs and a module descriptor describing them.

Lets say you have choosen path `my.package.myrealm` (it is recommanded to correspond with your Maven package) - you should place your JARs into `modules/my/package/myrealm/main` together with `module.xml` descriptor like this:

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

When the module is on place, you can create `custom-security-realm` in the WildFly CLI:

{% highlight text %}
/subsystem=elytron/custom-realm=myRealm:add(module=my.package.myrealm, class-name=my.package.MyRealm)
{% endhighlight %}

