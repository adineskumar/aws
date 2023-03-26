# S3 Cross Account Replication with KMS Encryption Enabled:
    There were instances where we need the data from production systems to be leveraged for some business analysis, 
    also some development needs real time data to be mimicked in lower environments for making better applications. 
    There are multiple ways that replicate S3 data from one bucket to another, even across accounts too. 
    We will be listing few of those options.

  1.	Having a lambda that gets triggered on each s3:PutObject action to copy an object.
  2.	Having a lambda that runs “aws s3 sync” command at fixed intervals to copy all the objects.
  3.	Having S3 replication rule, leaving S3 itself taking care or replication by its own.
  
      Option 1 and Option 2 needs additional development efforts, 
      whereas the Option 3 doesn’t require any development to replicate objects.

In this post, we will run through the steps that required to set-up S3 Replication between two accounts with KMS keys enabled in both the source and the target accounts
Let’s consider, we have two accounts “211258069784” and “880161310235” as source and destination accounts respectively. 

|Resources                |Values                                                               |
|------------------------------:|---------------------------------------------------------------------------|
|Source Account	                |211258069784                                                               |
|Source Bucket	                |stats-data-src                                                             |
|Source Bucket Encryption Key	|arn:aws:kms:us-east-1:211258069784:key/1234abcd-12ab-34cd-56ef-1234567890ab|
|Target Account	                |880161310235                                                               |
|Target Bucket	                |stats-data-tgt                                                             |
|Target Bucket Encryption Key	|arn:aws:kms:us-east-1:880161310235:key/0987dcba-09fe-87dc-65ba-ab0987654321|    
    
# S3 Replication rule has the below perquisite.
1.	Both the source and target buckets should be versioned.
2.	Setup a IAM role (s3-replication role) that has the below policies on the source account. Also provide KMS encrypt and decrypt actions on both the source and target accounts KMS keys to the s3-replication IAM role. Both source and target accounts KMS key policies should allow decrypt and encrypt access to this “s3-replication role”.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "GetSourceBucketConfiguration",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::211258069784:role/s3-replication-role"
            },
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation",
                "s3:GetBucketAcl",
                "s3:GetReplicationConfiguration",
                "s3:GetObjectVersionForReplication",
                "s3:GetObjectVersionAcl",
                "s3:GetObjectVersionTagging"
            ],
            "Resource": [
                "arn:aws:s3:::stats-data-src",
                "arn:aws:s3:::stats-data-src/*"
            ]
        },
        {
			"Sid": "ReplicateToDestinationBuckets",
            "Effect": "Allow",
            "Action": [
				"s3:List*",
                "s3:*Object",
                "s3:ReplicateObject",
                "s3:ReplicateDelete",
                "s3:ReplicateTags"
			],
            "Resource": [
                "arn:aws:s3:::stats-data-tgt",
                "arn:aws:s3:::stats-data-tgt/*"
            ]
        },
        {
			"Sid": "PermissionToOverrideBucketOwner",
            "Effect": "Allow",
            "Action": [
				"s3:ObjectOwnerOverrideToBucketOwner"
			],
            "Resource": [
                "arn:aws:s3:::stats-data-tgt/*"
            ]
        },
		{
			"Sid": "KMSEncryptDecryptAccess",
			"Effect": "Allow",
			"Action": [
			  "kms:DescribeKey",
			  "kms:GenerateDataKey",
			  "kms:Decrypt",
			  "kms:Encrypt",
			  "kms:ReEncrypt"
			],
			"Resource": [
			  "arn:aws:kms:us-east-1:211258069784:key/1234abcd-12ab-34cd-56ef-1234567890ab",
			  "arn:aws:kms:us-east-1:880161310235:key/0987dcba-09fe-87dc-65ba-ab0987654321"
			]
		}
    ]
}
```

# Trust Relationship for the s3-replication-role should be as below
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Principal" : {
		"Service" : "s3:amazonaws.com"
	  }
    }
  ]
}
```

3.	Create Replication Rule on the source account with the s3-replication IAM role.
    * Provide the replication rule name.
    * Set status as ‘Enabled’ to make it active.
    * Select “This rule applies to all objects in the bucket” to replicate all the objects in the source bucket. You can choose only files under specific folders to be replicated.
    * Select destination to be a bucket (stats-data-tgt) from the account 880161310235.
    * Select option to change object ownership to destination bucket owner.
    * Select the IAM role as “arn:aws:iam::211258069784:role/s3-replication-role”
    * Select additional replication actions like “RTC – Replication Time Control”, etc., as necessary.

4.	Apply the below bucket policy on the target account.

```json
{
   "Version": "2012-10-17",
   "Id": "PolicyForDestinationBucket",
   "Statement": [
       {
           "Sid": "Permissions on objects and buckets",
           "Effect": "Allow",
           "Principal": {
               "AWS": "arn:aws:iam::211258069784:role/s3-replication-role"
            },
            "Action": [
                "s3:List*",
                "s3:GetBucketVersioning",
                "s3:PutBucketVersioning",
                "s3:ReplicateDelete",
                "s3:ReplicateObject"
            ],
            "Resource": [
                "arn:aws:s3:::stats-data-tgt",
                "arn:aws:s3:::stats-data-tgt/*"
            ]
        },
        {
            "Sid": "Permission to override bucket owner",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::211258069784:root"
            },
            "Action": "s3:ObjectOwnerOverrideToBucketOwner",
            "Resource": "arn:aws:s3:::stats-data-tgt/*"
        }
    ]
}
```

That’s it, you are done.
You can see that from now on, any files that were uploaded to source bucket will be automatically replicated to the target account without any manual intervention.  


Note: Account numbers and KMS keys mentioned here were only used for illustration purposes, those were not actual ARNs.
