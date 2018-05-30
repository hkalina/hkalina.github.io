---
layout: post
title: Certificate authentication with password fallback in Elytron
---

This tutorial describes configuration of certificate authentication with password (BASIC/PLAIN) fallback authentication for management interface of WildFly using WildFly Elytron.

## Prepare LDAP directory

To login using certificate, LDAP entry needs to be resolvable using information in certificate.
In this example we will use certificate with subject `OU=Elytron, O=Elytron, C=UK, ST=Elytron, CN=Firefly` - we will be extracting the `CN` part to obtain name to obtain the identity from the LDAP.
The username will be in this case `Firefly`.
For simplicity we will not define new object class in this example, but we will use attribute `pager` to store serial number of the certificate.
The serial number of testing certificate is `01`.

{% highlight text %}
dn: cn=Firefly,dc=example,dc=com
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: person
objectClass: top
cn: Firefly
sn: admin
pager: 01
userPassword: secret
{% endhighlight %}

## Configure LDAP realm

Define connection to the LDAP server: (maybe you will need to specify `principal` and `credential-reference` too, if your LDAP server requires login)

{% highlight text %}
/subsystem=elytron/dir-context=ldap:add(url="ldap://localhost:10389")
{% endhighlight %}

Define LDAP realm with appropriate `x509-credential-mapper` (for certificates verification) and `user-password-mapper` (for strong password authentication - requires permission to read password hashes from the LDAP) OR `direct-verification` (for plain passsword authentication without receiving passwords from the LDAP):

{% highlight text %}
/subsystem=elytron/ldap-realm=ldapRealm:add(dir-context=ldap, identity-mapping={ \
	search-base-dn="dc=example,dc=com", filter-name="cn={0}", rdn-identifier="cn", \
	user-password-mapper={from="userPassword"}, \
	x509-credential-mapper={serial-number-from="pager"} \
}, direct-verification=true)
{% endhighlight %}

Resulting XML:

{% highlight xml %}
<security-realms>
    <ldap-realm name="ldapRealm" dir-context="ldap" direct-verification="true">
        <identity-mapping rdn-identifier="cn" search-base-dn="dc=example,dc=com" filter-name="cn={0}">
            <user-password-mapper from="userPassword"/>
            <x509-credential-mapper serial-number-from="pager"/>
        </identity-mapping>
    </ldap-realm>
</security-realms>
<dir-contexts>
    <dir-context name="ldap" url="ldap://localhost:10389"/>
</dir-contexts>
{% endhighlight %}

To check configuration try to obtain testing identity:

{% highlight text %}
/subsystem=elytron/ldap-realm=ldapRealm:read-identity(identity=firefly)
    "outcome" => "success",
    "result" => {
        "name" => "Firefly",
        "attributes" => undefined
    }
}
{% endhighlight %}

## Configure security domain for LDAP realm

Configure `ManagementDomain` to use one security realm - `ldapRealm`.

{% highlight text %}
batch
/subsystem=elytron/security-domain=ManagementDomain:list-remove(name=realms, index=0)
/subsystem=elytron/security-domain=ManagementDomain:list-add(name=realms, index=0, value={realm=ldapRealm})
/subsystem=elytron/security-domain=ManagementDomain:write-attribute(name=default-realm, value=ldapRealm)
run-batch
{% endhighlight %}

## Configure principal decoder

To be able to obtain identity from security realm by used certificate, you need to define `x500-attribute-principal-decoder` which will decode identity name from it:

{% highlight text %}
/subsystem=elytron/x500-attribute-principal-decoder=x500decoder:add(attribute-name=cn)
/subsystem=elytron/security-domain=ManagementDomain:write-attribute(name=principal-decoder, value=x500decoder)
{% endhighlight %}

In this example attribute `cn` from the subject of the certificate will be used.

## Enable Elytron for management interface

Configure management interface to use elytron for both, HTTP and SASL authentication:

{% highlight text %}
/core-service=management/management-interface=http-interface:write-attribute(name=http-authentication-factory,value=management-http-authentication)
/core-service=management/management-interface=http-interface:write-attribute(name=http-upgrade.sasl-authentication-factory, value=management-sasl-authentication)
/core-service=management/management-interface=http-interface:undefine-attribute(name=security-realm)
{% endhighlight %}

## Configure HTTPS for management interface

Configure SSL context for optional two-way authentication as described in [CLIENT-CERT authentication with Elytron](/2017/10/18/client-cert/):

{% highlight text %}
/subsystem=elytron/key-store=httpsKS:add(path="localhost.keystore", relative-to=jboss.server.config.dir, credential-reference={clear-text=Elytron}, type=JKS, required=true)
/subsystem=elytron/key-store=httpsTS:add(path="ca.truststore", relative-to=jboss.server.config.dir, credential-reference={clear-text=Elytron}, type=JKS, required=true)
/subsystem=elytron/key-manager=httpsKM:add(key-store=httpsKS,credential-reference={clear-text=Elytron})
/subsystem=elytron/trust-manager=httpsTM:add(key-store=httpsTS)
/subsystem=elytron/server-ssl-context=httpsSSC:add(key-manager=httpsKM, trust-manager=httpsTM, protocols=["TLSv1.1","TLSv1.2"], want-client-auth=true)
{% endhighlight %}

