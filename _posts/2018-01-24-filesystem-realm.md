---
layout: post
title: Filesystem realm in WildFly Elytron
---

Elytron provides Filesystem-Based Identity Store called `filesystem-realm`, which stores user identities in chosen directory of the filesystem.
In its principle it is similar to `properties-realm`, but it is designed to store much bigger amount of identities and it provides management operations for handling them.

## Creating realm

To create `filesystem-realm` it is sufficient to specify directory, where the identities should be stored:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:add(path="realms/MyRealm", relative-to=jboss.server.config.dir)
{% endhighlight %}

Realm basically stores individual identities in standalone XML files in the realm directory. Identities names are basically used as name of given identity file.
But because of limitations of some filesystems, there are two modificators allowing to workaround this limitations:

* `levels` - Some filesystems has limited amount of files stored directly in one directory. When this attribute is set to non-zero value, realm stores identities in directory structure by first characters of identity name, so it is possible to store practically unlimited amount of identities.
* `encoded` - Some filesystems are case-insensitive or restrict set of characters allowed in filenames. When this attribute is set to `true`, BASE32 is used to encode username into filename, instead of direct using of the username as the filename.

## Creating identity

In comparison to properties realm, filesystem realm allows to manage identities through management operations and this changes will take effect immediately. (In comparison to properties realm, which requires server reload.)

To add new identity:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:add-identity(identity=user1)
{% endhighlight %}

To set password of the identity:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:set-password(identity=user1, \
    clear={password=password1})
{% endhighlight %}

This will store unhashed password into the identity file. If you want to store hashed password, you need to use appropriate attribute:

To store password for `DIGEST-SHA` authentication mechanism and mechanism realm `Secured Kingdom` use:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:set-password(identity=user1, \
    digest={algorithm=digest-sha, realm=Secured Kingdom, password=password1})
{% endhighlight %}

Need to note, `set-password` operation will override all existing passwords of the user - if you want to be able to log-in using multiple mechanisms, you need to set all passwords types at once:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:set-password(identity=user1, \
    digest={algorithm=digest-md5, realm=Secured Kingdom, password=password1}, \
    simple-digest={algorithm=simple-digest-sha-256, password=password1})
{% endhighlight %}

# Identity attributes

Identity attributes are typically used to store identity roles, real name etc.

To set identity attribute use:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:add-identity-attribute(identity=user1, name=role, value=[ADMIN, MANAGER])
/subsystem=elytron/filesystem-realm=MyRealm:read-identity(identity=user1)
{
    "outcome" => "success",
    "result" => {
        "name" => "user1",
        "attributes" => {"role" => [
            "ADMIN",
            "MANAGER"
        ]}
    }
}
{% endhighlight %}

To remove individual values of the attribute:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:remove-identity-attribute(identity=user1, name=role, value=[MANAGER])
/subsystem=elytron/filesystem-realm=MyRealm:read-identity(identity=user1)
{
    "outcome" => "success",
    "result" => {
        "name" => "user1",
        "attributes" => {"role" => ["ADMIN"]}
    }
}
{% endhighlight %}

To remove whole attribute:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:remove-identity-attribute(identity=user1, name=role)
/subsystem=elytron/filesystem-realm=MyRealm:read-identity(identity=user1)
{
    "outcome" => "success",
    "result" => {
        "name" => "user1",
        "attributes" => undefined
    }
}
{% endhighlight %}

## Removing identity

To remove identity from the realm use:

{% highlight text %}
/subsystem=elytron/filesystem-realm=MyRealm:remove-identity(identity=user1)
{% endhighlight %}

