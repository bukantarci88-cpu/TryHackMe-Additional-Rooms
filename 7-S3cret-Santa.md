# AWS Security - S3cret Santa

****

### Task 1 - Introduction

One of our stealthiest infiltrated elves managed to hop their way into Sir Carrotbane’s office and, lo and behold, discovered a bundle of cloud credentials just lying around on his desktop like forgotten carrots. The agent suspects these could be the key to regaining access to TBFC’s cloud network. If only the poor hare had the faintest clue what “the cloud” is, he’d burrow in himself. Let's help the elf utilise these credentials to try to regain access to TBFC's cloud network.

AWS accounts can be accessed programmatically by using an Access Key ID and a Secret Access Key. For this room, both of those will be automatically configured for you. The AWS CLI will look for credentials at `~/.aws/credentials`, where you should see something similar to the following:

```bash
aws_access_key_id = AKIAU2VYTBGYOYXYZXYZ
aws_secret_access_key = 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
```

Amazon Security Token Service (STS) allows us to utilise the credentials of a user that we have saved during our AWS CLI configuration. We can use the `get-caller-identity` call to retrieve information about the user we have configured for the AWS CLI. Let's run the following command:

```powershell
      user@machine$ aws sts get-caller-identity
{
    "UserId": "AIDAU2VYTBGYOHNOCJMX3",
    "Account": "332173347248",
    "Arn": "arn:aws:iam::332173347248:user/sir.carrotbane"
}
```

Seeing the output, the elf was overjoyed. The credentials work, and as can be seen by the name at the end, they belong to Sir Carrotbane. The elf can now attempt to regain access to TBFC's cloud network using these credentials.

**Answer the questions below**

> Run `aws sts get-caller-identity`. What is the number shown for the "Account" parameter?
> 

123456789012

Run the query below in Linux terminal in order to returns details (such as the account ID, user ID, and ARN) about the IAM user or role currently making the request.: **aws sts get-caller-identity**

### Task 2 - IAM: Users, Roles, Groups and Policies

**IAM Overview:** AWS Identity and Access Management (IAM) controls user permissions to AWS resources, making proper configuration critical to preventing severe security breaches and data exposure.

**IAM Users:** A user represents a single identity in AWS. Each user has a set of credentials, such as passwords or access keys, that can be used to access resources. Furthermore, permissions can be granted at a user level, defining the level of access a user might have.

**IAM Groups:** An AWS IAM group simplifies access management by allowing administrators to assign permissions to multiple users at once, rather than individually.

**IAM Roles:** An AWS IAM role is a temporary identity that users, services, or external accounts can assume to gain specific, task-dependent permissions.

**IAM Policies:** Access provided to any user, group or role is controlled through IAM policies. A policy is a JSON document that defines the following:

- What action is allowed (Action)
- On which resources (Resource)
- Under which conditions (Condition)
- For whom (Principal)

```powershell
      {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificUserReadAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/Alice"
      },
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-private-bucket/*"
    }
  ]
}
```

This policy grants access to the AWS user Alice (Principal) to get an object from an S3 bucket (Action) for the S3 bucket named my-private-bucket (Resource).

**Answer the questions below**

> What IAM component is used to describe the permissions to be assigned to a user or a group?
> 

Policy

### Task 3 - Practical: Enumerating a User's Permissions

**Enumerating Users** `aws iam list-users`

**Enumerating User Policies**`aws iam list-user-policies --user-name sir.carrotbane`

We can find out by running the following command.

`aws iam list-attached-user-policies --user-name sir.carrotbane`

Sir Carrotbane is part of a group.

`aws iam list-groups-for-user --user-name sir.carrotbane`

Looks like **sir.carrotbane** is not a part of any group.

Let's get back to the inline policy we found for Sir Carrotbane's account. Let's see what permissions this policy grants by running the following command (replace `POLICYNAME` with the actual policy name you found):

`aws iam get-user-policy --policy-name POLICYNAME --user-name sir.carrotbane`

