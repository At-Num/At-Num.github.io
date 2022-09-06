---
title: "OCI Load Balancer"
date: 2022-09-02T23:53:00+03:00
description: Oracle Cloud Infrastructure TLS
summary: OCI Load Balancer(what they don't tell you)
draft: false
---
The diagram below shows possibly the simplest setup for what is termed by OCI as a Load Balancer configured for 'point to point' TLS.
![compartment network](/image/ca1.png)
OCI has extensive documentation for Load Balancers(LB) however, at least as it applies to TLS, it can lead to some confusion. For example SSL itself is depreciated and the system defaults to implementing TLS version 1.2, while the documentation refers to SSL, but this is just pedantic quibiling. 
 
The description of the architectures for implementing TLS (see oci docs for [concepts](https://docs.oracle.com/en-us/iaas/Content/Balance/Concepts/balanceoverview_topic-Load_Balancing_Concepts.htm) ) could be upgraded to make things clearer, they describe 3 types of TLS configurations TERMINATION, POINT-TO-POINT and TUNNELING.
- TERMINATION is what you would use unless forced by compliance or your backend set instances require TLS to function.
- TUNNELING is at network layer 4 and to be avoided, if you want tunneling better to use an actual Layer 4 load balancer
- POINT-TO-POINT is what we will discuss here and most devs would use the term `end to end` as in this very helpful [a-team blog](https://www.ateam-oracle.com/post/load-balancing-ssl-traffic-in-oci)
### Point to point tls
![compartment network](/image/ca4.png)
for the above diagram we are
- terminating the LB at the `front` of the LB with a Lets Encrypt(LE) CA bundle.
- configuring a point to point TLS connection with a CA bundle managed by OCI Certificates(service), where the CA bundle is produced with a vault signing key and a OCI user created CA, we outlined this in a previous [post](/posts/oci-cert-auth/)
- creating a TLS connection between the LB and the backend set instances using self signed certificates with nginx  

Configuration
--
- The Listener: typically LB Managed with a CA bundle from the CA bundling programmes of various OS/browsers, port 443 with https and not verifying peer certificate.
- Back end set:  typically Certificate Service managed CA bundle, port 443 with https and not verifying peer certificate.
- instances:  need to be configured for TLS with correct TLS version and compatible ciphers
 
Going Further
--
An astute reader at this point should be asking the following question, `why would we need the OCI CA bundle on the backend?` The TLS between the LB backend and the Nginx server is entirley handled by the Nginx certs and private key, we are not implementinmg mTLS so for the setup above the OCI CA bundle appears unused.
OCI presents ther LB to us as an opaque black box, we have the ability to configure its parts(Listener, WAF, HC, BE Set ect...) but there is no information about the internal architecture of the LB. What follows is entirely hypothetical, 
consider the following,
- TLS < V1.3 puts a load on the system above http, at scale it makes sense to take steps to mitigate this
- it is hoped OCI is using strategies to offload specific computational tasks to optimized hardware
- perhaps what we see as our LB is in fact a node in a cluster that can manage all their LB tasks 
 
In this scenario the requirement for the LB backed CA now makes a lot of sense as a trusted identifier for a process in the LB providing cluster, this is just speculation as to why the backend LB CA is required for this use case, that is point to point without mTLS.
 
 
 
 
Troubleshooting
--
- health check on port 443
- generally don't check peer certificate
- only TLS v1.2 supported by OCI
- need compatible cipher [suite](https://docs.oracle.com/en-us/iaas/Content/Balance/Tasks/managingciphersuites_topic-Predefined_Cipher_Suites.htm#predefined) on instances
- default routes for http request "/" so the server block for nginx is the default server for example
- tcpdump wont work with encrypted packets  

Notes
--
- https TLS version < 1.3 (OCI  only 1.2 currently)
- https TLS version < 1.3 puts a significant load on everything so TLS termination prefered
- prefer termination or point to point to stay at level 7, at layer 4 (TUNNELING) routing policies, sets and rule sets, filtering(WAF) are all unavailable


