# MariaDB-Runbook
## Introduction and Considerations 
### This Runbook’s Objectives
Currently TileDB Enterprise requires a MariaDB instance for account creation and token storage. Most clouds do not support a fully managed MariaDB instance, and on prem customers need a reliable way to deploy a resilient and reliable MariaDB instance. This runbook seeks to provide a recommended configuration of MariaDB using the MariaDB operator and Helm charts for customers seeking to run TileDB Enterprise within their environments.

### Words of Caution Around Production and Support 
This runbook was designed to help customers who do not wish to levarege a supported MariaDB deployment. This runbook is not meant to be the be all and end all guide to running MariaDB. 
The TileDB team **DOES NOT** guarantee **ANY** level of support for **ANY** deployed MariaDB instance or this runbook in general. We highly recommend using a managed MariaDB solution such as [Amazon RDS for MariaDB](https://aws.amazon.com/rds/mariadb/) or reaching out to the [MariaDB team directly](https://mariadb.com/services/technical-support-services/) for support. Should teams have database expertise in house, they can opt into using this runbook to deploy a MariaDB instance using the [MariaDB Operator](https://github.com/mariadb-operator/mariadb-operator) with [community support](https://github.com/mariadb-operator/mariadb-operator?tab=readme-ov-file#get-in-touch). We highly recommend teams seeking to leverage the MariaDB Operator have at the very least basic Kubernetes admin skills and foundational knowledge of the environment they wish to deploy their MariaDB instance to. The TileDB team **DOES NOT** gurantee **ANY** level of support for **ANY** deployed MariaDB instance or this runbook in general. By using this runbook you are accepting the level of risk that comes with deploying, manaaging, and supporting an OpenSource solution.
### Words of Caution Around the TileDB Enterprise Helm Chart MariaDB 
For testing purpose, the TileDB enterprise chart provides a minimal MariaDB deployment out of the box should a user opt into it via the Helm chart. We **DO NOT** support said MariaDB deployment and **HIGHLY** discourage using the provided test/dev MariaDB instance in production. Deploying TileDB Enterprise with the test/dev MariaDB is **NOT** considered a fully supported/deployed TileDB Enterprise instance.
### Clairification on Serverless SQL
MariaDB is used for TileDB's [serverless SQL](https://docs.tiledb.com/cloud/concepts/tiledb-cloud-internals/serverless-sql) feature. The MariaDB instance deployed with this runbook is not the serverless SQL instance, but a stateless MariaDB invocation using the MyTile(https://github.com/TileDB-Inc/TileDB-MariaDB) storage engine.

## Deploying and Configuring the MariaDB Operator
### Deployment Considerations and Outage Impacts
MariaDB currently hosts user information and potentially credentials depending on how you authenticate to cloud services. Therefore, an outage of MariaDB will create login issues and may impact workflows. The risk of data loss (i.e. TileDB data) is unlikely due to data being stored on object, but a MariaDB outage is considered a high impact outage since users cannot access their development environments and data via TileDB Enterprise. Therefore, it is recommended to encrypt your MariaDB instances, deploy multiple replicas, and to back up your MariaDB configurations and database. The frequency of backups and your number of replicas depend on your recovery time objectives (RTOs) and recovery point objectives (RPOs). A highly available deployment of MariadB would consist of deploying 3 replicas of MariaDB with node [anti-affinity rules](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) across zones. A properly backedup MariaDB instance may consist of frequent backups every 10 minutes. The backups should not significantly impact performance. In general, MariaDB performance is not a bottleneck for TileDB Enterprise workflows and does not require much optimization for TileDB Enterprise consumption.

### Requirements
* A Kubernetes Cluster that can support a [TileDB installation](https://docs.tiledb.com/enterprise/installation#kubernetes). This runbook assumes end users will be deploying MariaDB alongside their Enterprise TileDB deployments. Exposing and leveraging an external MariaDB is out the this documents scope.  
* [Kubectl access to the cluster](https://kubernetes.io/docs/reference/kubectl/)
* [Helm](https://helm.sh/)
* A text editor of your choice 


### Deployment Guide
### Clone tileDB MariaDB Runbook Repo
To clone the repo, run `git clone` from your commandline while referencing this repo. Then navigate to the newly created `MariaDB-Runbook` directory.
### Deploying the MariaDB Operator 
The MariaDB operator allows us to run and operate MariaDB in a cloud native way. Declaratively manage your MariaDB using Kubernetes [CRDs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) rather than imperative commands. This follows the standard operator pattern for Kubernetes. You can learn more about that [here](https://www.cncf.io/wp-content/uploads/2021/07/CNCF_Operator_WhitePaper.pdf).

You can choose to use a values file to configure the Helm chart for the MariaDB operator. We opted in to passing the values via command line. A more complete list can be found [here](https://artifacthub.io/packages/helm/mariadb-operator/mariadb-operator) but is not required. The Kubernetes user creating the MariaDB instance must be able to configure RBAC on the Kubernetes cluster including [ClusterRoles and ClusterRoleBindings](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). I.E. the kubeconfig user (if installing from a local machine) must have the appropriate privileges to install the mariaDB operator.

To add the MariaDB Operator Helm chart, run the following:
`helm repo add mariadb-operator https://mariadb-operator.github.io/mariadb-operator`
Then, to install the MariadDB Operator Helm chart , run:
`helm install mariadb-operator mariadb-operator/mariadb-operator  --set ha.enabled=true -n tiledb-cloud`
You should see an output similar to below.

![Installed operator](/images/operator_install.png "Installed Operator").

Double check the deployment is ready to go by running 
`Kubectl get pods -n tiledb-cloud` and confirming you see three pods with **mariadb-operator-** prefixes.

If you see no pending pods, you are ready to configure and deploy your MariaDB instance. 

## Deploying and Configuring Your MariaDB Instnace 
As previously mentioned, MariaDB and supporting configurations can be deployed via Custom Resource Definitions.  Our MariaDB instance Custom Resource Definitions are in the **/initialize** directory of the MariaDB Operator repo you just cloned. Navigate to **/initialize** to view, edit, and apply the manifests in order to configure your MariaDB instance.

Within that directory you should see the following files:

* tiledb-root-secret.yml
* tiledb-secret.yml
* database.yml 
* grant.yml 
* mariadb.yml 
* user.yml

Let’s dive into and configure these files one at a time.
###  tiledb-secret.yml and tiledb-root-secret.yml
 *tiledb-secret.yml* and *tiledb-root-secret.yml* contain [secrets](https://kubernetes.io/docs/concepts/configuration/secret/) that will provision a password and a root-password for our MariaDB database. You can edit these files and set  these values to whatever you’d like, but make sure they are base64 encoded in the files before you apply them to the cluster. 

#### Base64 Encoding and Decoding
Let’s quickly walkthrough base64 encoding and decoding. Let's say you open and read the *tiledb-root-secret.yml* file, and notice a root-password key with the value VTkxb055bi1FdSljSjNwMA==. This value is already base64 encoded. 
To decoded it, you would run:
`echo VTkxb055bi1FdSljSjNwMA== | base64 -d` from a BASH commandline of your choice. The output should be **U91oNyn-Eu)cJ3p0**. To re-encode the value (or encode a different value to configure your database with a different root password) you can run `echo <your desired password> | base64` from your command line. In our case it would be, `echo  U91oNyn-Eu)cJ3p0 | base64` 
Note: you may need to escape special characters
The output should be your base64 encoded value you can copy and paste into the **tiledb-root-secret.yml** or **tiledbd-secret.yml** files.

### mariadb.yml 
The **mariadb.yml** will create our `MariaDB` instance named *tile-mariadb*. This MariaDB instance will be referenced by all the other manifests and use the credentials secrets we created. Once created, to view the status of this instance, you can run `kubectl get mariadb -n tiledb-cloud`.

### grant.yml
**grant.yml** will grant our *tiledb* `user` we will create with the **user.yml** manifest access to the `tile-rest` database. Once created, you can view grants with `kubectl get grants -n tiledb-cloud`.

### user.yml
 **user.yml** will use the secret we create with the user's credentials. The default **user.yml** creates a `tiledb` user mapped to the default `tile-mariadb` MariaDB instance.
 
## High Availability Overview   
High availability guidelines are provided [here](https://github.com/mariadb-operator/mariadb-operator/blob/main/docs/HA.md).  

The Recommend HA setup for production is:
*Galera with at least 3 nodes. Always an odd number of nodes.
*Load balance requests using Kubernetes Services 
*Define pod disruption budgets.
*Object store for backup (Google Cloud Storage,Amazon S3)
The `mariadb-operator` provides cloud native support for provisioning and operating multi-master MariaDB clusters using Galera. This setup enables the ability to perform both read and write operations on all nodes, enhancing availability and allowing scalability across multiple nodes. All nodes support reads and writes. We have a designated primary where the writes are performed.
### A Word on High Availability
There are many ways to deploy an HA environment the Galera configuration is one of many ways. Another topology is a Single master HA via [SemiSync Replication](xhttps://github.com/mariadb-operator/mariadb-operator/blob/main/examples/manifests/mariadb_replication.yaml): The primary node allows both reads and writes, while secondary nodes only allow reads. We will not be covering that toplology in this runbook, but it is an approach you can take should you desire. 

## Deploying the MariaDB Instance for TileDB

The first step we need to do to deploy our highly available cluster is to apply our **tiledb-secret.yml** and **tiledb-root-secret.yml* secrets to the cluster by running 

`kubectl apply -f tiledb-secret.yml` and `kubectl apply -f tiledb-root-secret.yml`


You can then run `kubectl get secrets -n tiledb-cloud` to confirm the secrets have been created.

![TileDB Cloud Secrets](/images/get_secret.png "Get Secrets")



The next step will be to apply the **mariadb.yml** file by running `kubectl apply -f mariadb.yml`
You can then run `kubectl get mariadb -n tiledb-cloud` to view the mariadb instance being created.
![Deployed MariaDB Instance](/images/get_mariadb_resource.png "Get MariaDB Instances ")

It will take a few minutes for the cluster to be ready. To check the status of the pods run `kubectl get pods -n tiledb-cloud` you will then be able to see the pods name (in our case) `tile-mariadb-0` `tile-mariadb-1` `tile-mariadb-2` these are StatefulSets. Once they are ready, you can move on to apply the other manifests. 

![Deployed MariaDB Pods](/images/get_mariadb_pods.png "Get MariaDB Pods")

### MariaDB Custom Resource Troubleshooting Tips
Before moving onto the next step, we wanted to provide you with some additional information. Here are some tips and tricks.

* If MariaDB pods are not showing 2/2 then a container is not ready.  `Run kubectl logs <not ready pod> -n tiledb-cloud` to view logs and troubleshoot issues. A few common issue is not having applied the root password secret or not having the appropriate permissions to create resources. 
* If the pods are not present, you can run `kubectl describe mariadb <name of the applied mariadb resource> -n tiledb-cloud` and read any events the resource may be sharing.
* If you run into problems with deploying your initial MariDB resource and need to delete it and recreate it for whatever reason (most common being trying to edit an immutable field), you will need to delete the `PersistentVolumeClaims` to ensure there are no leftover configurations. First, delete the mariadb instance by running `kubectl apply -f mariadb.yml`. Then, delete the `PersitentVolumeClaims`. Do this by running `kubectl get pvc -n tiledb-cloud` to view the `PersistentVolumeClaims`. Then, delete the claims associated with the previously deployed MariaDB instance by running `kubectl delete  pvc <claim1 name> <claim2 name> <claim3 name> -n tiledb-cloud`

![Deployed MariaDB PVCs](/images/get_pvc.png "Get MariaDB PVCs")

You can then recreate your MariaDB instance just like you did in previous steps. You will see a new `PersistentVolumeClaims`.

**WARNING: Do NOT do the above PVC deletion step if the database is live and you have not created a `backup` resource (and associated job) you can confirm has run. You are deleting persistent volumes and can cause issues with users logging in and accessing resources. This step is most commonly executed on fresh MariaDB installs where no user data has been stored yet and you need to reattempt a deployment. DO NOT delete the backup PVC (see [Restoring to Your TileDB Cluster]())  either unless you have snapshotted it or can confirm you no longer need the backup data.**


## Deploying the MariaDB Configuration Manifests
### Deploying the MariaDB Database Custom Resource Definitions
Now that our MariaDB instance is up and running, we need to deploy the other resources. 
First, deploy the database by running  `kubectl create -f database.yml` then confirming the database was ready with `kubectl get database -n tiledb-cloud`.

![Deployed MariaDB Database](/images/create_and_get_db.png "Apply and Get Database")


### Deploying the MariaDB User and Grant Custom Resource Definitions

To deploy the required `user` and `grant` resources, run `kubectl create -f grant.yml and kubectl create -f user.yml`

![Deployed User and Grant](/images/create_user_and_grant.png "Apply and Get Grants and Users")


You can view the status of these resources using the command pattern of `kubectl get <resource name>` and `kubectl describe <resource name>` just like we did for the `mariadb` resource and other resources we’ve applied to the cluster. 

## Editing the TileDB values.yml to Reflect our Deployed MariaDB Instance
### Acquiring the TileDB Enterprise Values File
Once the resources are deployed, you are ready to make some adjustments to the TileDB Helm chart’s values file. Worth noting, this file is NOT in the MariaDB operator repository. You will need to follow the instructions [here](https://docs.tiledb.com/enterprise/installation) to get your values.yml file and access to the Helm chart.

### Configuring the Main Database

Roughly near line 157 in your values file, you will see a `Databases:` entry. That entry should look similar to below:

![Edit Databases values entry](/images/database_values.png "Edit Databases Values Entry")


Just like the above image, enter the password that you set in your **tiledb-secret.yml** file ensuring it is decoded using `base64 -d` . Then, edit the `Username` key value to be the name of the `user` you created and the `Schema` key value to be the name of the `database` you created. Lastly, update the `Host` entry with the [Kubernetes service]() that the MariaDB instance is exposed under. Do find this value, run `kubectl get svc -n tiledb-cloud` and use the service named after your mariadb resource. See below. The FQDN will look similar to what is in the above image.

![Get Deployed Services](/images/get_services.png "Get Services")


### Set MariaDB Database Connection DSN
We now need to update the DSN. At roughly line 489 there is a [DSN](https://support.microsoft.com/en-us/topic/what-is-a-dsn-data-source-name-ae9a0c76-22fc-8a30-606e-2436fe26e89f) value like below.

![Edit DSN](/images/DSN.png "Edit DSN")

The format is `mysql://<username>:<password>@tcp(<mariadb kubernetes service>:3306)/<schema>?parseTime=true.` 
Save the values file with your edits. You will still need to update the values file for the rest of TileDB install, but if applied and configured correctly, the MariaDB entries should be handled. Once the install is complete continue to the next section. 

## General Troubleshooting Tips
This section is not going to be a comprehensive troubleshooting guide for all things MariaDB, but will provide some techniques for troubleshooting common errors we've run into. 
### Initial Steps
The most beneficial first step is to run `kubectl get pods -n tiledb-cloud` and evaluate which pods are not in a  `Running` state.  Any pods reporting errors (see below for examples) are worth investigating. When it comes to potential MariaDB issues being reported by TileDB Enterprise, the `hydra` and `cloud-rest` pods are potential culprits. Therefore, running  `kubectl get logs <pod name> -n tiledb-cloud` and `kubectl describe pod <pod name> -n tiledb-cloud` command is a good first step. It is worth noting that MariaDB configuration errors are commonly reported by [init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) within the `hydra` and `rest` pods. When running the commands, you will need to specify a container with the `-c` flag. The init container for the `hydra` pod is named `hydra-automigrate` and the `init` container for the REST server is named `tiledb-cloud-rest-migrations`. These are great containers to gather logs from during the initial install process if they are failing. See example below.

![Get Logs](/images/gather_logs.png "Get Logs")

### Permission Errors
The most common type of errors we see when configuring MariaDB for TileDB Enterprise are permission errors.  The MariaDB Operator `grant` CustomResourceDefinition handles the permissions, but you may run into issues between a `user` being granted the appropriate permissions and assigned to the correct `database`.  

The first step we recommend is to ensure that the `grant`,`user`,`mariadb`,and `database` Custom Resources all properly reference each other.  

The next step is to confirm the `user` permissions and password work properly when accessing MariaDB directly.  

**NOTE: the provided commands are using our default configurations assuming you did not change the user or database. If you edited our manifests (or deployed your own) you will need to adjust them accordingly.**

You can exec into a MariaDB pod by running:

`kubectl exec -it tile-mariadb-0 -n tiledb-cloud -- bash`

Then you can test your root and user passwords by logging in and entering your passwords when prompted.

`mysql -u root -p`
`mysql -u tiledb -p`

Once logged in, you can run commands to validate permissions such as:

`SHOW GRANTS FOR 'tiledb'@'%';`

You could then grant privileges to your user directly using:


`GRANT ALL PRIVILEGES ON `tile-rest`.* TO 'tiledb'@'%' WITH GRANT OPTION; FLUSH PRIVILEGES;`


Then confirming your TileDB install finishes without issue.

An additional step is to ensure the host for the user is any host instead of a specific host by running

`SELECT Host, User FROM mysql.user WHERE User = 'tiledb' AND Host != '%';`


The query should return nothing thus confirming you have the proper host configuration.


## Declaratively Backing Up and Restoring MariaDB 
### Introduction and Overview 
The official MariaDB documentation on backing up and restoring MariaDB instances can be found [here](https://github.com/mariadb-operator/mariadb-operator/blob/main/docs/BACKUP.md). We provide example manifests in this  runbook repo under `manifests/data_protection`.

### Storage Overview
At the time of writing, the supported backup targets are:
* S3 compatible storage: Store backups in a S3 compatible storage, such as AWS S3 or MinIO.
* PVCs: Use the available StorageClasses in your Kubernetes cluster to provision a PVC dedicated to store the backup files.
* Kubernetes volumes: Use any of the volume types supported natively by Kubernetes

Since TileDB already uses object stores, we will use an object store for our backups. The MariaDB Operator community also recommends that we store the backups externally in S3 compatible storage. **The exception to this is Azure due to limited S3 support**. For Azure, we will be using a PVC. 

### Manifests Overview 
The `data_protection` directory contains three manifests: **Scheduled_backup.yml**, **restore.yml**, and **credentials.yml**. Before we continue, let's explore these manifests. 

**Scheduled_backup.yml**: is a MariaDB `backup` custom resource definition. The resource references `bucket`, `endpoint`,`accessKeyIdSecretKeyRef`,and `secretAccessKeySecretKeyRef` in order to define where to place backups for the MariaDB instance. The resource can also schedule backups and define retention policies. By default, all the logical databases are backed up when a `backup` is created.

**Restore.yml**: Is a `restore` instance resource that uses `backupRef` to restore a MariaDB from a `backup` and ensure `users`, `grants`, and `databases` are still mapped to the resource.  The operator will look for the closest backup available and utilize it to restore your MariaDB instance.
By default, spec.targetRecoveryTime will be set to the current time, which means that the latest available backup will be used. You can also specify a specific backup.

**Backup-creds.yml**: In order to access an object storage endpoint, the user must provide credentials as a `secret`. We recommend that you do not check the credentials file into git unless using a tool such as [sealed secrets](https://github.com/bitnami-labs/sealed-secrets) or [sops](https://github.com/getsops/sops). Those tools are outside the scope of this document. In this runbook, we will be applying the secrets directly to the cluster.

### Backup Processes
The first step to backing up and restoring a MariaDB resource is to ensure you have a backup target and the appropriate credentials. The credentials and backup targets depends on which cloud provider you are deploying your TileDB Enterprise solution to. 

#### Google Cloud Platform
For GCP, you will need:
A GCP bucket [guide here](https://cloud.google.com/storage/docs/creating-buckets)
A Service Account with minimally the [Cloud Storage](https://cloud.google.com/iam/docs/understanding-roles#storage.objectUser) User role assigned. To create a service account, follow [these instructions](https://cloud.google.com/iam/docs/service-accounts-create).
HMAC(Hash-based Message Authentication Code) keys attached to the previously created service account. Instructions can be found [here](https://cloud.google.com/storage/docs/authentication/managing-hmackeys#create).

##### Backing up to Cloud Storage
1. Create a service account, and a GCP bucket per the above instructions.
2. Create and store the HMAC credentials in the **backup-creds.yml**  per the above instructions 
3. Apply the **backup-creds.yml** file by running `kubectl create -f backup-creds.yml`
4. Edit the **scheduled_backup.yml** file by setting the `endpoint` value to `https://storage.googleapis.com` and the bucket to your bucket  (i.e. chase_poc).
5. Then add your backup folder (i.e. backups) to the `prefix` entry.  
6. Save and apply the file by running `kubectl create -f scheduled_backup.yml`
7. Check on the status of the `backup` by running `kubectl get backups -n tiledb-cloud`

![Get Backups](/images/get-backups.png "Get Backups")


#### Azure
For Azure you will need:
* A default [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)
##### Backing up to an Azure Disk
Azure blob does not have an easy out of the box S3 API, therefore instead of using S3, we will use a `PersistentVolume` backed by [Azure Disks](https://learn.microsoft.com/en-us/azure/aks/azure-csi-disk-storage-provision). Azure disks are bound to a single zone, therefore, for maximum protection of backups, we **recommend you snapshot** your Azure disks. That way if a zone is down, you can bootstrap the MariaDB instance from a snapshotted volume and restore functionality. If this is not a pattern you’d prefer to support, you can also opt into using [MinIO](https://min.io/) backed by Azure Blob. The upstream documentation has a tutorial on how to install MinIO [here](https://github.com/mariadb-operator/mariadb-operator/blob/main/docs/BACKUP.md#minio-reference-installation) and backup to MinIO [here](https://github.com/mariadb-operator/mariadb-operator/blob/main/docs/BACKUP.md#minio-reference-installation). For this runbook, we will be using a `PersistentVolumeClaim`.

Worth noting, these steps will allow you to backup to **any Kubernetes cluster** with a `StorageClass`. 
1. Comment out the storage section in the **scheduled_backup.yml**. 
2. Apply the scheduled_backup.yml file to the cluster. You **do not** need to apply the **backup-creds.yml**
3. Check the pods to ensure that the `job` has completed. 
4. Check the `PVC`s (see below) and you should see a backup `PVC` that has been provisioned. You are now ready to move onto a restore should you need it. You can define a specific backup or the resource will default to the most recent backup.
5.  To explore the backup volume, you can use and edit the **pvc-viewer.yml** manifests provided in the data_protection directory. All you need to do is apply the manifest and exec into it. See below. You can then follow the standard restore steps [here] (MD link)

***WARNING: The backups are now on a single Azure Disk. We recommend snapshotting the disk. You can do this declaratively in Kubernetes. A guide can be found [here](https://learn.microsoft.com/en-us/azure/storage/container-storage/volume-snapshot-restore).***

![Get Backups PVCs](/images/get_backup_pvc.png "Get Backup PVCs")


#### Amazon Web Services  
AWS (Amazon Web Services) is the simplest cloud to perform backups on due to its native support of S3. However, ***We highly recommend you use RDS***. More details on RDS can be found [here](https://aws.amazon.com/rds/mariadb/) 
For AWS you will need: 
* An AWS Access Key and Secret 
* A S3 bucket with a backup container (you can name it whatever you want. We will name ours maria-bak)

**Common EBS ISSUE!:** AWS EKS clusters **DO NOT** install the EBS driver by default. You may need to install the EBS Driver for your MariaDB instance using the guide [here](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) 

##### Backing up to Amazon S3
1. Create and copy your access keys and credentials. (see below)
2. Place the Access key value as the `object_id key` value in your and the Secret access key value as the `object_key key` value in the backup_creds.yml file. Do not forget to base64 encode them. 
3. Apply the file to your Kubernetes TileDB cluster

![Get Access Key](/images/retrieve_access_key.png "Get Access Key")

4. Edit the `endpoint` and `prefix` values as well as the `region` values.
5. Apply the **scheduled_backup.yml** file.

### Restoring Your MariaDB Instance  to Your TileDB Cluster 
Once you confirm that the bucket has a backup file, you can then deploy a `restore` resource that references the `backups`. To do this, apply the **restore.yml** manifest from the **data_protection** directory.  To use a specific backup, use the backup file name’s timestamp (I.E. /backup.2024-08-01T04:00:01Z.sql becomes 2024-08-01T04:00:01Z) and add it as the value to `targetRecoveryTime` in the **restore.yml** file. Once completed you should see an output similar to below. 

![Get Restores](/images/get_restore.png "Get Restores")
 

#### Confirming the Restore Worked 
To validate the restore worked properly run the following in order:

`kubectl exec -it tile-mariadb-0 -n tiledb-cloud -- bash`

`mysql -u tiledb -p`

`SHOW GRANTS FOR 'tiledb'@'%';`

`SHOW DATABASES;`

`USE  tile-rest;`

`SHOW tables;`

You should see several tables. To complete the restore you may need to delete the rest, hydra, and UI pods ensuring they connect to the restore database. Once pods are healthy, users may reconnect to the TileDB self-hosted instance. You may need to restart the pods. If you choose to redeploy TileDB using the previously provisioned MariaDB instance, you will not need to create fresh databases. The migration scripts should automatically detect the previous configuration and move onto the rest of the deployment. If you see initial errors, you may need to wait 5-10 minutes for the applications to resolve the issue. 


#### Backup Resource Troubleshooting Steps
 If you run into errors with the `backup` resource, you can describe the resource or you can get logs from the MariaDB Operator for more robust error reports. You can pipe the log into less for easier search like so: 
`kubectl logs tiledb-mariadb-mariadb-operator-549cb85bbb-bb8vh -n tiledb-cloud |less`
Once the `backup` resource has the status `scheduled` you can check the `jobs` resource by `running kubectl get jobs -n tiledb-cloud` and inspecting the scheduled jobs like below

![Get Sceduled Job](/images/get_jobs.png "Get Scheduled Jobs")


You can also inspect the pods associated with the jobs and get logs by executing a command similar to `kubectl logs scheduled-backup-28707834-zvbw9 -n tiledb-cloud`

To confirm the backup was successful, you should be able to see a backup file within the designated bucket (if using cloud storage see below) or `PersistentVolume` 

[View Backups on Bucket](/images/view_bucket.png "View Backups on Bucket")














 










