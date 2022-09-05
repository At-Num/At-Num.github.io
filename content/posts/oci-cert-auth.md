---
title: "OCI Certificate Authority (CA)"
date: 2022-09-02T23:48:30+03:00
description: Oracle Cloud Infrastructure TLS
summary: OCI certificate authorities nuts and bolts
draft: false
---
TLS everywhere
--
The diagram below shows the set of resources within the compartment manage-tls. We have a Certificate Authority(CA) and a Vault with a RSA signing key attached to the compartment.
![compartment view](/image/ca0.png)

What follows is an outline of the steps to get everything working in terms of OCI IAM to use the OCI Certificate Authority to generate certificates(certs) for the scaling solution using a public Layer 7 Load Balancer with end to end TLS as shown below.
![compartment network](/image/ca1.png)
 
Termination of TLS 
--
At the public side of the Load Balancer(LB) we will need a cert that the browser accepts, that is a cert signed by a well known Certificate Authority(CA), that is a CA from the set of well-established CA's that are part of the CA bundling programmes of various OS/browsers, here for example we are using Lets Encrypt(LE). Once the public IP of the LB DNS is set up with the domain everything just works. TLS is terminated and importantantly we stay at layer 7 so that the http packets are available to the LB for filtering via a WAF or routing to the backend set.
 
Backend TLS
--
To implement TLS in the backend we create an OCI CA with the Vault, we generate a signing key in the Vault and use it to generate a cert with our CA.  
The remainder of this post outlines the IAM permissions required by compartment manage-tls for all the resources to be authorized with the required permissions. In a previous post we discussed the role of [dynamic groups](/posts/oci-dynamic-groups/) in this process.
 
Groups and Policies
---
When you attempt to create a CA in a compartment if your CA lacks required permissions you will get an error like
```
Authorization failed or requested resource not found: Key Id ocid1.key.oc1..........
```
This follows as the resource principal, the CA in this case is not authenticated, We will go a bit further to the solution for this as outlined in [dynamic groups](/posts/oci-dynamic-groups/). Previously we set up the dynamic group CertificateAuthority-DG (CA_DG) with a matching rule
```
resource.type='certificateauthority'
```
once we establish this group every CA we create will be in this group. We then decide what compartments we will allow authorisation(s) then attach a policy(s) to compartments to allow authorisation(s) for the dynamic group.
From the [docs](https://docs.oracle.com/en-us/iaas/Content/certificates/managing-certificate-authorities.htm#creating_certificate_authority)
```
Allow dynamic-group CertificateAuthority-DG to use keys in compartment manage_tls
Allow dynamic-group CertificateAuthority-DG to manage objects in compartment manage_tls
```
now CA's in compartment manage_tls will have access to Vault signing keys and storage objects for export. Its best practice is to use IAM users and groups and the docs[1] contain examples for this however for simplicity we are assuming you are the tenancy administrator.
 
Note the members dynamic group CA_DG are resource principles and hence require the privileges granted by the policies in the compartment independent of the IAM status of the user principal.  
 
Note the dynamic group exists independent to the compartment, different policies for the members of this group can apply in different compartments. The diagram below summarizes where we are at.
![compartment with IAM](/image/ca2.png)
 
Create a backend cert
--
We use the Certificate service to create an [internal cert](https://docs.oracle.com/en-us/iaas/Content/certificates/managing-certificates.htm) that is it will be signed by our previously created CA. OCI will ask for a common name here we would typically use a descriptive name, then it will require a dns name typically a domain name to IP, then we would choose TLS Server or Client for type, and our CA. Note our cert exists in the compartment.
 
Summary
--
As an administrator user we have the required IAM permissions to create the security artifacts in the diagram below to the left, good practice would require we create an IAM user and groups then attach the needed permissions to the compartment as outlined in the  [docs](https://docs.oracle.com/en-us/iaas/Content/certificates/managing-certificate-authorities.htm#creating_certificate_authority). However to get the parts to work together we needed to use the dynamic group with policies attached to the compartment.
![final compartment with IAM](/image/ca3.png)