# oadp-acm-example

Purpose of this repo is to demo an example of [Red Hat Openshift API for Data Protection/OADP(Velero)](https://docs.openshift.com/container-platform/4.11/backup_and_restore/application_backup_and_restore/oadp-features-plugins.html) with [Red Hat Advanced Cluster Management(ACM)](https://www.redhat.com/en/technologies/management/advanced-cluster-management) running on OpenShift.S3 Storage for this example is provided by Red Hat Openshift Data Foundation via the [Multi Cloud Gatway Component(NooBaa)](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/4.11/html/managing_hybrid_and_multicloud_resources/about-the-multicloud-object-gateway). Repo will provide some examples of simple backup scenarios.

## Prerequisites

- [OpenShift Cluster](https://docs.openshift.com/container-platform/4.9/welcome/index.html) - Version>=4.9
- [oc client](https://docs.openshift.com/container-platform/4.9/cli_reference/openshift_cli/getting-started-cli.html) >= 4.9
- [Red Hat ACM Policy Generator Kustomize Plugin](https://github.com/stolostron/policy-generator-plugin)
- [Red Hat Advanced Cluster Management - ACM](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.0/html-single/install/index#installing) - Version>=2.5
- [User running commands must have subscription-admin privilege](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html-single/applications/index#granting-subscription-admin-privilege).
  - [Sample Solution](https://access.redhat.com/solutions/6010251)
  - [CLI Example](https://github.com/MoOyeg/rhacm-multi-kubernetes-example#attach-subscription-admin-policy-to-your-user-if-necessary)
- Environment "must" have the use of a default dynamic storage classi.e (storageclass.kubernetes.io/is-default-class: true)
- For Backup Example 2 the default storage class "must" be CSI compliant.
  
## Steps to Run

- Create a namespace to hold our ACM policies.

  ```bash
  oc new-project global-policies
  ```

- Add Subscription Admin role to the user running our ACM policies.
  
  ```bash
  oc adm policy add-cluster-role-to-user open-cluster-management:subscription-admin $(oc whoami)
  ```

- Create list of PlacementRules we want ACM to use and can be leveraged by other policies.
  
  ```bash
  oc apply -k ./placementrules/ -n global-policies
  ```

- Install the OADP Operator via an ACM Policy.
  
  ```bash
  kustomize build --enable-alpha-plugins ./oadp-operator/ | oc create -f -
  ```

- Install the ODF Operator via an ACM Policy.Please see prerequisites on default storageclass above before running.
  
  ```bash
  kustomize build --enable-alpha-plugins ./noobaa-operator/ | oc create -f -
  ```

- Create the NooBaa Object via an ACM Policy.Might take a bit before the noobaa object is fully ready.Confirm NooBaa object is ready and oadp-bucket claim is bound.

  ```bash
  kustomize build --enable-alpha-plugins ./noobaa-object/ | oc create -f -
  ```

- Create the OADP Data Protection Application with Noobaa Storage via an ACM Policy.
  
  ```bash
  kustomize build --enable-alpha-plugins ./oadp-backup | oc create -f -
  ```

## Backup Example 1 - Backup a Simple Application Manifest

Example shows the backup and restore of a single deployment. Backup Resource uses labels to select deployment to backed up.

- Create a simple deployment
  
  ```bash
  oc create -f ./backup-sample1/deployment.yaml
  ```

- Create a Backup for that deployment
  
  ```bash
  oc create -f ./backup-sample1/backup.yaml
  ```

- Confirm the backup status is 'Completed' before proceeding
  
  ```bash
  oc get backup backup-sample1 -n openshift-adp -o jsonpath='{.status.phase}'
  ```
  
- Delete the deployment

  ```bash
  oc delete -f ./backup-sample1/deployment.yaml
  ```

- Confirm the deployment is completly deleted.
  
  ```bash
  oc get deployment backup-sample1 -n default
  ```

- Create the restore
  
  ```bash
  oc create -f ./backup-sample1/restore.yaml
  ```

- Confirm the deployment is restored.

  ```bash
  oc get deployment backup-sample1 -n default
  ```

## Backup Example 2 - Backup an Application Manifest and PV with Restic

Example shows the backup and restore of a deployment and it's data from a pvc via the use of restic and [OADP data mover feature](https://docs.openshift.com/container-platform/4.11/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/backing-up-applications.html).Example will simulate state by writing a date to a pvc and restoring the volume with the file also having the new date of when volume was restored.

- Install the Volsync Operator which is required for restic use.
  
  ```bash
  kustomize build --enable-alpha-plugins ./volsync-operator | oc create -f -
  ```

- Patch the OADP DataProtectionApplication to allow restic.

  ```bash
  kustomize build --enable-alpha-plugins ./backup-sample2 | oc create -f -
  ```

- Confirm the volsync operator and DataProtectionApplication are available.

- Give RBAC permissions to serviceaccount to allow writing to pod volume.
  
  ```bash
  oc create -f ./backup-sample2/rbac.yaml
  ```

- Create PVC for deployment usage.Please see prerequisites on default storageclass above before running.

  ```bash
  oc create -f ./backup-sample2/pvc.yaml
  ```

- Create deployment for backup example.

  ```bash
  oc create -f ./backup-sample2/deployment.yaml
  ```

- Confirm deployment wrote date to volume file before continuing.
  
  ```bash
  oc exec deployment/backup-sample2 -n default -c container  -- cat /root/backup-sample2/backup
  ```

- Create backup for deployment and data.
  
  ```bash
  oc create -f ./backup-sample2/backup.yaml
  ```

- Confirm the backup status is 'Completed' before proceeding
  
  ```bash
  oc get backup backup-sample2 -n openshift-adp -o jsonpath='{.status.phase}'
  ```

- Delete Deployment to test restore.
  
  ```bash
  oc delete -f ./backup-sample2/deployment.yaml
  ```

- Delete PVC to test restore.
  
  ```bash
  oc delete -f ./backup-sample2/pvc.yaml
  ```

- Confirm the deployment is completly deleted.
  
  ```bash
  oc get deployment backup-sample2 -n default
  ```

- Confirm the PVC is completly deleted.
  
  ```bash
  oc get deployment pvc -n default
  ```

- Restore data and Manifests.
  
  ```bash
  oc create -f ./backup-sample2/restore.yaml
  ```

- Confirm the restore status is 'Completed' before proceeding
  
  ```bash
  oc get restore backup-sample2-restore -n openshift-adp -o jsonpath='{.status.phase}'
  ```

- Confirm restore deployment now has the date before and after the restore.
  
  ```bash
  oc exec deployment/backup-sample2 -n default -c container  -- cat /root/backup-sample2/backup
  ```
