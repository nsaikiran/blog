---
layout: post
title:  "How can containers access AWS resources?"
description: "How can containers access AWS resources"
categories: ["aws"]
date: 2025-01-29 19:45:31 +0530
author: "Sai Kiran"
comments: false
---

[Diving into IAM Roles for Service Accounts](https://aws.amazon.com/blogs/containers/diving-into-iam-roles-for-service-accounts/) helped me to get high level understanding.
But below diagram from above doc will give quick overview.

![Architecture](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2022/02/26/Screen-Shot-2022-02-25-at-9.23.07-PM.png "Architecture")

In my words:

* In AWS IAM, you'll create roles which are associated with policies, policies will have list of permissions.  You'll annotate the containers with that role via kubernetes service accounts.
* You'll need to create an IAM OIDC provider for your cluster, which provides tokens to your service.
* The above OIDC provider is trusted by the IAM.


[IAM Roles for Service Accounts (IRSA)](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/) was launched and then they launched [Amazon EKS Pod Identity](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/) further enahance experience. Not going into details but go through these docs for more info (or use chatbots to know about these).

How to use IRSA (for AWS EKS),
* Create an IAM OIDC provider for your cluster
* Assign IAM roles to Kubernetes service accounts
* Configure Pods to use a Kubernetes service account
* Use IRSA with the AWS SDK

These steps are clearly mentioend in above doc, but I'm having it here to get overview. 

OIDC is setup on top of OAuth2.0, remember the architecture of OAuth2.0 and know about OIDC also.

[Create OIDC provider (AWS Console)](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html#_create_oidc_provider_console) 


Other learnings:
We were fixing the AWS S3 connectivity issue at work. [AWS SDK for Java BOM](https://aws.amazon.com/blogs/developer/managing-dependencies-with-aws-sdk-for-java-bill-of-materials-module-bom/) helped us having all uniform AWS SDK components, we needed to upgrade the version of SDK. AWS SDK components can get access tokens while accessing AWS resources based on the IAM role set via service account config on the pod.
