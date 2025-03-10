---
section: self-hosted/latest
subsection: operational-guides
title: How to backup and restore Gitpod
---

<script context="module">
  export const prerender = true;
</script>

# How to Backup and Restore Gitpod

> ⚠️ **Self-hosted as a product is no longer supported**
>
> The last official update of this product is the [November 2022 self-hosted release](https://www.gitpod.io/changelog/november-self-hosted-release). We no longer sell commercial self-hosted licenses. If you want to self-host Gitpod, you can still request our [free community license](https://www.gitpod.io/community-license). However, we no longer offer support or updates for it. If you are interested in an isolated, private installation of Gitpod, take a look at [Gitpod Dedicated](/dedicated).
> Read [our blog on Gitpod Dedicated](/blog/introducing-gitpod-dedicated) to learn why we made the decision to discontinue self-hosted.

For [business continuity](https://en.wikipedia.org/wiki/Business_continuity_planning) purposes, it is important to think about how you might restore your ability to use Gitpod, and thus develop software in the event of a catastrophic failure of Gitpod or the underlying infrastructure it runs on. This guide assumes that you will use the backup and restore strategy for disaster recovery and will guide you towards what needs to be backed up and how to restore using said backup. Please see our background reading on [disaster recovery](./disaster-recovery) for more information.

> **Important:** When using Gitpod in a production setting, we recommend you base your installation on the [single cluster reference architecture](./reference-architecture/single-cluster-ref-arch). Using in-cluster dependencies is not recommended because there is no means to produce backups, and the database/storages systems are within failure domain of the cluster. If possible consider using Gitpod [SaaS](https://www.gitpod.io/pricing).

> **Note:** We recommend to regularly trial run a recovery using this method to ensure that it works in practice and to allow yourself the chance to spot any unforeseen issues.

## What to back up

It is critical to consider what needs to be backed up and ensure you take the necessary steps to secure each of the listed elements. What needs to be backed up is closely aligned with [Gitpod’s architecture](./reference-architecture/single-cluster-ref-arch#overview) and how it runs.

### Database

The database is a central component in Gitpod where all metadata about users and workspaces, as well as settings of the Gitpod instance (such as auth providers) are stored. This makes the database a critical component. In case of a database outage, you are not able to log in, use the Gitpod dashboard, or start workspaces. We recommend using a cloud provider native relational database service that supports MySQL - see [required components](./requirements). This means that you can rely on the best practices of each service for disaster recovery. For example:

- **AWS Relational Database Service**: You can rely on RDS’s [backup and restore mechanisms](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.BackupRestore.html) and set it up to run in [multiple availability zones](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/create-multi-az-db-cluster.html) to allow for higher availability. You can also consider other [disaster best practices](https://medium.com/tensult/amazon-rds-disaster-recovery-8a40dd8350ea).
- **Google Cloud SQL:** You can rely on Cloud SQLs [backup and restore features](https://cloud.google.com/sql/docs/mysql/backup-recovery/backups) and run the instance in a [high availability configuration](https://cloud.google.com/sql/docs/mysql/high-availability) to decrease the chance of failure. You can also refer to the general [Google Cloud disaster recovery best practices](https://cloud.google.com/architecture/dr-scenarios-planning-guide).

### Object Storage

Gitpod uses **object storage** to store blob data. This includes workspace backups created when a workspace stops and are used to restore the state upon restart. As such, to secure the work of your users, it is critical to think about backing up this data and/or relying on the best practices for disaster recovery of the object storage service being used. For example:

- **AWS S3:** You can consider using [cross-region replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html) to increase reliability further - although S3 already stores your data across multiple geographically distant _Availability Zones_ by default.
- **Google Cloud Storage:** Consider using the [Multi-Regional Storage option](https://cloud.google.com/storage/docs/storage-classes) for additional availability.

### OCI Image Registry

Gitpod uses an image registry to cache images and store images it builds on behalf of users. _Note: For non-airgapped environments, this is **not** the registry that contains the images of Gitpod’s services._ As such, losing this data means that workspace starts may take longer because images need to be re-built. Consider implementing best practices for securing the registry you are using.

### Application Config

> **Important:** KOTS Snapshots will **NOT** save any data from your Gitpod database, registry or object storage. It will also not backup any data outside of your `gitpod` namespace. It will backup:
>
> - the KOTS dashboard
> - the KOTS configuration
> - the version of Gitpod installed
> - the TLS certificate generated by cert-manager (if enabled)

Although you could simply re-install Gitpod using the regular [installation path](./installing-gitpod), this can take some time and you would need to re-configure it to the state you had last had it in. To minimize your recovery time, you can persist the application configuration (ideally regularly).

#### Configuring Velero

[Velero](https://velero.io/) is an open source tool to safely backup and restore, perform disaster recovery, and migrate Kubernetes cluster resources and persistent volumes. It is used by KOTS to connect to your backup location. It supports many [data sources](https://docs.replicated.com/enterprise/snapshots-storage-destinations), including AWS, Azure and GCP storage solutions.

Please follow the installation instructions as per the [Velero documentation](https://velero.io/docs/latest/basic-install). KOTS requires [Restic integration](https://velero.io/docs/v1.9/restic) to function correctly, which can be added by appending the `--use-restic` flag to the `velero install` command.

#### Triggering Your First Backup

> For full documentation on the KOTS backup solution, please see their [documentation](https://docs.replicated.com/enterprise/snapshots-understanding).

To create a new backup via the KOTS CLI or in the Snapshots section of your KOTS dashboard, you can do this by running the following command:

```bash
kubectl kots backup --namespace gitpod
```

When that has finished, you will be able to list your backups:

```bash
kubectl kots backup ls
```

And it will display a list that looks similar to this:

```bash
NAME              STATUS       ERRORS    WARNINGS    STARTED                          COMPLETED                        EXPIRES
instance-ab1cd    Completed    0         0           2022-08-11 13:36:38 +0100 BST    2022-08-11 13:36:54 +0100 BST    29d
```

### Cluster Configuration

To reduce the time it takes you to re-create a cluster, you can move to an infrastructure as code flow, e.g. by codifying the infrastructure you need using Terraform.

## How to restore

The following explains how you might restore Gitpod after its underlying cluster fails.

1. Recreate your infrastructure. Ideally, you do this using something like a Terraform script.
2. Configure Velero using the instructions [above](#configuring-velero) - it is recommended that you install the same version that you used previously.
3. List your available backups using:

```bash
kubectl kots backup ls
```

4. Restore the backup using:

```bash
kubectl kots restore --from-backup instance-ab1cd
```

5. Load the KOTS dashboard:

```bash
kubectl kots admin-console --namespace gitpod
```

6. Hit the "Redeploy" button.
7. This should result in your Gitpod instance having the same state as before, thus allowing your users to pick up where they left off.

## Security considerations

### Application Config

Velero should be configured so that is deployed to a different namespace to Gitpod. The Velero deployment will contain secrets which will allow access to your backup source of choice. Your Kubernetes cluster should be configured to limit access to these resources with use of a role-based access policy (RBAC).

You should always consult with the Velero documentation to ensure that you are following their best practice guidelines to ensure the integrity of your backup artifacts.