```bash
{
    "UserName": "sir.carrotbane",
    "PolicyName": "POLICYNAME",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "iam:ListUsers",
                    "iam:ListGroups",
                    "iam:ListRoles",
                    "iam:ListAttachedUserPolicies",
                    "iam:ListAttachedGroupPolicies",
                    "iam:ListAttachedRolePolicies",
                    "iam:GetUserPolicy",
                    "iam:GetGroupPolicy",
                    "iam:GetRolePolicy",
                    "iam:GetUser",
                    "iam:GetGroup",
                    "iam:GetRole",
                    "iam:ListGroupsForUser",
                    "iam:ListUserPolicies",
                    "iam:ListGroupPolicies",
                    "iam:ListRolePolicies",
                    "sts:AssumeRole"
                ],
                "Effect": "Allow",
                "Resource": "*",
                "Sid": "ListIAMEntities"
            }
        ]
    }
}
```

So, it looks like Sir Carrotbane has access to enumerate all the different kinds of users, groups, roles and policies (IAM entities), but that is about it. That is not a lot of help getting TBFC's access back. We might need to try something different to do that. If you look carefully, you'll notice sir.carrotbane can perform the `sts:AssumeRole` action. Maybe there's still hope!

**Answer the questions below**

> What is the name of the policy assigned to `sir.carrotbane`?
> 

SirCarrotbanePolicy

Run the query below in Linux terminal , I retrieved the details of a specific inline policy (POLICYNAME) attached to the user sir.carrotbane. : **** `aws iam get-user-policy --policy-name POLICYNAME --user-name sir.carrotbane`

### Task 4 - Assuming Roles

**Enumerating Roles** `aws iam list-roles`

```bash
{
    "Roles": [
        {
            "Path": "/",
            "RoleName": "bucketmaster",
            "RoleId": "AROARZPUZDIKJJZ6OWN27",
            "Arn": "arn:aws:iam::123456789012:role/bucketmaster",
            "CreateDate": "2025-11-26T01:54:01.342203+00:00",
            "AssumeRolePolicyDocument": {
                "Statement": [
                    {
                        "Action": "sts:AssumeRole",
                        "Effect": "Allow",
                        "Principal": {
                            "AWS": "arn:aws:iam::123456789012:user/sir.carrotbane"
                        }
                    }
                ],
                "Version": "2012-10-17"
            },
            "MaxSessionDuration": 3600
        }
    ]
}
```

There's a role named `bucketmaster`, and it can be assumed by `sir.carrotbane`. To check the inline policies.

`aws iam list-role-policies --role-name bucketmaster`

There is one policy assigned to this role. Before checking that policy, let's see if there are any attached policies assigned to the role as well.

`aws iam list-attached-role-policies --role-name bucketmaster`

Looks like we only have the inline policy assigned. Let's see what permissions we can get from the policy.

`aws iam get-role-policy --role-name bucketmaster --policy-name BucketMasterPolicy`

```bash
{
    "RoleName": "bucketmaster",
    "PolicyName": "BucketMasterPolicy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "s3:ListAllMyBuckets"
                ],
                "Effect": "Allow",
                "Resource": "*",
                "Sid": "ListAllBuckets"
            },
            {
                "Action": [
                    "s3:ListBucket"
                ],
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:s3:::easter-secrets-123145",
                    "arn:aws:s3:::bunny-website-645341"
                ],
                "Sid": "ListBuckets"
            },
            {
                "Action": [
                    "s3:GetObject"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:s3:::easter-secrets-123145/*",
                "Sid": "GetObjectsFromEasterSecrets"
            }
        ]
    }
}
```

We can see that the `bucketmaster` role can perform three different actions (ListAllBuckets, ListBucket and GetObject) on some resources of a service named S3. This might just be the breakthrough we were looking for.

**Assuming Role:** To gain privileges assigned by the `bucketmaster` role, we need to assume it. We can use AWS STS to obtain the temporary credentials that enable us to assume this role.

