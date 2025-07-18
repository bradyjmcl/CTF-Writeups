# Scepter - Hack the Box

---

This is a writeup for the hard-level CTF "Scepter" on Hack the Box. This room is located at https://app.hackthebox.com/machines/Scepter and is a retired room. I am documenting the process I used to find all information in this writeup **WITHOUT** including any flags, in the spirit of the game. However, following this process exactly should result in a full compromise of the target system.

NOTE: Common issues with this machine stem from clock skew (fixed easily with `sudo ntpdate <DC IP>`) and a clean-up script that runs as a scheduled task on this machine. Every 15 minutes, this machine will reset certain elements that you may have changed, so if something is failing that worked before, retrace your steps. If you don't take good notes, this is a great reason to start :)

---

## Recon, Scanning, and Enumeration

My first step was to confirm connectivity to the victim IP address.

![](./screenshots/ping.png)

Next I ran a quick `nmap` scan to see which ports were responding on the host:

![](./screenshots/nmap.png)

The two important things we can note from the output (other than services that suggest we are looking at a domain controller) are the suggestion of NFS shares and the DNS hostname, which we can add to our /etc/hosts file:

![](./screenshots/etc_hosts.png)

We can use `showmount -e` to see which shares are exported on the domain controller:

![](./screenshots/showmount.png)

We can mount the NFS share, but it will have no permissions that we can use, as the permissions are still native to the share itself:

![](./screenshots/nfs_no_perms.png)

If we make our own copy of the NFS share's contents, however, we can interact with these files however we'd like:

![](./screenshots/nfs_openmount.png)

Optionally, we can give our machine user permissions so that we don't have to `sudo` everything:

![](./screenshots/chmod_666.png)

We can use `pfx2john` to extract the password hashes from these files in a format that `john` can crack, which results in the same password for all three files:

![](./screenshots/pfx2john.png)

However, if we attempt to auth with one of these certs (after stripping the encryption password), we see that these certs have been revoked:

![](./screenshots/clark_cert_failure.png)

Since we have "baker.crt" and "baker.key," we can assume the encryption password is the same and assemble a certificate that will successfully authenticate:

![](./screenshots/baker_key.png)

I had issues using this TGT for authentication to LDAP, so I used the hashes instead. We can use the hashes we got from `certipy` and run bloodhound-ce-python to aggregate data to import into Bloodhound:

![](./screenshots/bhce-python.png)

## Domain Enumeration and Initial Access

By checking d.baker's outbound object control, we can see that this user has permissions to change a.carter's password:

![](./screenshots/baker_outbound.png)

With these permissions, we can use `impacket-changepasswd` to change a.carter's password to something super secure, like "password" :)

By checking a.carter's outbound object control, we can see that this user has unrolled Full Control permissions over d.baker:

![](./screenshots/carter_outbound.png)

This may seem useless at first, but since the d.baker is unlikely to have rights to modify its own attributes, a.carter will be able to modify these values. This is a hint that we will be working with weak certificate mapping to escalate privileges.

We can use `certipy` to enumerate certificate templates that will work for our privilege escalation attempt(s). We can see that the "StaffAccessCertificate" template allows for both client and server authentication, and the "Staff" group can enroll. Note also that the `SubjectAltRequireEmail` flag is set, indicating that to enroll, a principal will need to have its `mail` attribute set:

![](./screenshots/certipy_find.png)
![](./screenshots/certipy_find_output.png)
NOTE: Certipy identifies that the "StaffAccessCertificate" template is vulnerable to ESC9. However, due to a registry key on the domain controller that `certipy` is unable to read, ESC9 **will not work** in this environment. Feel free to poke around and figure out why ;)

We can confirm in Bloodhound that d.baker is a member of the "Staff" group:

![](./screenshots/baker_staff.png)

Because the first escalation that I thought of using the `mail` attribute is ESC14b, I checked different principals to see if any of them had the `altSecurityIdentities` attribute set to include a mail address:

![](./screenshots/bloodyAD_get_object_mail.png)

So now we have identified our attack chain - we can change a.carter's password using d.baker's change password permissions, then we can use our new access to a.carter to set d.baker's mail attribute to "h.brown@scepter.htb," which will allow certificates from the "StaffAccessCertificate" template to authenticate as h.brown:

![](./screenshots/esc14b_1.png)

Now if we authenticate with this certificate, we can select a username, and because of the weak X509 mail mapping we can authenticate as h.brown:

![](./screenshots/esc14b_auth.png)

## User Flag and Privilege Escalation

Using Bloodhound, we can see that h.brown is in a handful on interesting groups:

![](./screenshots/brown_groups.png)

For example, because of membership in the "Protected Users" group, h.brown is denied NTLM authentication:

![](./screenshots/brown_nxc.png)

However, since we are in the "Remote Desktop Users" group, we are granted Win-RM access as long as we use an authentication method that is permitted for the "Protected Users" group - like our TGT!

![](./screenshots/brown_winrm.png)
Note that we will need to populate a realm in the /etc/krb5.conf file, and I have found that using `sudo` to access it ensures we are using our realm properly.

After we have grabbed our flag, we can use `bloodyAD` to check what we can write to, since we assume we can escalate privileges from the h.brown user:

![](./screenshots/brown_get_writable.png)

We can see that we have some kind of write access to the "p.adams" user. In Bloodhound, we don't have a direct path from h.brown to p.adams, but we can see that p.adams has DCSync rights over the domain, so this is a full domain compromise if we can get there.

![](./screenshots/adams_outbound.png)

Since Bloodhound didn't show us what the Write access we saw in `bloodyAD` was referring to, we'll need to enumerate ACLs that Bloodhound doesn't currently have edges for. Using `impacket-dacledit` to see what privileges we have, we come up dry at first:

![](./screenshots/dacledit_read_p.adams_1.png)

But, if we check the "CMS" group that h.brown is in, we can see that we have write access to `altSecurityIdentities`, which will enable us to perform ESC14a:

![](./screenshots/dacledit_read_p.adams_2.png)

There are several different routes for ESC14a, several of which are exploitable on this machine using either the "StaffAccessCertificate" template or the "HelpdeskEnrollmentCertificate" template, which is not discussed in this writeup (Check other writeups to see who used this method and to broaden your understanding of different methodologies). 

Since we already have a known route using ESC14b, we can follow the same path for ESC14a, writing to the `altSecurityIdentities` attribute of p.adams and setting it to a weak mapping for mail, the same way we exploited ESC14b to escalate to h.brown initially:

![](./screenshots/esc14a.png)

Using the DCSync rights granted to p.adams, we can compromise the domain and gain a session as the administrator, allowing us to read the root flag.

![](./screenshots/dcsync.png)

This was an extremely fun AD challenge. Major credit to EmSec for the creation of this machine, and acknowledgement to user "Malte" from the HTB discord server for helping me troubleshoot some inner workings after I completed my initial run of the box. Also, check out [this writeup](https://benheater.com/hackthebox-scepter/) written by OxBEN, who indulged me in some further theory discussions after initial machine completion. 