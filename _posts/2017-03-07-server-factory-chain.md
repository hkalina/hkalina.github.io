---
layout: post
title: Server factories chains in Elytron
---

WildFly Elytron use one special way to configure `SaslServerFactories` and `HttpServerFactories`.
When you will try to debug them, you will find indefinitely long chaing of delegating server factories.
Because it is hard and exhaustive to find point, where is this chain constructed, I will describe it in this post for my future me, or for anybody who could need it.

The most of magic happens in SaslServerDefinition/HttpServerDefinition in elytron-subsystem in block like this:

{% highlight java %}
TrivialService<SaslServerFactory> saslServiceFactoryService = new TrivialService<SaslServerFactory>(() -> {
    SaslServerFactory theFactory = saslServerFactoryInjector.getValue();
    theFactory = new SetMechanismInformationSaslServerFactory(theFactory);
    theFactory = protocol != null ? new ProtocolSaslServerFactory(theFactory, protocol) : theFactory;
    theFactory = serverName != null ? new ServerNameSaslServerFactory(theFactory, serverName) : theFactory;
    theFactory = propertiesMap != null ? new PropertiesSaslServerFactory(theFactory, propertiesMap) : theFactory;
    theFactory = finalFilter != null ? new FilterMechanismSaslServerFactory(theFactory, finalFilter) : theFactory;
    return theFactory;
});
{% endhighlight %}

The original server factory is obtained from parent, which is obtained by name specified in `sasl-server-factory` attribute of given element in XML config.

Most of delegators is created in special, for this purpose created, `configurable-sasl-server-factory`/`configurable-http-server-factory`.
If you will need to set, or check if is set, same part of created SASL/HTTP servers, you will probably find it here.
But `configurable-*-server-factory` creates delegators servers only. If you are looking for original source of servers, look for `provider-*-server-factory` and `service-loader-sasl-server-factory` ;)

The delegating server factories are very simple: (ok, simplified, look into [source](https://github.com/wildfly-security/wildfly-elytron/tree/master/src/main/java/org/wildfly/security/sasl/util) for full detail)

{% highlight java %}
public final class SetSomethingSaslServerFactory implements SaslServerFactory {
    protected final String something;
    protected final SaslServerFactory delegate;
    
    public String getSomething() {
        return this.something; // replace output from parent by own value
    }
    
    public String getSomethingElse() {
        return delegate.getSomethingElse(); // just delegate to parent
    }
}
{% endhighlight %}

As you can see, it is really very simple - you will see how much it is annoying when you will be trying to debug it and will not be able to find end of the delegating chain ;)

A bit different is special delegating server factory `SetMechanismInformationSaslServerFactory` - it takes parameters from children, which delegate to it and send this paremeters back to the top caller using special `MechanismInformationCallback` - callback which provides all this params to callback handler, which was at the beginning set by the top caller! How easy... well it is not, but if you have a tip how to implement such feature without robustness degradation...