`aws sts assume-role --role-arn arn:aws:iam::123456789012:role/bucketmaster --role-session-name TBFC`

This command will ask STS, the service in charge of AWS security tokens, to generate a temporary set of credentials to assume the `bucketmaster` role. The temporary credentials will be referenced by the session-name "TBFC"

```bash
{
    "Credentials": {
        "AccessKeyId": "ASIA RZPUZDIKDM4AUIJK",
        "SecretAccessKey": "WUzUY46CdgMOLkhuO5llc4G0W92QUaOBNhhzfmSm",
        "SessionToken": "FQoGZXIvYXdzEBYaDdK2KFhRR9GhgoUk9LmzZZjzjJw+r++BFMc7nXyjTE3swUL4ddYyyl47fn6DJeR760L2LxSI+5ur33zUe8b5HwaSqkAbb4xXZdGkvcBv0VWVaLYAMHbbOs7M6WT7ffgcW7Z0bj4I3lB9sN056nuKvcmOG7PQpu/+1wZ2hq1e77/toQKKO+UkhCJU+qMK9iNChMfnuJvFciudTVuyqgG2lhbLK53WWnFxVCHhzjNnCZzZ5QzlBockhcZAq/tVUsp4IVFcz/cOqnT5Xdb5Ovz2Wc9nHOjrRjApIWqNqNb+Saj37le7FZEytuYlpSRtG1QP7n6vdtGPBb1t1ywT2So=",
        "Expiration": "2025-11-26T03:40:11.117460+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROARZPUZDIKJJZ6OWN27:TBFC",
        "Arn": "arn:aws:sts::123456789012:assumed-role/bucketmaster/TBFC"
    },
    "PackedPolicySize": 6
}
```

The output will provide us the credentials we need to assume this role, specifically the `AccessKeyID`, `SecretAccessKey` and `SessionToken`. To be able to use these, run the following commands in the terminal, replacing with the exact credentials that you received on running the `assume-role` command.

```powershell
      user@machine$ export AWS_ACCESS_KEY_ID="ASIAxxxxxxxxxxxx"
user@machine$ export AWS_SECRET_ACCESS_KEY="abcd1234xxxxxxxxxxxx"
user@machine$ export AWS_SESSION_TOKEN="FwoGZXIvYXdzEJr..."
```

Once we have done that, we can officially use the permissions granted by the `bucketmaster` role. To check if you have correctly assumed the role, you can once again run:

`aws sts get-caller-identity`

This time, it should show you are now using the `bucketmaster` role.

**Answer the questions below**

> Apart from GetObject and ListBucket, what other action can be taken by assuming the bucketmaster role?
> 

ListAllMyBuckets

### Task 5 - Grabbing a file from S3

**What Is S3?**

Before we continue, we need to know what exactly is S3. Amazon S3 stands for **Simple Storage Service**. It is an object storage service provided by Amazon Web Services that can store any type of object such as images, documents, logs and backup files. Companies often use S3 to store data for various reasons, such as reference images for their website, documents to be shared with clients, or files used by internal services for internal processing. Any object you store in S3 will be put into a "Bucket". You can think of a bucket as a directory where you can store files, but in the cloud.

**Listing Contents From a Bucket**`aws s3api list-buckets`

There is one interesting bucket in there that references easter.`aws s3api list-objects --bucket easter-secrets-123145`

let's copy the file in this directory to our local machine. This might have a secret message.

`aws s3api get-object --bucket easter-secrets-123145 --key cloud_password.txt cloud_password.txt`

We have successfully infiltrated Sir Carrotbane's S3 bucket and exfiltrated some sensitive data.

**Answer the questions below**

> What are the contents of the cloud_password.txt file?
> 

THM{more_like_sir_cloudbane}

Run the query below in Linux terminal in order to get secret message: I downloaded a specific file (cloud_password.txt) from an S3 bucket (easter-secrets-123145).

`aws s3api get-object --bucket easter-secrets-123145 --key cloud_password.txt cloud_password.txt`
