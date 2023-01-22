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

- Install the ODF Operator via an ACM Policy.
  
  ```bash
  kustomize build --enable-alpha-plugins ./noobaa-operator/ | oc create -f -
  ```

- Create the NooBaa Object via an ACM Policy.

  ```bash
  kustomize build --enable-alpha-plugins ./noobaa-object/ | oc create -f -
  ```

- Create the OADP Data Protection Application with Noobaa Storage via an ACM Policy.
  
  ```bash
  kustomize build --enable-alpha-plugins ./oadp-backup | oc create -f -
  ```

## Backup a Simple Application Manifest

- Create a simple deployment
  
  ```bash
  oc create -f ./backup-sample1/deployment.yaml
  ```

- Create a Backup for that deployment
  
  ```bash
  oc create -f ./backup-sample1/backup.yaml
  ```

- Confirm the backup completes before proceeding
  
- Delete the deployment

  ```bash
  oc delete -f ./backup-sample1/deployment.yaml
  ```

- Confirm the deployment is completly deleted.

- Create the restore
  
  ```bash
  oc create -f ./backup-sample1/restore.yaml
  ```

- Confirm the deployment is restored.
