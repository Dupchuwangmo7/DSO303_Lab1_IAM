# Lab 01 — IAM Practical Report

## Student Information

- **Student Name:** Dupchu Wangmo
- **Student Number:** 02230282
- **Lab:** Lab 01 — IAM
- **Environment:** Floci AWS-compatible local environment
- **Date:** 21 August 2026

## 1. Introduction

This practical focused on understanding Identity and Access Management (IAM) concepts using the Floci AWS-compatible environment. The practical involved creating and configuring IAM users, groups and policies, assigning permissions through group membership, configuring AWS CLI profiles, and testing the resulting permissions against an S3 test bucket.

The main objective of my work was to demonstrate that permissions can be assigned to a group rather than directly to an individual user. I also tested the difference between permissions that are allowed and permissions that are not granted by an IAM policy.

## 2. Objectives

The objectives of the practical work completed were:

- Create and configure an IAM user.
- Create an IAM group.
- Add the IAM user to the group.
- Create and attach a read-only S3 policy to the group.
- Verify the relationship between the user, group and policy.
- Configure an AWS CLI profile for the IAM user.
- Verify the authenticated IAM identity using AWS STS.
- Create and verify an S3 test bucket.
- Test the user's ability to list objects in the bucket.
- Test whether `s3:PutObject` is allowed.
- Use IAM policy simulation to verify the effective permission.
- Investigate unexpected S3 behaviour in the Floci environment.

## 3. Environment

The practical was performed on a MacBook using the AWS CLI and the Floci local AWS-compatible environment.

The AWS CLI profiles used during the practical were:

- `floci`
- `lab-developer`

The `floci` profile was used for administrative IAM operations, while `lab-developer` was used to test the permissions of the IAM user.

## 4. IAM Group and User Structure

The IAM group used for the practical was:

- `developers`

The IAM user was:

- `lab-developer`

The final relationship was:

```
developers
│
└── lab-developer
```

The `DeveloperReadOnly` managed policy was attached to the `developers` group.

Therefore, the user inherited the permissions from the group rather than having a policy attached directly to the user.

The relationship was verified using IAM commands.

## 5. DeveloperReadOnly Policy

The `DeveloperReadOnly` policy was inspected using:

```bash
aws iam get-policy --policy-arn arn:aws:iam::000000000000:policy/DeveloperReadOnly --profile floci
```

The policy was found to use version:

- `v1`

The actual policy document was then inspected using:

```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::000000000000:policy/DeveloperReadOnly \
  --version-id v1 \
  --profile floci
```

The policy contained the following permissions:

- `s3:ListAllMyBuckets`
- `s3:GetBucketLocation`
- `s3:ListBucket`

The policy did not contain:

- `s3:PutObject`

Therefore, the policy was configured as a read/list-only policy.

## 6. Verification of IAM Configuration

The managed policy attached to the group was verified using:

```bash
aws iam list-attached-group-policies \
  --group-name developers \
  --profile floci
```

The result showed:

```
DeveloperReadOnly
```

The user was also checked for direct policies:

```bash
aws iam list-user-policies \
  --user-name lab-developer \
  --profile floci
```

The result was:

```json
{
    "PolicyNames": []
}
```

This confirmed that the user had no inline policy attached directly.

The user was also checked for directly attached managed policies:

```bash
aws iam list-attached-user-policies \
  --user-name lab-developer \
  --profile floci
```

The result was:

```json
{
    "AttachedPolicies": []
}
```

The group was also checked for inline policies:

```bash
aws iam list-group-policies \
  --group-name developers \
  --profile floci
```

The result was:

```json
{
    "PolicyNames": []
}
```

These checks confirmed that the `DeveloperReadOnly` group policy was the intended source of the user's read/list permissions.

## 7. AWS CLI Developer Profile

Initially, the `lab-developer` CLI profile was incorrectly using root credentials. The problem was identified using:

```bash
aws sts get-caller-identity --profile lab-developer
```

Initially, the result showed:

```
arn:aws:iam::000000000000:root
```

The IAM user was then checked for access keys:

```bash
aws iam list-access-keys \
  --user-name lab-developer \
  --profile floci
```

The result was:

```json
{
    "AccessKeyMetadata": []
}
```

This showed that the user had no access keys.

A new access key was subsequently created for `lab-developer`, and the AWS CLI profile was configured with the new credentials.

The identity was then successfully verified using:

```bash
aws sts get-caller-identity --profile lab-developer
```

The final result was:

```json
{
    "UserId": "000000000000",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:user/lab-developer"
}
```

This confirmed that the AWS CLI was finally operating as the intended IAM user rather than the root identity.

## 8. S3 Test Bucket

A test S3 bucket was created using:

```bash
aws s3api create-bucket \
  --bucket usms-iam-test \
  --profile floci
```

The command returned:

```json
{
    "Location": "/usms-iam-test"
}
```

The bucket was then verified using:

```bash
aws s3api list-buckets --profile floci
```

The bucket appeared as:

```
usms-iam-test
```

The bucket was created specifically for testing IAM permissions.

## 9. Test Object

A small test file was created using:

```bash
echo "IAM lab test" > /tmp/iam-test.txt
```

The file was uploaded using the administrative `floci` profile:

```bash
aws s3 cp /tmp/iam-test.txt \
  s3://usms-iam-test/iam-test.txt \
  --profile floci
```

The upload succeeded.

The object was then verified using:

