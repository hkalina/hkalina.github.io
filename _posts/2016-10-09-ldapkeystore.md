---
layout: post
title: LDAP KeyStore
---

The aim of this blog post is to provide information about new feature of Elytron, the LDAP KeyStore.
If you didn't know it yet, Elytron is new security framework of WildFly application server.

I suppose you have already heard about KeyStore, facility in Java which allows to store certificates and cryptographic keys in encrypted form.
KeyStores are traditionally used as storage of private keys for SSL secured services and also as truststore - storage of trusted certificates - of certification authorities (CA) or directly certificates of individual peers (clients for server or servers for client).

The traditional KeyStore is a one file somewhere in filesystem. It is very practical if you need to store one or few private keys or CA certificates, but in same cases we can meet the need to manage trusted certificates or cryptographical keys from one central place.

A de facto standard for central management are today LDAP based directory services. They are used not only as central directory of basic information about users and computers in the network. Solutions, like Red Hat Identity Management or Microsoft Active Directory, use LDAP (together with Kerberos) for authentication of users and for computers configuration.

The purpose of LDAP KeyStore is to allow using of any LDAP directory as certificates and keys storage, accessible through KeyStore API, indistinguishable from traditional file-backed KeyStore from client side of view.

## Certificates and keys in LDAP

The idea to store certificates and keys in LDAP is not new - already [RFC 2798](https://tools.ietf.org/html/rfc2798) from 2000 brings `inetOrgPerson` object class, which covers:

* `userCertificate` - the X.509 certificate of the user
* `userSMIMECertificate` - the S/MIME certificates chain encoded in PKCS#7
* `userPKCS12` - the PKCS12 archive of PFX PDUs (personal information exchange protocol data units) - certificates chain and private key packed in one password encrypted file

As you can see, this LDAP attributes covers all needs of Java KeyStore - they allow to store certificates, certificates chains and cryptographical keys. The only limitation is need of standalone LDAP entries for individual KeyStore aliases/items. The LDAP entry containing KeyStore item illustrates following image.

![KeyStore item stored in Apache Directory](/images/keystore.png "KeyStore item stored in Apache Directory")

## Using LDAP KeyStore in Elytron

Lets start using LDAP KeyStore in WildFly. At first we have to define connection to the LDAP server - the `dir-context`:

{% highlight bash %}
/subsystem=elytron/dir-context=Ldap1/:add(authentication-level=simple, url="ldap://localhost:11390/", principal="uid=server,dc=elytron,dc=wildfly,dc=org", credential="serverPassword")
{% endhighlight %}

In this example we have defined password protected but unencrypted connection to the LDAP server. Use this for testing purposes only - the SSL should be added for real use cases. Without it nor integrity will be ensured, so this is not good choice nor for a truststore (not to mention about keystore). Creating secured `dir-context` will be described in standalone blog post.

When is `dir-context` created, we can create the `ldap-key-store` itself. In most simple case you need to specify only name of `dir-context` and `search-path` where will be KeyStore aliases searched. Just note, if not specified differently, the search is not recursive - only direct subentries of specified path will be searched.

{% highlight bash %}
/subsystem=elytron/ldap-key-store=LKS1/:add(dir-context=Ldap1, search-path="ou=keystore,dc=elytron,dc=wildfly,dc=org")
{% endhighlight %}

If we was successful, we can read list of aliases in our LDAP-backed KeyStore:

{% highlight bash %}
/subsystem=elytron/ldap-key-store=LKS1/:read-children-names(child-type=alias)
{
    "outcome" => "success",
    "result" => ["firefly", "scarab"],
    "response-headers" => {"process-state" => "reload-required"}
}
{% endhighlight %}

Such a KeyStore can be used by the same way as classical file based KeyStore - for example, you can use it as truststore of SSLContext - create appropriate TrustManager and SSLContext using it:

{% highlight bash %}
/subsystem=elytron/trust-managers=MyTrustManager/:add(algorithm=SunX509, key-store=LKS1)
/subsystem=elytron/server-ssl-context=MySslContext/:add(key-managers=twoWayKM, trust-managers=MyTrustManager, need-client-auth=true, protocols=[TLSv1_1,TLSv1_2])
{% endhighlight %}

Then SSLContext can be used in Undertow's HTTPS listener to secure access to applications on the server:

{% highlight bash %}
/subsystem=undertow/server=default-server/https-listener=https/:write-attribute(name=ssl-context, value=MySslContext)
{% endhighlight %}

If you don't need to obtain users identity from an application, this configuration will be sufficient to protect access to your application using client SSL certificates stored in LDAP.

## Storing certificates in LDAP

If you have not SSL certificates in LDAP directory yet, you will following steps to get them in.
Certificates, certificates chains and potentially encrypted keys have to be converted into form specified above and encoded into Base64 to they can be passed into LDIF file, which can be imported into LDAP directory.

Lets suppose we have PEM certificate of user and we want to obtain LDIF to import it into LDAP entry. We need to convert it into DER format using `openssl` first and format it into LDIF Base64 attribute using `ldif` utility:

{% highlight bash %}
openssl x509 -in user.pem -outform DER -out /tmp/outcert.der
ldif -b "usercertificate" < /tmp/outcert.der
{% endhighlight %}

By the similar way we can convert this certificate together with CA certificate into PKCS#7 certificate chain:

{% highlight bash %}
openssl crl2pkcs7 -nocrl -certfile user.pem -certfile ca.pem -out /tmp/chain.p7b
ldif -b "userSMIMECertificate" < /tmp/chain.p7b
{% endhighlight %}

The private key can be stored in PKCS#12 in the directory - that mean the key will be encrypted using passphrase, which will never be send to the LDAP. Even through I would not recommand to store private keys on remote machine if you don't have to, as it is less secure in every case. In any case I recommand to use SSL secured LDAP connection.

If you have your private key in classical Java keystore now and you want to store it into LDAP, you have to export it into PKCS#12 file and convert it into LDIF by the same way as certificates:

{% highlight bash %}
keytool -importkeystore -srckeystore my.keystore -srcstoretype jks -destkeystore /tmp/export.p12 -deststoretype pkcs12
ldif -b "userPKCS12" < /tmp/export.p12
{% endhighlight %}

By this way you can put as much truststore or keystore entries into LDAP as you want.

Example of complete LDIF is available as [part of WildFly Elytron tests](https://github.com/wildfly-security/wildfly-elytron/blob/11d2aca181419deee792fefc9f16a7601c41da7d/src/test/resources/ldap/elytron-keystore-tests.ldif).

