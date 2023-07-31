# Route 53 DNSSec Implementation.   
**Objectives:**  
1. Understand and emphasise the role DNSSec plays and how to implement it in Route 53. 
2. Establish a chain of trust from the parent TLD zone to the domain zone.
3. Examine records returned before and after enabling DNSSec. Observe the signed records returned after enabling DNSSec.

First let us examine some core concepts and players in DNS.  
When a client tries to connect to a website or load an application, Domain Name System is what helps the client determine the IP address of the website/application. This happens via what is called 'Walking the DNS tree' where the DNS resolver on the client's computer performs multiple queries until it gets the IP address of the website.  
First the local cache is checked for the IP address and if it is not found, the ISP cache is checked. If it is still not found the DNS Root domain is queried. These are 13 IP addresses that the browser and all operating systems are aware of. The Root then returns the name server of the Top Level Domain of the queried website (.com, .org, .net) and the DNS resolver queries the TLD servers. The TLD server returns the record for the server that hosts the zone file of the requested domain, which is then queried and the server responds with the IP address. This IP address is returned to the user.  
As a result of the DNS query, the user gets either a cached or non authoritative response, or a response from an actual server that is authoritative for that domain. 

**DNSSec ensures 2 things, which are critical for security**. 
1. **Authenticity** of the record. It helps validate that the record returned is authentic for that domain. 
2. **Integrity** of the record. It ensures that the record has not been altered or tampered with. 
Without DNSSec, a bad actor could poison the DNS record which could result in the client being redirected to a malicious website.  In an organization it could cause user traffic to be hijacked.<br>
DNSSec achieves this by a **cryptographic signing process** where it digitally signs the record using a key.  The resolver/client can validate it to ensure the authenticity of the record. This process follows a chain of trust where the zone for that domain is trusted by the TLD zone which in turn is trusted by the Root Zone. The Root Zone is implicitly and universally trusted. The Root Key Signing ceremony happens at regular intervals where security officers perform the signing in an extremely secure, supervised and audited process. This ensures the security of the keys generated in the Root Zone which are used to digitally sign the keys of the TLDs which in turn sign the keys of the hosted zone. 

**Keys involved in DNSSec:**  
Let's start from the individual hosted zone. 
Every DNS zone has a Zone signing key which has a public and corresponding private portion. When DNSSec is enabled, DNS clients get an RRSIG record in addition to the DNS record which they can use to validate the authenticity of the DNS Record. This RRSIG record is a digitally signed DNS record, signed using the private portion of the Zone key.  
The public portion of the Zone signing key is stored in the DNS Key record of the zone, and the private portion is saved separate from the zone by the zone administrator. The DNS Key stores the public key of the Zone key. DNS Key also stores another important public key, the **Key Signing Key**.  
The DNS key of the zone is then signed by the private portion of the Key Signing key to create the RRSIG of the DNS key record. Using this record, clients can validate if the DNS key of the zone itself is valid. Now to establish a chain of trust, the public part of the Key Signing key is provided to the parent TLD zone as a Delegated Signer (DS) record. This is then digitally signed by the Zone signing key of the TLD zone. The validity of the TLD Zone signing key can be ensured because it is trusted by the Root zone. The Root zone has a DS record of the TLD Zone signing key key which it digitally signs using it's Key Signing Key.  The Key Signing Key of the Root zone is generated during a securely audited Root Key Signing Ceremony. To be really specific, it is not the key generated during the ceremony but is derived from the secure key.

> **Note:**
> As a prerequisite for this hands-on we need a public hosted zone on Route 53.
> It can be any name of your choice. When you register a domain using Route 53, a public hosted zone is automatically created as part of the process. [This link to AWS documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-register.html) has detailed steps that can be followed for registering a doman with Route 53.
> For every hosted zone we maintain in our AWS account, AWS charges $.50 per hosted zone per month. 

