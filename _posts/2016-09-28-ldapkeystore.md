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





{% highlight bash %}
/subsystem=elytron/dir-context=LocalLDAPanon/:add(authentication-level=none)
/subsystem=elytron/dir-context=LocalLDAP/:add(authentication-level=none,credential="serverPassword",principal="uid=server,dc=elytron,dc=wildfly,dc=org",referral-mode=IGNORE)
/subsystem=elytron/ldap-key-store=TestingLdapKeyStore/:add(dir-context=LocalLDAPanon,search-path="ou=keystore,dc=elytron,dc=wildfly,dc=org",search-recursive=true)
{% endhighlight %}


## Storing certificates in LDAP