Resulting XML:

{% highlight xml %}
<tls>
    <key-stores>
        <key-store name="httpsKS">
            <credential-reference clear-text="Elytron"/>
            <implementation type="JKS"/>
            <file required="true" path="localhost.keystore" relative-to="jboss.server.config.dir"/>
        </key-store>
        <key-store name="httpsTS">
            <credential-reference clear-text="Elytron"/>
            <implementation type="JKS"/>
            <file required="true" path="ca.truststore" relative-to="jboss.server.config.dir"/>
        </key-store>
    </key-stores>
    <key-managers>
        <key-manager name="httpsKM" key-store="httpsKS">
            <credential-reference clear-text="Elytron"/>
        </key-manager>
    </key-managers>
    <trust-managers>
        <trust-manager name="httpsTM" key-store="httpsTS"/>
    </trust-managers>
    <server-ssl-contexts>
        <server-ssl-context name="httpsSSC" want-client-auth="true" protocols="TLSv1.1 TLSv1.2" key-manager="httpsKM" trust-manager="httpsTM"/>
    </server-ssl-contexts>
</tls>
{% endhighlight %}

Start using created SSL context for management interface:

{% highlight text %}
/core-service=management/management-interface=http-interface:write-attribute(name=ssl-context, value=httpsSSC)
/core-service=management/management-interface=http-interface:write-attribute(name=secure-socket-binding, value=management-https)
{% endhighlight %}

## Configure CLIENT_CERT and BASIC in HTTP authentication factory

The authentication of user accesing web console is secured by HTTP authentication factory.
Configure it to try to use CLIENT_CERT at first and, if it fails, to use BASIC as fallback.

{% highlight text %}
/subsystem=elytron/http-authentication-factory=management-http-authentication:list-add(name=mechanism-configurations, index=0, value={mechanism-name=CLIENT_CERT})
/subsystem=elytron/http-authentication-factory=management-http-authentication:list-add(name=mechanism-configurations, index=1, value={mechanism-name=BASIC})
/subsystem=elytron/http-authentication-factory=management-http-authentication:list-remove(name=mechanism-configurations, index=2)
{% endhighlight %}

XML:

{% highlight xml %}
<http-authentication-factory name="management-http-authentication" security-domain="ManagementDomain" http-server-mechanism-factory="global">
    <mechanism-configuration>
        <mechanism mechanism-name="CLIENT_CERT"/>
        <mechanism mechanism-name="BASIC"/>
    </mechanism-configuration>
</http-authentication-factory>
{% endhighlight %}

## Configure EXTERNAL and PLAIN in SASL authentication factory

{% highlight text %}
/subsystem=elytron/sasl-authentication-factory=management-sasl-authentication:list-add(name=mechanism-configurations, index=1, value={mechanism-name=EXTERNAL})
/subsystem=elytron/sasl-authentication-factory=management-sasl-authentication:list-add(name=mechanism-configurations, index=2, value={mechanism-name=PLAIN})
/subsystem=elytron/sasl-authentication-factory=management-sasl-authentication:list-remove(name=mechanism-configurations, index=3)
{% endhighlight %}

XML:

{% highlight xml %}
<sasl-authentication-factory name="management-sasl-authentication" sasl-server-factory="configured" security-domain="ManagementDomain">
    <mechanism-configuration>
        <mechanism mechanism-name="JBOSS-LOCAL-USER" realm-mapper="local"/>
        <mechanism mechanism-name="EXTERNAL"/>
        <mechanism mechanism-name="PLAIN"/>
    </mechanism-configuration>
</sasl-authentication-factory>
{% endhighlight %}

## Troubleshotting

For troubleshotting you may want to enable debug logging of Elytron security:

{% highlight text %}
/subsystem=logging/logger=org.wildfly.security:add(level=TRACE)
{% endhighlight %}

## Testing

### Web console (HTTP auth)

Try to access management console over HTTPS:

[https://localhost:9993/console/]

Web browser should ask for client SSL certifice first, BASIC login dialog should be shown if certificate authentication fails.

### Password auth via CLI

Try to connect using `jboss-cli` with password authentication:

{% highlight text %}
bin/jboss-cli.sh --no-local-auth
[disconnected  /] connect https-remoting://localhost
Username: Firefly
Password: secret
[standalone@localhost:9993  /]
{% endhighlight %}

### Certificate auth via CLI

Try to connect using `jboss-cli` with SSL certificate:

You will need to add the server into `bin/jboss-cli.xml`:

{% highlight xml %}
<ssl>
  <alias>firefly</alias>
  <key-store>../firefly.keystore</key-store>
  <key-store-password>Elytron</key-store-password>
  <trust-store>../ca.truststore</trust-store>
  <trust-store-password>Elytron</trust-store-password>
</ssl>
{% endhighlight %}

The paths are relative to working directory (from where you are starting `jboss-cli.sh`).

Now you can connect:

{% highlight text %}
bin/jboss-cli.sh --no-local-auth
[disconnected  /] connect https-remoting://localhost
[standalone@localhost:9993  /]
{% endhighlight %}

If the SSL authentication fail, the client will request PLAIN credentials.

