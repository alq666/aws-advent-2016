# Secure (and usable) multi-AWS account setup

## From one to many: Account sprawl
With an email and a credit card anyone can sign up for AWS. And everyone did to the point that, if you are part of the team managing the AWS infrastructure at your organization, you’ve had to wrestle with this for some time now. Multiple accounts, even when you have assembled the definitive list of what’s in use by your colleagues create complexity in many areas: billing, compliance, security. It is through the lens of security that we are looking at managing multiple accounts, with the hope to provide a pattern that you can apply or tweak to your own case.

## The benefits of multiple AWS Accounts
While having multiple AWS account adds operational complexity, especially when they are managed in an ad hoc way, the principle is, from a security standpoint, defensible. Compared with a single account that hosts every single bit of infrastructure, separate accounts give you, from a security standpoint natural boundaries on multiple levels.

1. At the network level, accounts are isolated from one another unless their VPCs are explicitly peered.
1. At the API level, a compromised account has by default no access to other accounts, unless, again, API calls have been explicitly allowed with role delegation. 
1. At the compute level, if one account gets turned into a bitcoin-mining field, billing alerts or careful CloudTrail monitoring will alert you right away. And you can set per-account spend limits that make taking over a limited account, a much less palatable target.

From a security standpoint all of this is appealing, which leads to the question: how to make it work without adding significant overhead with each new account that is added to the mix.

Caveat lector: what’s about to be presented will only work with up to 10 different accounts, based on current limitations in IAM groups. It will thus apply to production environments much more so than to development environments—especially if every single developer gets her own account.

