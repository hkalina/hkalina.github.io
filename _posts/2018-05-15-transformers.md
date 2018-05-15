---
layout: post
title: Testing WildFly subsystem transformers
---

Transformers in WildFly allows replication of configuration from newer domain controller in a WildFly servers domain to an older (non-upgraded yet) slave server.
This article describes basic examples of creating transformers and their testing.

Transformers for one subsystem can look like following example. They should be defined in class implementing `ExtensionTransformerRegistration`.

The first few lines defines ModelVersion object for individual versions of model.
This should correlate with version of XSD for XML configuration files.

The registering method defines individual transformers for individual transitions.
No need to define transformers from version 3 to version 1 - it is sufficient to have defined transitions from 3 to 2 and from 2 to 1.
Transformations do always downgrading only - using newer slave server with older domain controller is not supported by WildFly.

{% highlight java %}
public final class ElytronSubsystemTransformers implements ExtensionTransformerRegistration {
    private static final ModelVersion ELYTRON_1_2_0 = ModelVersion.create(1, 2); // oldest version of subsystem model
    private static final ModelVersion ELYTRON_2_0_0 = ModelVersion.create(2, 0);
    private static final ModelVersion ELYTRON_3_0_0 = ModelVersion.create(3, 0); // latest version of subsystem model

    @Override
    public String getSubsystemName() {
        return ElytronExtension.SUBSYSTEM_NAME;
    }

    @Override
    public void registerTransformers(SubsystemTransformerRegistration registration) {
        ChainedTransformationDescriptionBuilder chainedBuilder = TransformationDescriptionBuilder.Factory.createChainedSubystemInstance(registration.getCurrentSubsystemVersion());

        // 3.0.0 to 2.0.0 (new attribute should be dropped if it is not set, otherwise the transformation should be rejected)
        ResourceTransformationDescriptionBuilder builder3_0_0To2_0_0 = chainedBuilder.createBuilder(ELYTRON_3_0_0, ELYTRON_2_0_0);
        builderCurrentTo2_0_0
                .addChildResource(PathElement.pathElement("file-audit-log")) // it is attribute of /subsystem=elytron/file-audit-log=*
                .getAttributeBuilder()
                    // attribute can be discarted if it is undefined
                    .setDiscard(DiscardAttributeChecker.UNDEFINED, "flushed")
                    // in all other cases the transformation should be rejected
                    .addRejectCheck(RejectAttributeChecker.ALL, "flushed")
                    .end();

        // 2.0.0 to 1.2.0 (no real changes)
        chainedBuilder.createBuilder(ELYTRON_2_0_0, ELYTRON_1_2_0);
        chainedBuilder.buildAndRegister(registration, new ModelVersion[] { ELYTRON_2_0_0, ELYTRON_1_2_0 });

    }
}
{% endhighlight %}

In this example the transition from 2.0.0 to 1.2.0 is blank - no changes requiring transformers was done in the model.

In the version 3.0.0 was on the other had defined new attribute "flushed". This attribute, when set, change behavior of `file-audit-log` by way, which cannot be archieved in older version. (Or it would be too complicated to ensure it using transformer.) This meand we need to set two transformations:
* When the attribute is undefined, it should be discarded
* When the attribute is used, the transformation has to be rejected

How to archieve it you can see in the example above.

## Testing

