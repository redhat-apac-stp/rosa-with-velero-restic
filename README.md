# ROSA with Velero

This article describes how to integrate ROSA with Velero for backing up user-managed projects. This is separate from the Velero instance that is used by RedHat SRE for backing up managed components. This integration will leverage short-term credentials using IAM Roles for Service Accounts and the default OIDC provider. Doing so avoids needing to create a dedicated Velero user in IAM and storing long-term credentials in a Kubernetes secret.

ROSA can be deployed as either a public or private cluster in STS mode as per these instructions:

https://mobb.ninja/docs/rosa/sts/

Do not install the Velero operator (OADP) from Operator Hub as the basic install method does not support STS operation. Instead we will download the Velero CLI and install the operator manually so that we can enable all necessary features including support for Restic (required for backup/restore of persistent volumes).

The latest Velero CLI (v1.7.0) can be downloaded from this URL:

https://github.com/vmware-tanzu/velero/releases/tag/v1.7.0

Create an S3 bucket in AWS that will be used as the target for all user-managed backups.

	aws s3api create-bucket --bucket velero-`uuid` --region ap-southeast-1 --create-bucket-configuration LocationConstraint=ap-southeast-1

Disable public access for this bucket from the AWS web console.

Create an IAM policy named velero-s3-access with the following permissions. Substitute with the name of your S3 bucket accordingly.

	{
	    "Version": "2012-10-17",
	    "Statement": [
	       {
	            "Effect": "Allow",
	            "Action": [
	                "ec2:DescribeVolumes",
	                "ec2:DescribeSnapshots",
	                "ec2:CreateTags",
	                "ec2:CreateVolume",
	                "ec2:CreateSnapshot",
	                "ec2:DeleteSnapshot"
	            ],
	            "Resource": "*"
	        },			
	        {
	            "Effect": "Allow",
	            "Action": [
	                "s3:GetObject",
	                "s3:DeleteObject",
	                "s3:PutObject",
	                "s3:AbortMultipartUpload",
	                "s3:ListMultipartUploadParts"
	            ],
	            "Resource": [
	                "arn:aws:s3:::<your S3 bucket>/*"
	            ]
	        },
	        {
	            "Effect": "Allow",
	            "Action": [
	                "s3:ListBucket"
	            ],
	            "Resource": [
	                "arn:aws:s3:::<your S3 bucket>"
	            ]
	        }
	    ]
	}

Create an IAM role named velero-s3-irsa and attach the velero-S3-access policy along with the following trust relationship. Substitute with the name of your OIDC provider accordingly.

	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Principal": {
	        "Federated": "arn:aws:iam::635859128837:oidc-provider/<your OIDC provider>"
	      },
	      "Action": "sts:AssumeRoleWithWebIdentity",
	      "Condition": {
	        "StringEquals": {
	          "<your OIDC provider>:sub": "system:serviceaccount:velero:velero"
	        }
	      }
	    }
	  ]
	}

The above trust relationship assumes that Velero will be installed into a namespace velero with a service account of velero. If this is not true substitute your values accordingly for the conditional check. You can obtain the identity of your OIDC provider using the rosa describe cluster and remove the https:// prefix.

Before installing Velero we will setup the environment to address prerequisites for installing Restic that are not addressed by the Velero installer.

	oc new-project velero
	oc annotate namespace velero openshift.io/node-selector=""
	oc adm policy add-scc-to-user privileged -z velero -n velero

Run the Velero installer with the following options set to obtain short-term credentials via STS and integrated support for Restic backup/restore of persistent volumes. Substitute with the name of your S3 bucket and account accordingly.

	velero install \
	--provider aws \
	--plugins velero/velero-plugin-for-aws:v1.2.1 \
	--bucket <your S3 bucket> \
	--backup-location-config region=ap-southeast-1 \
	--snapshot-location-config region=ap-southeast-1 \
	--no-secret \
	--use-restic \
	--default-volumes-to-restic \
	--sa-annotations eks.amazonaws.com/role-arn=arn:aws:iam::<your AWS account>:role/velero-s3-irsa

Inspect all the resources created. These should be failing with a CrashLoopBackoff error.

Apply the following patch to fix this error.

	oc patch ds/restic \
  	--namespace velero \
  	--type json \
  	-p '[{"op":"add","path":"/spec/template/spec/containers/0/securityContext","value": { "privileged": true}}]'

After a few minutes the resources should show as healthy. If not just delete the velero and restic pods.