```bash
aws s3api head-object \
  --bucket usms-iam-test \
  --key iam-test.txt \
  --profile floci
```

The object was successfully found and had a content length of 13 bytes.

## 10. Testing the Developer's List Permission

The S3 list permission was tested using the actual `lab-developer` credentials:

```bash
aws s3api list-objects-v2 \
  --bucket usms-iam-test \
  --profile lab-developer
```

The command successfully returned:

```
Key: iam-test.txt
Size: 13
StorageClass: STANDARD
```

This demonstrated that `lab-developer` could successfully list objects in the bucket.

Therefore:

- `s3:ListBucket` = **Allowed**

This was consistent with the `DeveloperReadOnly` policy.

## 11. Testing PutObject Permission

The next test attempted to upload an object using the `lab-developer` profile:

```bash
aws s3 cp /tmp/iam-test.txt \
  s3://usms-iam-test/developer-test.txt \
  --profile lab-developer
```

Unexpectedly, the upload succeeded.

This was different from the expected IAM behaviour because the `DeveloperReadOnly` policy does not grant:

- `s3:PutObject`

The created object was confirmed using:

```bash
aws s3api head-object \
  --bucket usms-iam-test \
  --key developer-test.txt \
  --profile lab-developer
```

The object existed successfully.

## 12. Investigation of the Unexpected Upload

Because the upload succeeded, I investigated the possible sources of the PutObject permission.

The following checks were performed.

**User inline policy**

```bash
aws iam list-user-policies \
  --user-name lab-developer \
  --profile floci
```

Result:

```
PolicyNames: []
```

**User managed policies**

```bash
aws iam list-attached-user-policies \
  --user-name lab-developer \
  --profile floci
```

Result:

```
AttachedPolicies: []
```

**Group inline policies**

```bash
aws iam list-group-policies \
  --group-name developers \
  --profile floci
```

Result:

```
PolicyNames: []
```

**Bucket policy**

```bash
aws s3api get-bucket-policy \
  --bucket usms-iam-test \
  --profile floci
```

Result:

```
NoSuchBucketPolicy
```

Therefore, no bucket policy existed.

## 13. IAM Policy Simulation

To determine the actual IAM decision independently of the S3 data-plane behaviour, the `s3:PutObject` action was tested using IAM policy simulation.

The following command was used:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::000000000000:user/lab-developer \
  --action-names s3:PutObject \
  --resource-arns arn:aws:s3:::usms-iam-test/iam-test.txt \
  --profile floci
```

The result was:

```json
{
    "EvaluationResults": [
        {
            "EvalActionName": "s3:PutObject",
            "EvalResourceName": "arn:aws:s3:::usms-iam-test/iam-test.txt",
            "EvalDecision": "implicitDeny",
            "MatchedStatements": [],
            "MissingContextValues": []
        }
    ]
}
```

The important result was:

- `EvalDecision: implicitDeny`

This confirms that the IAM policy evaluation did not grant `s3:PutObject` to `lab-developer`.

## 14. Discussion

The practical demonstrated an important difference between IAM policy evaluation and the behaviour of the local Floci S3 service.

The IAM policy simulator returned:

```
implicitDeny
```

for `s3:PutObject`.

This is consistent with the `DeveloperReadOnly` policy because the policy only grants:

- `s3:ListAllMyBuckets`
- `s3:GetBucketLocation`
- `s3:ListBucket`

However, the actual S3 upload was accepted by the local environment.

The unexpected upload was therefore not used as evidence that the user had PutObject permission. Instead, the IAM policy simulator was used to establish the actual IAM authorization decision.

This investigation also demonstrated the importance of checking the complete IAM configuration rather than assuming that a successful service request always means the corresponding IAM action has been granted.

## 16. Problems Encountered and Solutions

### Problem 1 — Developer profile authenticated as root

Initially:

```
Arn: arn:aws:iam::000000000000:root
```

The problem was caused by the CLI profile using root credentials.

**Solution:** A new access key was created for `lab-developer` and the AWS CLI profile was reconfigured.

The identity then correctly became:

```
arn:aws:iam::000000000000:user/lab-developer
```

### Problem 2 — `lab-developer` had no access key

The following command showed no access keys:

```bash
aws iam list-access-keys \
  --user-name lab-developer \
  --profile floci
```

**Solution:** A new access key was created for the user.

### Problem 3 — S3 upload unexpectedly succeeded

The IAM policy did not contain `s3:PutObject`, but the S3 upload still succeeded.

**Solution:** The policy, user policies, group policies and bucket policy were inspected. IAM policy simulation was then used to determine the actual authorization decision.

The simulator returned:

```
implicitDeny
```

## 17. Conclusion

This practical exercise helped me gain practical knowledge on IAM users, groups, managed policies, AWS CLI profile and S3 permissions.

The last configuration managed to show inheritance of group based permissions. The `lab-developer` user belonged to the `developers` group, which was associated with the `DeveloperReadOnly` policy. That policy allowed S3 listing actions, but it did not provide the `s3:PutObject` permission.

The AWS CLI identity was switched successfully from the root identity to `lab-developer`, and the developer managed to list objects in the test S3 bucket. IAM policy simulation showed that `s3:PutObject` gave an `implicitDeny`.

One of the most notable things about this practical exercise was the limitation of the local Floci environment. S3 upload was accepted despite IAM policy simulation denying `s3:PutObject`. It shows the importance of policy simulation and IAM inspection when working in a local AWS compatible environment.