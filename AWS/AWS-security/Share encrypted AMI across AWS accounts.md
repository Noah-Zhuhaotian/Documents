AWS has made sharing encrypted AMI cross accounts a bit easier now, check this out [Share encrypted AMIs across accounts to launch instances in a single step](https://aws.amazon.com/about-aws/whats-new/2019/05/share-encrypted-amis-across-accounts-to-launch-instances-in-a-single-step/)

Here is a sample of how to share encrypted AMI across accounts and launch an instance from it: [How to share encrypted AMIs across accounts to launch encrypted EC2 instances](https://aws.amazon.com/blogs/security/how-to-share-encrypted-amis-across-accounts-to-launch-encrypted-ec2-instances/)

If you need to run autoscaling group from the encrypted AMI, it requires a few extra steps. Mostly it is about KMS grant. <br/>
1. Determine which service-linked role to use for this Auto Scaling group.
2. Allow the Auto Scaling group account access to the CMK.
3. Define an IAM user or role in the Auto Scaling group account that can create a grant.
4. Create a grant to the CMK with the service-linked role as the grantee principal.
5. Update the Auto Scaling group to use the service-linked role.<br/><br/>
It is easier to explain it with a example. And I use Packer to bake the encrypted AMI in the example. Lets call the source account `11111111`, target account `22222222` <br/><br/>

**Step one**: Setup a key (e.g `arn:aws:kms:ap-southeast-2:11111111:key/d587cf57-80b5-4d64-a2f9-xxxxxxxxxx`) that is dedicated for ami encryption, and it has to be symmetric key. Lets call it ami-sharing-key in source account (e.g `11111111`)

**Step two**: In the source account `11111111`, edit the KMS policy to allow a user or role from target account `22222222` to grant the key. In the sample policy, I grant the role `arn:aws:iam::22222222:role/kms-admin-22222222` kms admin permissions. It may be too much to give the key admin access. I will review it later to make the permission just enough.
```json
    {
         "Sid": "Allow access for Key Administrators",
         "Effect": "Allow",
         "Principal": {
             "AWS": [
                 "arn:aws:iam::11111111:role/kms-admin-11111111",
                 "arn:aws:iam::22222222:role/kms-admin-22222222"
             ]
         },
         "Action": [
             "kms:Create*",
             "kms:Describe*",
             "kms:Enable*",
             "kms:List*",
             "kms:Put*",
             "kms:Update*",
             "kms:Revoke*",
             "kms:Disable*",
             "kms:Get*",
             "kms:Delete*",
             "kms:TagResource",
             "kms:UntagResource",
             "kms:ScheduleKeyDeletion",
             "kms:CancelKeyDeletion"
         ],
         "Resource": "*"
     },
```
**Step three**: Assume the `kms-admin-22222222` role in target account `22222222` then run the following command to create grant for the cmk to the AWS service role for autoscaling.
```
aws kms create-grant --key-id arn:aws:kms:ap-southeast-2:11111111:key/d587cf57-80b5-4d64-a2f9-xxxxxxxxxx --grantee-principal arn:aws:iam::22222222:role/aws-service-role/autoscaling.amazonaws.com/AWSServiceRoleForAutoScaling --operations "Encrypt" "Decrypt" "ReEncryptFrom" "ReEncryptTo" "GenerateDataKey" "GenerateDataKeyWithoutPlaintext" "DescribeKey"
```
**Step four**: Update the packer template file to add the following two parameters in the builder section to enforce the AMI encryption in source account.
```json
   "encrypt_boot": true,
   "kms_key_id": "arn:aws:kms:ap-southeast-2:11111111:key/d587cf57-80b5-4d64-a2f9-xxxxxxxxxx",
```
**Step five**: Bake a new AMI with the above packer template in the source account `11111111`, then share the encrypted AMI with a target account `22222222` <br/><br/>


> [!NOTE]
> 1. Packer takes a longer to build an encrypted AMI, under the hood it bakes an unencrypted AMI first, then make a copy of that AMI and encrypts the new AMI.
> 2. If the target account uses different key for EBS encryption, then it is all good. The ami-sharing-key is only used for decrypting the encrypted AMI/snapshot.

References:

https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html#key-policy-default
https://docs.aws.amazon.com/cli/latest/reference/kms/create-grant.html
https://forums.aws.amazon.com/thread.jspa?threadID=277523
https://stackoverflow.com/questions/55859757/using-encrypted-ebs-volumes-in-auto-scaling-groups-with-cmk-owned-by-a-different
