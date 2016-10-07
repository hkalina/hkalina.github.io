---
layout: post
title: LDAP KeyStore
---

The aim of this blog post is to provide information about new feature of Elytron, new security framework of WildFly application server, the LDAP KeyStore.
I suppose you have already heard that KeyStore is facility in Java, which allow to store certificates and cryptographic keys in encrypted form.
KeyStores are traditionally used as storage of private keys for SSL secured connections and also as truststore - storage of trusted certificates.
The traditional KeyStore is a one file somewhere in filesystem. It is very practical if you need to store one of few private key or a few CA certificates, but in same cases we can meet the need to allow to manage list of trusted certificates or used cryptographical keys from one central place.

De facto standard for central management are today LDAP based directory services. They are used not only as central directory of basic information about users and computers in the network. Solutions like Red Hat Identity Management or Microsoft Active Directory use LDAP (together with Kerberos) for authentication of users and computers configuration too.

The purpose of LDAP KeyStore is to allow using of any LDAP directory as certificates and keys storage, accesible through KeyStore API, undistinguishable from traditional file-backed KeyStore.

## Certificates and keys in LDAP

The idea to store certificates and keys in LDAP is not new - already [RFC 2798](https://tools.ietf.org/html/rfc2798) from 2000 brings `inetOrgPerson` object class, which covers:

* `userCertificate` - the X.509 certificate of the user
* `userSMIMECertificate` - the S/MIME certificates chain encoded in PKCS#7
* `userPKCS12` - the PKCS12 archive of PFX PDUs (personal information exchange protocol data units) - certificates chain and private key packed in password encrypted archive

As you can see, this LDAP attributes covers all needs of Java KeyStore - they allow to store certificates, certificates chains and cryptographical keys. The only limitation is need of standalone LDAP entries for individual KeyStore aliases/items. The LDAP entry containing KeyStore item illustrates following image.

![KeyStore item stored in Apache Directory](/images/keystore.png "KeyStore item stored in Apache Directory")

## Using LDAP KeyStore in Elytron

Lets start using LDAP KeyStore in WildFly. At first we have to define connection to the LDAP server - `dir-context`:

{% highlight bash %}
/subsystem=elytron/dir-context=Ldap1/:add(authentication-level=simple, url="ldap://localhost:11390/", principal="uid=server,dc=elytron,dc=wildfly,dc=org", credential="serverPassword")
{% endhighlight %}

In this example we have defined password protected but unencrypted connection to the LDAP server. Use this for testing puposes only - the SSL should be added for real use cases. Without it nor integrity will be checked, so this is not good chooise nor for a truststore. Creating secured `dir-context` will be described in standalone blog post.

When is `dir-context` created, we can create the `ldap-key-store` itself. In most simple case you need to specify only name of `dir-context` and `search-path` where will be KeyStore aliases searched. Just note, if not specified differently, the search is not recursive - only direct subentries of specified entry will be searched.

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

Such a KeyStore can be now used by the same way as classical file based KeyStore - for example, you can use it as truststore of SSLContext - create appropriate TrustManager and SSLContext using it:

{% highlight bash %}
/subsystem=elytron/trust-managers=MyTrustManager/:add(algorithm=SunX509, key-store=LKS1)
/subsystem=elytron/server-ssl-context=MySslContext/:add(key-managers=twoWayKM, trust-managers=MyTrustManager, need-client-auth=true, protocols=[TLSv1_1,TLSv1_2])
{% endhighlight %}

Then SSLContext can be used in Undertow's HTTPS listener to secure access to web:

{% highlight bash %}
/subsystem=undertow/server=default-server/https-listener=https/:write-attribute(name=ssl-context, value=MySslContext)
{% endhighlight %}

If you don't need to obtain users identity from an application, this configuration will be sufficient to protect access to your application using client SSL certificates stored in LDAP.

## Storing certificates in LDAP

If you don't store SSL certificates in LDAP directory yet, passing them in can be a bit complicated.
Certificates, certificates chains and potentionally encrypted keys have to be converted into form specified above and encoded into Base64 before they can be passed into LDIF file, which can be imported into LDAP directory.

Lets suppose we have PEM certificate of user and we want to obtain LDIF importable into his LDAP entry. We need to convert it into DER format using `openssl` and place it into LDIF in Base64 using `ldif` utility:

{% highlight bash %}
openssl x509 -in user.pem -outform DER -out /tmp/outcert.der
ldif -b "usercertificate" < /tmp/outcert.der
{% endhighlight %}

By the similar way we can convert more certificates into PKCS#7 certificate chain:

{% highlight bash %}
openssl crl2pkcs7 -nocrl -certfile user.pem -certfile ca.pem -out /tmp/chain.p7b
ldif -b "userSMIMECertificate" < /tmp/chain.p7b
{% endhighlight %}

The private key can be stored in PKCS#12 in the directory - that mean the key will be encrypted using passphrase, which will be never send to the LDAP. Even through I would not recommand to store private keys on remote machine if you don't have to.

If you have your private key in classical Java keystore now and you want to store them in the LDAP, you have to export it into PKCS#12 file and convert it into LDIF by the same way as certificates:

{% highlight bash %}
keytool -importkeystore -srckeystore my.keystore -srcstoretype jks -destkeystore /tmp/export.p12 -deststoretype pkcs12
ldif -b "userPKCS12" < /tmp/export.p12
{% endhighlight %}

Example of complete LDIF is available as [part of WildFly Elytron tests](https://github.com/wildfly-security/wildfly-elytron/blob/11d2aca181419deee792fefc9f16a7601c41da7d/src/test/resources/ldap/elytron-keystore-tests.ldif).


