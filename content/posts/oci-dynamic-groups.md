---
title: "OCI Dynamic Groups vs AWS STS"
date: 2022-09-02T20:19:49+03:00
description: Oracle Cloud Infrastructure IAM
summary: OCI Dynamic Groups vs AWS STS RBAC
draft: false
---

If you are coming into OCI from AWS then you are used to using [STS](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html) credentials with roles to allow resources such as instances or functions to make authenticated requests.
An AWS STS token grant, assigns a
 [temporary token](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html) via a user or AWS(Service-Linked ) defined role, we can think of this loosely as [RBAC](https://en.wikipedia.org/wiki/Role-based_access_control).  

AWS RBAC allows both for permissions to be defined in policies and Trust relationships to be defined in separate ’trust’ policies.  

A number of security issues exist when you have privileged tokens getting passed around temporary or not, to mitigate against this, AWS roles uses the defined Trust relationships attached to resources, these ’trust’ policies have a syntax that allows you to have a very fine grained control over the scope of the permissions attached to the token.  

In the AWS ecosystem it is common to allow third party vendors of logging, security, data integrations, or other services to access your account.

 AWS encourages you to grant access to third parties via the cross-account-assume-role mechanism with the use of an [external ID](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html). 

Aside from the obvious risk of poorly crafted trust policies, the complexity introduced when you allow
[cross-account role enumeration](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html) can allow privilege escalation.

 AWS has worked [hard](https://aws.amazon.com/blogs/security/how-to-use-external-id-when-granting-access-to-your-aws-resources/) on this architecture which has many advantages over other cloud vendors that use RBAC with secrets, however it has a critical [vulnerability](https://en.wikipedia.org/wiki/Confused_deputy_problem) when the external ID declared in the trust policy is weak(guessable).


In general as developers we need to work hard to keep things as simple as possible knowing complexity will increase over time, it follows that the more complex a system is then the more vulnerable it becomes. At the IaaS level OCI has a very simple security architecture and has no concept of roles, instead we use [dynamic groups](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingdynamicgroups.htm) to bind permissions to resources in a context called a compartment(a logically related set of resources).


To illustrate this, the [following](https://docs.oracle.com/en-us/iaas/Content/certificates/managing-certificate-authorities.htm) is extracted from the oci documentation for the OCI Certificate Management Service. We use this as an example to illustrate how we can use Dynamic groups to allow resources to make authenticated requests analogous to AWS STS RBAC.

We have a compartment (manage-tls) where we are using Certificate authority’s (CA's) to make authenticated calls to oci services, for example a Vault to use a signing key. To allow the CA to be authenticated to use the Vault service, first we create a dynamic group (CA-DG) then attach CA-DG to the compartment. We create  the following  “matching” rule in CA-DG
```
resource.type='certificateauthority'
```
Secondly we create a policy in the compartment, for CA-DG that gives permission to members of CA-DG when accessing a signing key from a vault in the compartment.
```
Allow dynamic-group CA-DG to use keys in compartment manage-tls
``` 
Now any CA in the compartment can access the Vault to obtain a signing key. This is a much simpler security model, more resilient to user error, as the complexity of our infrastructure increases the security will stay simpler and clearer.

Note: while the policy that grants permission is attached to the compartment, the dynamic group is independent of any compartment, it has tennacy scope.

AtNum recommends OCI as a cloud vendor based on what we consider as its simple but highly effective security architecture. OCI dynamic groups allow authenticated resources to access services inside clear delimited boundaries.

AWS STS RBAC allows many complex scenarios such as cross account RBAC, however with the complexity comes the security responsibility.













