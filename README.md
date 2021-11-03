# ROSA with Velero

This article describes how to integrate ROSA with Velero for backing up user-managed projects. This is separate from the Velero instance that is used by RedHat SRE for backing up managed components. This integration will leverage short-term credentials using IAM Roles for Service Accounts and the default OIDC provider. Doing so avoids needing to create a dedicated Velero user in IAM and storing long-term credentials in a Kubernetes secret.

ROSA can be deployed as either a public or private cluster in STS mode as per these instructions:

https://mobb.ninja/docs/rosa/sts/

Do not install the Velero operator (OADP) from Operator Hub as the basic install method does not support STS operation. Instead we will download the Velero CLI and install the operator manually so that we can enable all necessary features including support for Restic (required for backup/restore of persistent volumes).

The latest Velero CLI (v1.7.0) can be downloaded from this URL:

https://github.com/vmware-tanzu/velero/releases/tag/v1.7.0




