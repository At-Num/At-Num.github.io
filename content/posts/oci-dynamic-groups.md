---
title: "OCI Dynamic Groups"
date: 2022-09-02T20:19:49+03:00
description: Oracle Cloud Infrastructure IAM
summary: OCI Dynamic Groups vs AWS STS RBAC
draft: false
---

If you are coming into OCI from AWS then you are used to using [STS](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html) credentials with roles to allow resources such as instances or functions to make authenticated requests.
An AWS STS token grant, assigns a
 [tempory token](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html) via a user or AWS(Service-Linked ) defined role aka [RBAC](https://en.wikipedia.org/wiki/Role-based_access_control).  

AWS RBAC allows for permissions to be defined in policies and Trust relationships are defined in separate ’trust’ policies.  

A number of security issues exist when you have privileged tokens getting passed around temporary or not, to mitigate against this, AWS roles use the defined Trust relationships attached to resources, these ’trust’ policies have a syntax that allows you to have a very fine grained control over the scope of the permissions attached to the token.
Aside from the obvious risk of poorly crafted trust policies, the complexity introduced when you allow
[cross-account role enumeration](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html) can allow privilege escalation.

In the AWS ecosystem it is common to allow third party vendors of logging, security, data integrations, or other services to access your account. AWS encourages you to grant access to third parties via the cross-account-assume-role mechanism with the use of an [external ID](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html). AWS has worked [hard](https://aws.amazon.com/blogs/security/how-to-use-external-id-when-granting-access-to-your-aws-resources/) on this architecture which has many advantages over other cloud vendors that use RBAC with secrets, however it has a critical [vunerability](https://en.wikipedia.org/wiki/Confused_deputy_problem) when the external ID declared in the trust policy is weak(guessable).


In general as developers we need to work hard to keep things as simple as possible knowing complexity will increase over time, as the more complex a system the more vulnerable it becomes. OCI has a very simple security architecture at the infrastructure level. OCI has no concept of roles at this level, instead we use [dynamic groups](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingdynamicgroups.htm) to bind permissions to resources in a context called a compartment(a logically related set of resources).


To illustrate this the [following](https://docs.oracle.com/en-us/iaas/Content/certificates/managing-certificate-authorities.htm) is extracted from the oci documentation for the OCI Certificate Management Service. We use this as an example to illustrate how we can use Dynamic groups to allow resources to make authenticated requests analogous to AWS STS RBAC.

We have a compartment (manage-tls) where we are using Certificate authority’s (CA) to make authenticated calls to oci services, for example a Vault to use a signing key. To allow these calls we create a dynamic group (CA-DG) attached to the compartment that has a “matching” rule
```
resource.type='certificateauthority'
```
now we can attach a permission to the dynamic group CA-DG for accessing a signing key from a vault in the compartment simply as
```
Allow dynamic-group CA-DG to use keys in compartment manage-tls
``` 
Now any CA in the compartment can access the Vault to obtain a signing key.


Atnum recommends OCI as a cloud vendor based on what we consider as its simple but highly effective security architecture. OCI dynamic groups allow authenticated resources to access services inside clear delimited boundaries.

AWS STS RBAC allows many complex scenarios such as cross account RBAC, however the complexity requires vast experience to safely architect at scale.