## The qualities of a secure and usable setup
There is plenty of detailed documentation on AWS Identity and Access Management (IAM), but very limited guidance on how to put everything together. The [AWS IAM best practices documentation](http://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) only covers very general topics e.g. “never have root API credentials” so they are good guidelines but not enough to come up with a design. 

To help come up with a solution, we are spelling out a number of requirements which we will then figure out how to implement.

1. IAM users can only exist in one account.
1. By default IAM users have very limited API access to any account.
1. Access to privileged API calls requires a user to assume a role in the target account.
1. To assume a role, a user needs an active MFA with a limited time-to-live.
1. Privileged API calls are grouped together by topic, e.g. VPC, DNS, IAM.

Let’s now look at each requirement and its rationale in detail.

### IAM users can only exist in one account
This simply keeps user management tractable. When someone joins and needs access to the AWS infrastructure, or when someone leaves, you want to only have to look into one spot. By the same token, if you need to apply password complexity rules, you’re in much better spot if you need only apply them in one account.

Conversely, by having IAM users only in one account, you can monitor all other accounts and alert if an IAM user gets created there. It should simply never happen.

### Why have IAM users at all?
Managing users independently in AWS IAM sounds error prone and dangerous. Why would an administrator not choose to use Single Sign On (SSO) and their existing central identity provider? The primary reason not to use SSO is because of one very important tradeoff. When an AWS account uses SSO for authentication it loses the ability to MFA protect tasks. Much of this strategy requires that the administrator is able to use MFA protection with fine grained precision. 

### By default IAM users have very limited API access to any account
By “very limited” we mean that a user should be able to:

1. Look at her own record.
1. Update her console password.
1. Create, update or delete her API credentials.
1. Create, update (and optionally delete) her own multi-factor authentication device.

And that’s it. So a user in the main account only allows authentication self-service but cannot do much beyond that. This helps limit the “blast radius” if the user’s credentials are compromised (as long as the MFA is not also compromised): an attacker with stolen username/password could at most look at the IAM user record and any attempt to user API credentials would trigger `401 Unauthorized` responses from AWS’ APIs.

### Access to privileged API calls requires a user to assume a role in the target account
This is how a user can actually get something done in AWS. Since the user operates under a very restrictive policy by default, she needs to (temporarily) elevate her privileges in a controlled fashion. This pattern is very similar to sudo in Unix land: she requests from the API elevated privileges by calling `sts:AssumeRole` with her own API credentials. If the call succeeds, she gets temporary API credentials and a session token to call the API and do something useful.

These credentials have a short time-to-live (maximum 1 hour by default) so even if they get lost, the time window to exploit them is short.

### To assume any role, a user needs an active MFA with a limited time-to-live
This again, follows the sudo pattern, but rather than expecting the user to re-enter her unix password, we force an MFA to be present for the `sts:AssumeRole` to even have a chance to succeed. Thus the 2 conditions for privilege escalations are:

1. User has an active MFA and, of course the MFA with her.
1. User has the right to assume the role she wants to assume (spoiler alert: this will be set by a group policy).

The MFA requirement, like any security measure, does not make an exploit impossible, just more expensive. It limits the damage that misplaced API credentials could have since any user API credentials without the MFA only provide read-only access to the user’s IAM record.

### Privileged API calls  are grouped together by topic, e.g. VPC, DNS, IAM
This is the final piece: to keep mapping users to activities reasonably easy to manage, we group them by scope. For instance, let’s assume you want a group of people to manage the network and DNS but not EC2 instances, and vice-versa, and a third group to manage S3. In this case you create 3 roles:

1. A `vpc` role whose policy allows network configuration to be changed.
1. An `ec2` role whose policy allows all EC2 calls except VPC.
1. An `s3` role whose policy allows all S3 calls.

## Tying it all together: an example
Now that we have all the requirements spelt out, we can get into the details of the implementation. The design is simple and relies solely on simple IAM constructs: users, groups and roles.

Let’s assume that you need 3 accounts named A, B and M and let’s start with 3 users, Alice, Bob and Charlie.

1. Alice is the network engineer responsible for managing VPCs across both accounts.
1. Bob is a developer who needs control over B’s EC2 infrastructure only (excluding VPC).
1. Charlie is an SRE who needs full administrative access over A and B.

At this point you also have full administrative access to all accounts, which you’ll be able to relinquish when you are done.

### Step 1: create a `users` group in the M(anagement) account
That `users` group will host all users and needs to have a policy that lets its members look at their own record and update their credentials and control their MFA, nothing more.

Here is an example of policy, `users.json`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowUsersAllActionsForCredentials",
            "Effect": "Allow",
            "Action": [
                "iam:ListAttachedUserPolicies",
                "iam:GenerateServiceLastAccessedDetails",
                "iam:*LoginProfile",
                "iam:*AccessKey*",
                "iam:*SigningCertificate*"
            ],
            "Resource": [
                "arn:aws:iam::__M__:user/${aws:username}"
            ]
        },
        {
            "Sid": "AllowUsersToSeeStatsOnIAMConsoleDashboard",
            "Effect": "Allow",
            "Action": [
                "iam:GetAccount*",
                "iam:ListAccount*"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "AllowUsersToListUsersInConsole",
            "Effect": "Allow",
            "Action": [
                "iam:ListUsers"
            ],
            "Resource": [
                "arn:aws:iam::__M__:user/*"
            ]
        },
        {
            "Sid": "AllowUsersToListOwnGroupsInConsole",
            "Effect": "Allow",
            "Action": [
                "iam:ListGroupsForUser"
            ],
            "Resource": [
                "arn:aws:iam::__M__:user/${aws:username}"
            ]
        },
        {
            "Sid": "AllowUsersToCreateTheirOwnVirtualMFADevice",
            "Effect": "Allow",
            "Action": [
                "iam:CreateVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:ResyncMFADevice"
            ],
            "Resource": [
                "arn:aws:iam::__M__:mfa/${aws:username}",
                "arn:aws:iam::__M__:user/${aws:username}"
            ]
        },
        {
            "Sid": "AllowUsersToListVirtualMFADevices",
            "Effect": "Allow",
            "Action": [
                "iam:ListMFADevices",
                "iam:ListVirtualMFADevices"
            ],
            "Resource": [
                "arn:aws:iam::__M__:*"
            ]
        }
    ]
}
```

### Step 2: create Alice, Charlie and Bob in the M(anagement) account as regular IAM users
This part is easy:

1. Create all 3 users in the `M` account
1. Add all 3 to the `users` group
1. Share the console credentials and instruct all 3 users to set up an MFA for their account.

### Step 3: create 3 roles in A and B accounts
For symmetry reasons it’s best to create the 3 roles in all accounts even if, in our example, no-one needs just EC2 access to the A account.

Thus you need to create a total of 6 roles:
1. A `vpc` role in A and B, with a policy  that gives control over VPC
1. An `ec2` role in A and B, with a policy that gives control over EC2 minus VPC
1. An `admin` role in A and B, with a policy that gives full control

Here are some example of policies that would make sense for each role:

* Admin policy for the `admin` role

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

* EC2 policy for the `ec2` role
```json
{
    "Version": "2012-10-17",
    "Statement": [
    {
      "Sid": "AllowEC2",
      "Action": [
        "ec2:*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "DenyVPC",
      "Action": [
        "ec2:AcceptVpcPeeringConnection",
        "ec2:AllocateAddress",
        "ec2:AssignPrivateIpAddresses",
        "ec2:AssociateAddress",
        "ec2:AssociateDhcpOptions",
        "ec2:AssociateRouteTable",
        "ec2:AttachClassicLinkVpc",
        "ec2:AttachInternetGateway",
        "ec2:AttachNetworkInterface",
        "ec2:AttachVpnGateway",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CreateCustomerGateway",
        "ec2:CreateDhcpOptions",
        "ec2:CreateFlowLogs",
        "ec2:CreateInternetGateway",
        "ec2:CreateNatGateway",
        "ec2:CreateNetworkAcl",
        "ec2:CreateNetworkAcl",
        "ec2:CreateNetworkAclEntry",
        "ec2:CreateNetworkInterface",
        "ec2:CreateRoute",
        "ec2:CreateRouteTable",
        "ec2:CreateSecurityGroup",
        "ec2:CreateSubnet",
        "ec2:CreateVpc",
        "ec2:CreateVpcEndpoint",
        "ec2:CreateVpcPeeringConnection",
        "ec2:CreateVpnConnection",
        "ec2:CreateVpnConnectionRoute",
        "ec2:CreateVpnGateway",
        "ec2:DeleteCustomerGateway",
        "ec2:DeleteDhcpOptions",
        "ec2:DeleteFlowLogs",
        "ec2:DeleteInternetGateway",
        "ec2:DeleteNatGateway",
        "ec2:DeleteNetworkAcl",
        "ec2:DeleteNetworkAclEntry",
        "ec2:DeleteNetworkInterface",
        "ec2:DeleteRoute",
        "ec2:DeleteRouteTable",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSubnet",
        "ec2:DeleteVpc",
        "ec2:DeleteVpcEndpoints",
        "ec2:DeleteVpcPeeringConnection",
        "ec2:DeleteVpnConnection",
        "ec2:DeleteVpnConnectionRoute",
        "ec2:DeleteVpnGateway",
        "ec2:DetachClassicLinkVpc",
        "ec2:DetachInternetGateway",
        "ec2:DetachNetworkInterface",
        "ec2:DetachVpnGateway",
        "ec2:DisableVgwRoutePropagation",
        "ec2:DisableVpcClassicLink",
        "ec2:DisassociateAddress",
        "ec2:DisassociateRouteTable",
        "ec2:EnableVgwRoutePropagation",
        "ec2:EnableVpcClassicLink",
        "ec2:ModifyNetworkInterfaceAttribute",
        "ec2:ModifySubnetAttribute",
        "ec2:ModifyVpcAttribute",
        "ec2:ModifyVpcEndpoint",
        "ec2:MoveAddressToVpc",
        "ec2:RejectVpcPeeringConnection",
        "ec2:ReleaseAddress",
        "ec2:ReplaceNetworkAclAssociation",
        "ec2:ReplaceNetworkAclEntry",
        "ec2:ReplaceRoute",
        "ec2:ReplaceRouteTableAssociation",
        "ec2:ResetNetworkInterfaceAttribute",
        "ec2:RestoreAddressToClassic",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:UnassignPrivateIpAddresses"
      ],
      "Effect": "Deny",
      "Resource": "*"
    }
  ]
}
```

* VPC policy for the `vpc` role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AcceptVpcPeeringConnection",
        "ec2:AllocateAddress",
        "ec2:AssignPrivateIpAddresses",
        "ec2:AssociateAddress",
        "ec2:AssociateDhcpOptions",
        "ec2:AssociateRouteTable",
        "ec2:AttachClassicLinkVpc",
        "ec2:AttachInternetGateway",
        "ec2:AttachNetworkInterface",
        "ec2:AttachVpnGateway",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CreateCustomerGateway",
        "ec2:CreateDhcpOptions",
        "ec2:CreateFlowLogs",
        "ec2:CreateInternetGateway",
        "ec2:CreateNatGateway",
        "ec2:CreateNetworkAcl",
        "ec2:CreateNetworkAcl",
        "ec2:CreateNetworkAclEntry",
        "ec2:CreateNetworkInterface",
        "ec2:CreateRoute",
        "ec2:CreateRouteTable",
        "ec2:CreateSecurityGroup",
        "ec2:CreateSubnet",
        "ec2:CreateTags",
        "ec2:CreateVpc",
        "ec2:CreateVpcEndpoint",
        "ec2:CreateVpcPeeringConnection",
        "ec2:CreateVpnConnection",
        "ec2:CreateVpnConnectionRoute",
        "ec2:CreateVpnGateway",
        "ec2:DeleteCustomerGateway",
        "ec2:DeleteDhcpOptions",
        "ec2:DeleteFlowLogs",
        "ec2:DeleteInternetGateway",
        "ec2:DeleteNatGateway",
        "ec2:DeleteNetworkAcl",
        "ec2:DeleteNetworkAclEntry",
        "ec2:DeleteNetworkInterface",
        "ec2:DeleteRoute",
        "ec2:DeleteRouteTable",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSubnet",
        "ec2:DeleteTags",
        "ec2:DeleteVpc",
        "ec2:DeleteVpcEndpoints",
        "ec2:DeleteVpcPeeringConnection",
        "ec2:DeleteVpnConnection",
        "ec2:DeleteVpnConnectionRoute",
        "ec2:DeleteVpnGateway",
        "ec2:Describe*",
        "ec2:DetachClassicLinkVpc",
        "ec2:DetachInternetGateway",
        "ec2:DetachNetworkInterface",
        "ec2:DetachVpnGateway",
        "ec2:DisableVgwRoutePropagation",
        "ec2:DisableVpcClassicLink",
        "ec2:DisassociateAddress",
        "ec2:DisassociateRouteTable",
        "ec2:EnableVgwRoutePropagation",
        "ec2:EnableVpcClassicLink",
        "ec2:ModifyNetworkInterfaceAttribute",
        "ec2:ModifySubnetAttribute",
        "ec2:ModifyVpcAttribute",
        "ec2:ModifyVpcEndpoint",
        "ec2:ModifyVpcPeeringConnectionOptions",
        "ec2:MoveAddressToVpc",
        "ec2:RejectVpcPeeringConnection",
        "ec2:ReleaseAddress",
        "ec2:ReplaceNetworkAclAssociation",
        "ec2:ReplaceNetworkAclEntry",
        "ec2:ReplaceRoute",
        "ec2:ReplaceRouteTableAssociation",
        "ec2:ResetNetworkInterfaceAttribute",
        "ec2:RestoreAddressToClassic",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:UnassignPrivateIpAddresses"
      ],
      "Resource": "*"
    }
  ]
}
```

You also need to let the `M` account assume each role so you’ll need to attach the following role delegation stanza to each role in all accounts. Note the requirement for the MFA.

```json

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::__M__:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

If you store the policy above in `assume-role-from-m.json` you can create the roles with the following:

```bash
# Create the admin role with the appropriate policy and role delegation
aws iam create-role --role-name admin --assume-role-policy-document file://assume-role-from-m.json
aws iam put-role-policy --role-name admin --policy-name admin --policy-document file://admin.json

# Create the vpc role with the appropriate policy and role delegation
aws iam create-role --role-name vpc   --assume-role-policy-document file://assume-role-from-m.json
aws iam put-role-policy --role-name vpc --policy-name vpc --policy-document file://vpc.json

# Create the ec2 role with the appropriate policy and role delegation
aws iam create-role --role-name ec2   --assume-role-policy-document file://assume-role-from-m.json
aws iam put-role-policy --role-name ec2 --policy-name ec2 --policy-document file://ec2.json
```

You need to do so in all accounts, `A`, `B` and `M`.

### Step 4: create the corresponding groups in M
This is the IAM constructs that binds users, the right to assume roles and the target roles. You’ll need 6 groups in the M account.

* `vpc-A`: can assume the `vpc` role in `A`
* `vpc-B`: same in `B`
* `ec2-A`: can assume the `ec2` role in `A`
* `ec2-B`: same in `B`
* `admin-A`: can assume the `admin` role
* `admin-B`: same in `B`

Here is how you bind the group to the right to assume a given role. You simply attach a policy (or inline it) to the group:

```bash
# Using the CLI to attach a policy
# http://docs.aws.amazon.com/cli/latest/reference/iam/put-group-policy.html

# Let members of the group admin-A become admins in the A account and so on
# To run in account __M__
aws iam put-group-policy --group-name admin-A --policy-document file://admin-A-delegation.json --policy-name admin-A
aws iam put-group-policy --group-name admin-B --policy-document file://admin-B-delegation.json --policy-name admin-B
aws iam put-group-policy --group-name vpc-A --policy-document file://vpc-A-delegation.json --policy-name vpc-A
aws iam put-group-policy --group-name vpc-B --policy-document file://vpc-B-delegation.json --policy-name vpc-B
aws iam put-group-policy --group-name ec2-A --policy-document file://ec2-A-delegation.json --policy-name ec2-A
aws iam put-group-policy --group-name ec2-B --policy-document file://ec2-B-delegation.json --policy-name ec2-B
```

Now any member of `vpc-A` can call `sts:AssumeRole` on the `vpc` role in account A. If the MFA is present, the `sts:AssumeRole` will now succeed.

### Step 5: add users to groups
Now that the machinery is in place, you just need to assign users to groups:

1. Alice belongs to `vpc-A` and `vpc-B`.
1. Bob belongs to `ec2-B`.
1. Charlie belongs to `admin-A` and `admin-B`.

And you are done.

## 10, and closing thoughts
If you have followed thus far, you may have spotted a limitation of this scheme: an IAM user cannot be part of more than 10 groups at once. Until that limit is raised, you’ll need to use group membership sparingly. How to make this work for more than 10 accounts at once is left as an exercise to the reader (to riff on Fermat’s famous and most nagging ever margin note).

IAM is a complex topic, with relatively little that’s shared beyond the official documentation. This is my modest attempt at reversing the trend so I more than welcome feedback, critiques and (civil) comments.
