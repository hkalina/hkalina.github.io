---
layout: post
published: false
title: WildFly subsystem development
---

* ATTRIBUTE.resolveModelAttribute(context, model).asString() fills to default value
* wildfly service dependency injecting
* allowNull for ObjectTypeAttributeDefinition.Builder; SimpleAttributeDefinitionBuilder; ListAttributeBuilder



Inheritance:

    static final SimpleAttributeDefinition PATH = new SimpleAttributeDefinitionBuilder(ElytronDescriptionConstants.PATH, ModelType.STRING, false)
        .setAllowExpression(true)
        .setMinSize(1)
        .setAttributeGroup(ElytronDescriptionConstants.FILE)
        .setFlags(AttributeAccess.Flag.RESTART_RESOURCE_SERVICES)
        .build();

    static final SimpleAttributeDefinition PATH =
            new SimpleAttributeDefinitionBuilder(ElytronDescriptionConstants.PATH, FileAttributeDefinitions.PATH)
                    .setAttributeGroup(ElytronDescriptionConstants.FILE)
                    .setAllowNull(false)
                    .build();



