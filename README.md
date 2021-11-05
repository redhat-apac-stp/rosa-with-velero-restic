# ROSA with Velero

This article describes how to integrate ROSA with Velero for backing up user-managed projects. This is separate from the Velero instance that is used by RedHat SRE for backing up managed components. This integration will leverage short-term credentials using IAM Roles for Service Accounts and the default OIDC provider. Doing so avoids needing to create a dedicated Velero user in IAM and storing long-term credentials in a Kubernetes secret.

ROSA can be deployed as either a public or private cluster in STS mode as per these instructions:

https://mobb.ninja/docs/rosa/sts/

Do not install the Velero operator (OADP) from Operator Hub as the basic install method does not support STS operation. Instead we will download the Velero CLI and install the operator manually so that we can enable all necessary features including support for Restic (required for backup/restore of persistent volumes).

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

Create an IAM role named velero-s3-irsa and attach the velero-S3-access policy along with the following trust relationship. Substitute with the name of your OIDC provider accordingly. Importantly use the name of a service account that is not already installed in the cluster.

	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Principal": {
	        "Federated": "arn:aws:iam::<your AWS account>:oidc-provider/<your OIDC provider>"
	      },
	      "Action": "sts:AssumeRoleWithWebIdentity",
	      "Condition": {
	        "StringEquals": {
	          "<your OIDC provider>:sub": "system:serviceaccount:velero:velero-server"
	        }
	      }
	    }
	  ]
	}

You can obtain the identity of your OIDC provider using the rosa describe cluster command.

Create a namespace for Velero with annotations leveraged by Restic, and pre-assign a privileged security context to the Velero service account that will be created during the installation.

	oc new-project velero
	oc annotate namespace velero openshift.io/node-selector=""
	oc adm policy add-scc-to-user privileged -z velero-server -n velero

Prepare a Helm values.yaml file with the following contents:

	configuration:
	  provider: aws
	  backupStorageLocation:
	    bucket: <your S3 bucket>
	    config:
	      region: <your AWS region for the S3 bucket>
	  volumeSnapshotLocation:
	    config:
	      region: <your AWS region for the S3 bucket>
	  defaultVolumesToRestic: true
	#
	serviceAccount:
	  server:
	    create: true
	    name: velero-server
	    annotations:
	      eks.amazonaws.com/role-arn: "arn:aws:iam::<your AWS account>:role/velero-s3-irsa"
	#
	initContainers:
	  - name: velero-plugin-for-aws
	    image: velero/velero-plugin-for-aws:v1.3.0
	    imagePullPolicy: IfNotPresent
	    volumeMounts:
	      - mountPath: /target
		name: plugins
	#
	credentials:
	  useSecret: false
	#
	deployRestic: true	
	#
	restic:
	  privileged: true

Install Velero (v1.7.0) using Helm.

	helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
	helm repo update
	helm install velero vmware-tanzu/velero --namespace velero -f values.yaml
	
Run the following command to verify that velero can access the S3 bucket.

	oc get all -n velero
	oc logs <velero pod> | grep "Backup storage location valid, marking as available"

Test a backup and restore of a persistent volume (e.g., deploy PostgreSQL from a developer template in OpenShift and enter some data). Next backup the namespace with the following YAML.

	apiVersion: velero.io/v1
	kind: Backup
	metadata:
	  name: postgresql-backup-1
	  namespace: velero
	spec:
	  includedNamespaces:
	  - my-postgresql

Check the status of the backup using the following command and confirm Restic was in fact used to backup any persistent volumes.

	oc describe backup postgresql-backup-1

Delete the namespace and resources therein and then restore everything via the following YAML.

	kind: Restore
	metadata:
	  name: postgresql-restore-1
	  namespace: velero
	spec:
	  backupName: postgresql-backup-1
	  includedNamespaces:
	  - my-postgresql

Check the status of the restore using the following command to confirm all items were restored.

	oc describe restore postgresql-restore-1

Confirm the contents of the database were restored in the namespace specified.

***

If you experience any errors or have questions please contact the author at jwilms@redhat.com
