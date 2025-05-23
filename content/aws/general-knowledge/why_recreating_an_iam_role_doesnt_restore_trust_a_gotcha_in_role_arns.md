---
author_name: Nick Frichette
title: "Why Recreating an IAM Role Doesn't Restore Trust: A Gotcha in Role ARNs"
description: "In AWS, deleting and recreating an IAM role results in a new identity that breaks existing trust policies. This behavior improves security by preventing identity spoofing but can cause failures in cross-account access and third-party integrations if not properly understood."
---

**TL;DR**: In AWS IAM, trust policies often reference other roles by their Amazon Resource Name (ARN). But if a referenced IAM role is deleted and recreated, even with the *same name*, the trust policy breaks. The new role may look identical, but AWS assigns it a different internal identity, and trust relationships no longer apply.

This subtle behavior can disrupt cross-account access, third-party integrations, and automation workflows.

!!! info
    While this article is primarily focused on IAM roles and avoiding making this mistake in a SaaS context, it is important to know that the same idea applies to IAM users as well. In addition, IAM principals can be referenced in resource-based policies of a variety of resources, not just role trust policies.

## The Scenario

Imagine you have a trust policy attached to a role named `Bobby` and that this trust policy permits the role named `Megan` to assume it as shown below.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111111111111:role/Megan"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

If the `Megan` role is deleted and then recreated, perhaps using automation or Terraform, users may expect this policy to continue working. After all, the ARN is the same, why wouldn't it work? Well, how about we delete the Megan role, recreate it, and see what happens to the trust policy?

## Deleting and Recreating the Role
Should we delete `Megan`, then recreate her role and check back in on `Bobby`'s trust policy we will find it now looks like this:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "AROAABCDEFGHIJKLMNOPQ"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

What happened? Where did the ARN go? Megan?

## Why This Happens

At first glance, it’s easy to assume that the ARN of a role is its unique identifier. After all, it's what we use in trust policies, logs, error messages, and Terraform or CloudFormation templates. It looks like a primary key, acts like a primary key, so it must be the primary key, right?

Not quite.

Under the hood, AWS assigns each IAM role an internal, immutable [principal ID](../general-knowledge/iam-key-identifiers.md) when it is created. This principal ID, not the ARN, is what actually identifies the role in trust relationships, policy evaluations, and service-level authorizations.

The ARN is best thought of as a human-readable label, similar to a username. It’s a convenient pointer, but it’s not the source of truth. When you delete a role, AWS also discards the associated principal ID. Recreating a role, even with the exact same name, results in a completely new role with a new principal ID. The ARN may be identical, but the underlying identity is not.

This is why a trust policy that still references the original ARN will be replaced with the principal ID: it's pointing to an identity that no longer exists. The policy is technically valid JSON, but AWS can no longer resolve that ARN to a live principal with the matching principal ID.

AWS explicitly calls this out in their [IAM documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html#principal-roles):

> If your `Principal` element in a role trust policy contains an ARN that points to a specific IAM role, then that ARN transforms to the role (sic) unique principal ID when you save the policy. This helps mitigate the risk of someone escalating their privileges by removing and recreating the role. You don't normally see this ID in the console, because IAM uses a reverse transformation back to the role ARN when the trust policy is displayed. However, if you delete the role, then you break the relationship. The policy no longer applies, even if you recreate the role because the new role has a new principal ID that does not match the ID stored in the trust policy

This is further elaborated in this [re:Post article](https://repost.aws/articles/ARSqFcxvd7R9u-gcFD9nmA5g/understanding-aws-s-handling-of-deleted-iam-roles-in-policies).

!!! Note
    This behavior has an additional benefit for enumeration. We can use resource based policies to convert the [Principal ID to an ARN](../enumeration/enumerate_principal_arn_from_unique_id.md).

## Why This is a Good Thing

While frustrating at times, this behavior is a security feature. It prevents someone from deleting a trusted IAM role and then recreating it to inherit that trust, which could otherwise lead to unintended privilege escalation or lateral movement.

By locking trust relationships to unique principal IDs, AWS ensures that trust must be explicit and intentional, not assumed by name reuse.

## Why this may be Dangerous

While tying trust relationships to immutable principal IDs is a sound security decision, this behavior can introduce operational risk, especially in SaaS integrations.

Many SaaS platforms, especially in the security, observability, or data pipeline space, allow customers to establish integrations by trusting a specific IAM role via an ARN. The SaaS provider configures their side to call `sts:AssumeRole` on a role in a customer’s AWS account and uses that role to perform whatever their service needs to do.

Say that SaaS provider makes a mistake and deletes the trusted IAM Role and recreates it (intentionally or not), that new IAM role will have a different principal ID. While the ARN may be the same, from AWS' perspective that is not the same IAM role. The result? 

The SaaS provider will not be able to assume any of their customer roles.

To make matters worse, the only solution in this situation is for every single customer to modify the trust policy of their SaaS roles in every single AWS account to trust the new IAM role in the SaaS account. This can introduce downtime, addition support requests, and other issues.

## How to Avoid This

The simplest method for avoiding this operational risk is to have your SaaS customer trust an entire AWS account, and not a specific IAM role. Below is an example of such a trust policy.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111111111111:root"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

This trust policy will permit any AWS principal in the account `111111111111` to assume the role in the customer account. 

!!! warning
    It is important that you restrict who has access to the trusted account and limit what principals have `sts:AssumeRole` privileges.

As an alternative, you can modify the trust policy to trust the entire AWS account and then use a `Condition` block to restrict access to a specific role ARN. Unlike when a role ARN is listed directly in the `Principal` field, using `aws:PrincipalArn` in a condition does not evaluate the principal's role ID, it matches only the string value of the ARN. This means that if you delete and recreate the role with the same name, the trust policy will continue to work as long as the ARN matches.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "ArnEquals": {
          "aws:PrincipalArn": "arn:aws:iam::111111111111:role/Megan"
        }
      }
    }
  ]
}
```

!!! warning
    Of course, while this approach improves resilience to role deletion, it also removes the protection from privilege escalation and lateral movement abuse offered by the original example. You should carefully evaluate what behavior you need depending on your specific scenario. 

## Conclusion

IAM roles may look simple on the surface, but the way AWS handles trust relationships is complex: identity resources are more than their name. When you delete and recreate a role, even with the same ARN, AWS treats it as a completely different entity. That distinction can lead to subtle, hard-to-debug failures, especially in SaaS integrations.

Whether you’re building secure infrastructure, managing third-party access, or testing cloud security boundaries, understanding this behavior is essential. Trust isn’t just about syntax—it’s about identity, and AWS is very specific about who it trusts.