WildFly core provides testing framework for testing proper behavior of transformers. It is part of [wildfly-subsystem-test-framework](https://github.com/wildfly/wildfly-core/tree/master/subsystem-test/framework).

### Simple transformers testing test

To create simple transformers test case you need to extend AbstractSubsystemBaseTest:

{% highlight java %}
public class MyTransformersTestCase extends AbstractSubsystemBaseTest {
    public MyTransformersTestCase() {
        super("mysubsystem", new MySubsystemExtension());
    }
    @Override
    protected String getSubsystemXml() throws IOException {
        return readResource("testing.xml");
    }
}
{% endhighlight %}

Now you can add transformation test. The test will load a testing XML file into latest subsystem and will try to initialize legacy subsystem from it.

At first, lets specify *target version* of transformation. The frameworks requires to specify version of the controller using ModelTestControllerVersion enum and version of the model of the tested subsystem (ModelVersion as specified in the transformers). In following example we will transform to the subsystem model version default to the given controller version:

{% highlight java %}
ModelTestControllerVersion controller = ModelTestControllerVersion.EAP_7_1_0;
ModelVersion subsystemModelVersion = controller.getSubsystemModelVersion("elytron");
{% endhighlight %}

The AbstractSubsystemBaseTest provides a method to obtain KernelServicesBuilder allowing to boot up controller for latest version of the subsystem:

{% highlight java %}
KernelServicesBuilder builder = createKernelServicesBuilder(AdditionalInitialization.MANAGEMENT);
{% endhighlight %}

The builder will be used to boot up the old version of controller too. Take a note the old subsystem is downloaded from Maven repository:

{% highlight java %}
builder.createLegacyKernelServicesBuilder(AdditionalInitialization.MANAGEMENT, controller, subsystemModelVersion)
       .addMavenResourceURL("org.wildfly.core:wildfly-elytron-integration:4.0.0.Final");
{% endhighlight %}

Now you can finish the builder and verify that both controllers has booted up successfully:

{% highlight java %}
KernelServices services = builder.build();
assertTrue(services.isSuccessfulBoot());
assertTrue(services.getLegacyServices(subsystemModelVersion).isSuccessfulBoot());
{% endhighlight %}

The model parsed by latest controller from XML provided by{{getSubsystemXml()}} is now already transformed and used by the legacy controller. The framework provides method to check that both models are the same and valid:

{% highlight java %}
checkSubsystemModelTransformation(services, subsystemModelVersion, null, true);
{% endhighlight %}

### Testing of rejecting rules

Part of the transformers are also rejecting rules, which needs to be tested too. Lets image we have a new attribute {{"flushed"}} which cannot be part of the legacy model. To test it will be rejected, we need to write and XML configuration containing it, to parse it and to try to transform it:

{% highlight java %}
List<ModelNode> operations = builder.parseXml(
    "<subsystem xmlns=\"urn:wildfly:mysubsystem:3.0\">\n" +
    "    <my-resource name=\"test\" flushed=\"true\"/>\n" +
    "</subsystem>");

ModelTestUtils.checkFailedTransformedBootOperations(services, subsystemModelVersion, operations, new FailedOperationTransformationConfig()
    .addFailedAttribute(myResourceTestPathAddress,
        new FailedOperationTransformationConfig.NewAttributesConfig("flushed"))
);
{% endhighlight %}

Util of testing subsystem will check that ALL listed attributes (and nothing else) will be rejected.

Be aware the test framework use FailedOperationTransformationConfig to fix the model to continue checking following failures.
The `NewAttributesConfig` will ensure the attribute `flushed` will be undefined after its failing.
If you are checking rejecting subattribute of complex attribute, the whole attribute will be undefined after its rejecting to continue evaluation. But that can cause operation failure if the complex attribute was required. You need to use `REJECTED_RESOURCE` config in such case - the whole resource adding operation will be removed before continuing the evaluation.

### Full example

{% highlight java %}
public class SubsystemTransformerTestCase extends AbstractSubsystemBaseTest {

    public SubsystemTransformerTestCase() {
        super(ElytronExtension.SUBSYSTEM_NAME, new ElytronExtension());
    }

    @Override
    protected String getSubsystemXml() throws IOException {
        return readResource("elytron-transformers-3.0.xml");
    }

    @Test
    public void testTransformationToEAP710() throws Exception {
        ModelTestControllerVersion controller = EAP_7_1_0; // target version
        ModelVersion elytronVersion = controller.getSubsystemModelVersion(getMainSubsystemName()); // target model version

        KernelServicesBuilder builder = createKernelServicesBuilder(AdditionalInitialization.MANAGEMENT)
                .setSubsystemXml(getSubsystemXml());

        builder.createLegacyKernelServicesBuilder(AdditionalInitialization.MANAGEMENT, controller, elytronVersion)
                .addMavenResourceURL(controller.getCoreMavenGroupId() + ":wildfly-elytron-integration:" + controller.getCoreVersion())
                .skipReverseControllerCheck()
                .dontPersistXml();

        KernelServices services = builder.build();
        Assert.assertTrue(ModelTestControllerVersion.MASTER + " boot failed", services.isSuccessfulBoot());
        Assert.assertTrue(controller.getMavenGavVersion() + " boot failed", services.getLegacyServices(elytronVersion).isSuccessfulBoot());

        // check that both versions of the legacy model are the same and valid
        checkSubsystemModelTransformation(services, elytronVersion, null, true);

        ModelNode transformed = services.readTransformedModel(elytronVersion);
        Assert.assertTrue(transformed.isDefined());
    }

    @Test
    public void testRejectingFileAuditLogFlushed() throws Exception {
        ModelTestControllerVersion controller = EAP_7_1_0; // target version
        ModelVersion elytronVersion = controller.getSubsystemModelVersion(getMainSubsystemName()); // target model version

        // Boot up empty controllers with the resources needed for the ops coming from the xml to work
        KernelServicesBuilder builder = createKernelServicesBuilder(AdditionalInitialization.MANAGEMENT);
        builder.createLegacyKernelServicesBuilder(AdditionalInitialization.MANAGEMENT, controller, elytronVersion)
                .addMavenResourceURL(controller.getCoreMavenGroupId() + ":wildfly-elytron-integration:" + controller.getCoreVersion())
                .dontPersistXml();

        KernelServices services = builder.build();
        assertTrue(services.isSuccessfulBoot());
        assertTrue(services.getLegacyServices(elytronVersion).isSuccessfulBoot());

        List<ModelNode> ops = builder.parseXml("<subsystem xmlns=\"urn:wildfly:elytron:3.0\">\n" +
                "    <audit-logging>\n" +
                "        <file-audit-log name=\"audit\" path=\"audit.log\" synchronized=\"false\" flushed=\"true\"/>\n" +
                "    </audit-logging>\n" +
                "</subsystem>");
        PathAddress subsystemAddress = PathAddress.pathAddress(ModelDescriptionConstants.SUBSYSTEM, ElytronExtension.SUBSYSTEM_NAME);
        ModelTestUtils.checkFailedTransformedBootOperations(services, elytronVersion, ops, new FailedOperationTransformationConfig()
                .addFailedAttribute(subsystemAddress.append(PathElement.pathElement(ElytronDescriptionConstants.FILE_AUDIT_LOG)),
                        new FailedOperationTransformationConfig.NewAttributesConfig(AuditResourceDefinitions.FLUSHED))
        );
    }
}
{% endhighlight %}