## Steps:
1. Create a Stack with one click deployment which will provision a corporate website. Create a DNS record in the public hosted zone for the domain. [Details](#Step1)
2. Perform DNS query before enabling DNSSec in the zone for the domain. [Details](#Step2)
3. Enable DNSSec via AWS admin console. [Details](#Step3)
4. Establish a chain of trust from the TLD .net zone to our zone. [Details](#Step4)
5. Perform DNS query after enabling DNSSec and observe the signed records returned. [Details](#Step5)
6. Cleanup [Details](#Step6)
7. Summary & Lessons learnt [Details](#summary)

# Implementation steps:
# Step1:<a name="Step1"></a>  
Create the VPC using the Cloudformation Template [here](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/files/01-SingleCustomVPCWithPublicSubnet.yaml).  

In Route 53, create an A record pointing to the IP address of the EC2 instance. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/publicZone.png) 

Test it.  
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/Step4-1.png)  
Bird graphic courtesy of freepik.
# Step2:<a name="Step2"></a>  
Perform DNS query before enabling DNSSec in the zone for the domain. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/digDnsA.png) 
# Step3:<a name="Step3"></a>
Enable DNSSec via Admin Console. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DNSSecEnable.png) 

Create a Key Signing Key. This is a KMS CMK. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/createKSK.png)

This also means that we now have a Zone signing key. The KSK will be used to sign the Zone signing key.  
The blurb below shows that this is complete.
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DNSEnabled2.png) 

Now when running the dig command, the DNSKEY record is returned. 
The record with **256** represents the **Zone signing Key** and the record with **257** represents the **Key Signing Key**. 
We also have the RRSIG of the DNSKEY which is the DNSKEY record digitally signed by the private portion of the Key Signing Key.  Resolvers can verify the validity of the DNSKEY record to make sure it is valid.
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/afterEnabling.png)
# Step4:<a name="Step4"></a>  
Establish a chain of trust by adding a Delegated Signer record to the TLD zone for .net.  
For this we need the DS Record information from our Route 53 registrar. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DSRecordInformation.png)  

If the domain was registered with Route 53, pick this. If it was registered via another registrar, select the relevant information.  
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DSRecordInformation2.png)

Now add the algorithm and the public key. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/ADDKSK.png)

The blurb shows the add request is in progress. We should get an email notification.
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DSRequested.png)

Email notification has been received. So now a Delegated Signer record has been added to the .net TLD for our domain.
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/KSKEmail.png)

Now let's try to view the DS record on the TLD, the .net domain.
For this, we need to get the name servers for the .net TLD as below:  
```dig net NS +short```  

Then query one of the servers for a DS record for this zone as below:  
```dig birds4ever.net DS @<nameserver>```  

We see the DS record that was successfully added. **This means that a chain of trust has been established from the .net TLD to our zone.**  
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DSRecordAdded.png)  
# Step5:<a name="Step5"></a>
Now when ```dig www.birds4ever.net A ``` returns just the A record, 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/digDnsA.png)  

The query ```dig www.birds4ever.net A +dnssec``` with dnssec flag returns RRSIG record. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DNSEnabledRRSIG2.png)  
The RRSIG record is the digitally signed A record, signed with the private part of the Zone signing key. 
This record can be verified by the DNS key which contains the public part of the Zone signing key. 
# Step6:<a name="Step6"></a>
Cleanup:  
First the DS record needs to be removed from the parent zone. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DeleteDNSSecKey.png) 

We should get an email notification when this has been completed in the TLD zone for .net.
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/requestToDelete.png) 

Received notification. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/DeleteDSEmail.png) 

Now we can disable DNSSec. 
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/disableDNSSEC.png) 

After this is done we can schedule a deletion of the KMS key.  
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/scheduleKeyDeletion.png) 

The key is scheduled to be deleted within the period specified.  
![Alt text](https://github.com/veeCan54/03-Route53DNSSECImplementation/blob/main/images/pendingDeletion.png) 

# Summary:<a name="summary"></a> 
**Concepts:**
1. How DNSSec works, what are the different keys involved.
2. How to enable DNSSec in a zone and establish a chain of trust.
3. How to view the DS records in the parent TLD zone. 

From AWS Documentation: **Disabling DNSSec in a production environment needs to be done after taking into consideration the TTL of the parent zone. Otherwise it could cause disruption of traffic to our site**.  
***DNSSec adds an extra layer on top of DNS, for security - so why would we want to disable it? Probably we might want to temporarily disable it for key rotation?***  

**TODO:**  
1. Research use cases for disabling DNSSec and under what circumstances would it be done. <br>
   Bird graphic courtesy of freepik <img src="https://github.com/veeCan54/00-EnvelopeEncryptionHandsOn/blob/main/images/freepic.png" width="70" height="10" />
