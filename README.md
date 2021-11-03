# ROSA with Velero

This article describes how to integrate ROSA with Velero in a manner that will leverage short-term credentials using IAM Roles for Service Accounts.

ROSA can be deployed as either a public or private cluster in STS mode as per these instructions:

https://mobb.ninja/docs/rosa/sts/

Do not install the Velero operator (OADP) from Operator Hub as the basic install does not support STS operation. Instead we will download the Velero CLI and install it manually so that we can switch on all necessary features including Restic for backing up and restoring persistent volumes.

The latest Velero CLI (v1.7.0) can be downloaded from this URL:

https://github.com/vmware-tanzu/velero/releases/tag/v1.7.0

