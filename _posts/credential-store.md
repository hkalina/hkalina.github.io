---
layout: post
published: false
title: Credential stores
---

How to create:

/subsystem=elytron/credential-store=testCS:add(uri="cr-store://cred-store-default/cred-store.jceks?keyStoreType=JKS;modifiable=true;create=true", relative-to=jboss.server.config.dir, credential-reference={clear-text=password})

/subsystem=elytron/credential-store=cs007:add(uri="cr-store://test/folderNotExist/keystorecs007.jceks?create=true", credential-reference={clear-text=pass123})


New alias:

/subsystem=elytron/credential-store=testCS/alias=aliasEntryType:add(secret-value=secretVALUE, entry-type=org.wildfly.security.credential.PasswordCredential)



/subsystem=elytron/credential-store=cs007:add(uri="cr-store://test/keystorecs007.jceks?create=true", credential-reference= {clear-text=pass123})
/subsystem=elytron/credential-store=cs007/alias=newCs007:add(secret-value=Elytron)

Now you can see in EAP_FOLDER (it can be different, see above) keystorecs007.jceks file
