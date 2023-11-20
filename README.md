# K10-RestoreApp-ArgoCD-Ansible
***Status:** Work-in-progress. Please create issues or pull requests if you have ideas for improvement.*

# **Kasten K10 full automated Disaster Recovery with Ansible and ArgoCD**
Example of using the Kasten K10 Disaster Recovery feature with Ansible and ArgoCD to automate the Disaster Recovery Plan for Kubernetes workloads.

## Summary
This projects demostrates the process of recovering a Kasten K10 instance and all the protected Kubernetes workloads (applications) after a disaster ocurrs on Kubernetes cluster.  

All the automation is done using Ansible playbooks, ArgoCD, and leveraging the [Kasten K10 API](https://docs.kasten.io/latest/api/cli.html). In this project, we will use Ansible to deploy Kasten and manage Kasten configuration, ArgoCD to re-deploy the applications in the DR cluster, and then we will use Kasten to restore the application's data.

## Disclaimer
This project is an example of an deployment and meant to be used for testing and learning purposes only. Do not use in production. 


# Table of Contents

1. [Getting started](#Getting-started)
2. [Prerequisites](#Prerequisites)
3. [Recovering K10 and all Kubernetes Applications From a Disaster](#Recovering-K10-and-all-Kubernetes-Applications-From-a-Disaster)
4. [File structure and deployment workflow](#File-structure-and-deployment-workflow)
5. [Variables](#Variables)
6. [Ansible Vault](#Ansible-Vault)

# Getting started

K10 Disaster Recovery (DR) aims to protect K10 from the underlying infrastructure failures. In particular, this feature provides the ability to recover the K10 platform in case of a variety of disasters such as the accidental deletion of K10, failure of underlying storage that K10 uses for its catalog, or even the accidental destruction of the Kubernetes cluster on which K10 is deployed.

K10 enables Disaster Recovery with the help of an internal policy to backup its own data stores and store these in an object storage bucket or an NFS file storage location configured using a Location Profile.

## Prerequisites

To run this project you need to have some software installed and configured: 
1. A workstation with the next tools installed:
	- Kubectl
	- Kubernetes Collection for Ansible
	- Helm
    - (If restoring in AWS EKS) aws-cli, aws-iam-authenticator and a AWS named profile.
	- (If restoring in Azure AKS) azure cli 
	- (If restoring in Google GKE) gcloud CLI and gke-gcloud-auth-plugin for use with kubectl 
1. A working [Ansible installation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).
1. A working clean Kubernetes cluster to be used to restore Kasten K10 and all applications.  In this project we will provide the process and the Ansible Playbooks to restore Kasten K10 and all kubernetes applications in **AWS EKS, Azure AKS and Google Cloud GKE**.
1. The Production Kubernetes cluster (cluster to be recovered) must have been using [Kasten K10](https://docs.kasten.io/latest/install/index.html) to backup all applications and cluster-wide resources.
	- Kasten K10 Disaster Recovery must have been enabled in the Production cluster.
	- A [Location Profile](https://docs.kasten.io/latest/usage/configuration.html) must be provided to store the Kasten configuration backups.
	- During the enabling process it's necessary to provide a passphrase for encrypting the snapshot data. The passphrase can be provided directly or provided from a HashiCorp Vault instance if K10 is configured to access Vault.   This passphrase **will be required** afterward to restore the Kasten configuration.
	- Once K10 Disaster Recovery is enabled, a  confirmation message with the cluster ID will be displayed when Disaster Recovery is enabled. Save the cluster ID safely, **it is required to recover K10 from a disaster**.
1. Kasten K10 Disaster Recovery Policy and all Policies protecting Kubernetes applications must have been run at least once to provide the required restore points for this project.

**IMPORTANT**: After enabling K10 Disaster Recovery, it is essential that you copy and save the following to successfully recover K10 from a disaster:
1. The cluster ID displayed on the disaster recovery page
1. The Disaster Recovery passphrase provided above
1. The credentials and object storage bucket or the NFS file storage information (used in the Location Profile configuration above)

Without this information, K10 Disaster Recovery will not be possible.

## Recovering K10 and all Kubernetes Applications From a Disaster
Recovering from a K10 backup involves the following sequence of actions:

1. Install a fresh K10 instance
1. Create a Kubernetes Secret, k10-dr-secret, using the passphrase provided while enabling Disaster Recovery.
1. Provide bucket information and credentials for the object storage location where previous K10 backups are stored
1. Restoring the K10 backup
1. Restoring the cluster-wide resources from backups
1. Restoring the applications from backups

All these steps will be automated using the Ansible playbooks provided in this project.

**Kasten DR with ArgoCD and Ansible playbooks in Video:**

[![Alt text](https://img.youtube.com/vi/NXuGvU0VivQ/0.jpg)](https://www.youtube.com/watch?v=NXuGvU0VivQ)


## File structure and deployment workflow

The Deployment consists thre main playbook triggering multible tasks:
1. dr_cluster/01_k10_install.yaml: 
	- This playbook installs a fresh Kasten K10 instance in the target Kubernetes cluster.
1. dr_cluster/02_k10_dr_restore.yaml:
	- This playbook creates the  Kubernetes Secret "k10-dr-secret" using the passphrase provided while enabling Disaster Recovery
	- Then, this playbook creates a Location Profile using the provided Object Storage, which of course MUST contain the Kasten configuration backup (this is the Location Profile used when enabling the Kasten K10 Disaster Recovery in the Production Kubernetes cluster).  
	- Finally this playbook restores the Kasten K10 configuration from the Location Profile created.
1. dr_cluster/03_k10_restoreapps.yaml: 
	- This playbook look for the most recent restore points available to restore the cluster-wide resources and each application.
	- Then the playbook restore the cluster-wide resources from the most recent restore point found in previous step.
	- Next the playbook creates a namespace for every application to be restored.
	- Finally the playbook restore every application from the most recent restore point found in previous step.

**Important**: The playbooks must be run in the mentioned order in order to make the recovery process works.

**NOTE**: For testing purposes, the prod_cluster directory contains Ansible playbooks to deploy Kasten K10 in Production environment, configure the Location Profiles, and create the Policies to backup all Kubernetes applications.  The goal in this project is to provide the Ansible playbooks to deploy Kasten in Production cluster, and then use Ansible and ArgoCD to recover all the applications in another Kubernetes cluster (DR)


## Variables
Some deployment variables must be set into the vars files.  Alter the parameters according to your needs:

[For Production Cluster](prod_cluster/vars/vars.yaml) "vars.yaml" in the prod_cluster/vars folder.
[For DR Cluster](dr_cluster/vars/vars.yaml) "vars.yaml" in the dr_cluster/vars folder.

| Name               | Type     | Default value       | Description                                                                                    |
| ------------------ | -------- | ------------------- | ---------------------------------------------------------------------------------------------- |
| `k10_infra`        | `string` | `AWSEKS`            | Choose between AWSEKS, AZAKS and GCGKE                                                         |
| `storageclass`     | `string` | `standard-rwo`      | Name of the Storage Class in the target Kubernetes cluster for importing applications          |
| `dr_bucket`        | `string` | `k10immutable`      | Bucket to be used as Location Profile for K10 Disaster Recovery                                |
| `drloc_prof`       | `string` | `k10immutable`      | Location Profile name for K10 Disaster Recovery                                                |
| `drloc_prof_type`  | `string` | `awss3`             | Bucket type for K10 Disaster Recovery Location Profile  Options: awss3/azblob/gcsa             |
| `loc_prof`         | `string` | `s3-k10`            | Location Profile name to be used to backup all applications.  Used when Policies are created.  |
| `region`           | `string` | `eu-west-3`         | Bucket region                                                                                  |
| `company`          | `string` | `Demolab`           | Company name for EULA                                                                          |
| `email`            | `string` | `pcerda@demolab.com`| E-Mail address for EULA                                                                        |


## Ansible Vault
Some additional variables must be set in an Ansible Vault to keep the sensitive data required for authentication.

[For Production Cluster](prod_cluster/vars/vault_vars.yaml) "vault_vars.yaml" in the prod_cluster/vars folder.
[For DR Cluster](dr_cluster/vars/vault_vars.yaml) "vault_vars.yaml" in the dr_cluster/vars folder.

| Name                    | Type     | Default value          | Description                                                                                                            |
| ----------------------- | -------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `aws_access_key_id`     | `string` | `aaaaaaaaa`            | AWS Access Key to add AWS S3 bucket                                                                                    |
| `aws_secret_access_key` | `string` | `aaaaaaaaa`            | AWS Secret Access Key to add AWS S3 bucket                                                                             |
| `tenantID`              | `string` | `aa-431b-b-a7-1012`    | Azure Tenant ID to add Azure Blob                                                                                      |
| `azureclientID`         | `string` | `aaaaaaaaa`            | Azure Client ID to add Azure Blob                                                                                      |
| `Azureclientsecret`     | `string` | `aaaaaaaaa`            | Azure Client Secret to add Azure Blob                                                                                  |
| `azure_storage_key`     | `string` | `aaaaaaaaa`            | Azure Storage Access Key to add Azure Blob                                                                             |
| `azure_storage_env`     | `string` | `AzureCloud`           | AzureCloud is the default in Azure.  More info in https://docs.kasten.io/latest/usage/configuration.html#azure-storage |
| `project_id`            | `string` | `gcloud-project-id`    | Google Project ID to add Google Cloud Storage Account                                                                  |
| `LOGIN`                 | `string` | `admin:$apr1$aa.pY/bb/`| For K10 Basic Authentication.  Use 'htpasswd -n admin' and provide a password to autenticate to Kasten K10             |

**NOTE**: It is recommended to use Ansible Vaults to keep this data instead of using just a text file, considering all the sensitive data to be kept here.
