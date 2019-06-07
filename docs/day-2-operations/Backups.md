# 3scale Backup & Restore

## Overview

This section intends to provide an Operator of a 3scale installation the information they need to:

* Setup the backup procedures for persistent data
* Perform a restore from backup of the persistent data

Such that, in the case of a failure, they can restore 3scale to correctly running operational state and continue operating.

### Persistent Volumes

In a 3scale deployment on OpenShift all persistent data is stored either in a storage service in the cluster (not currently used), a Persistent Volume (PV) provided to the cluster by the underlying infrastructure or a storage service external to the cluster (be that in the same Data Center or elsewhere).

### Considerations

The backup and restore procedures for persistent data are required to vary depending on the storage used, to ensure the backups and restores preserve data conistsency (e.g. that a partial write, or a partial transaction is not captured). i.e. It is not sufficient to backup the underlying Persistent Volumes for a database, but instead the databases backup mechanisms should be used.

Also, some parts of the data are synchronized between different components. One copy is considered the "source of truth" for the data set, and the other a copy that is not modified locally, just synchronized from the "source of truth". In these cases, upon restore, the "source of truth" should be restored and then the copies in other components synchronized from it.

## Data Sets

This section goes into more detail on the different data sets in the different persistent stores, their purposes, the storage type used and if it is the "source of truth" or not.

The full state of a 3scale deployment is stored across these services and their persistent volumes:

* `system-mysql`:  Mysql database (`mysql-storage`)
* `system-storage`: Volume for Files
* `zync-database`: Postgres database for `zync` component. This uses "HostPath" as storage. If the pod is moved into another node the data is lost. That is not a problem because the data are sync jobs and does not need to be 100% persistent.
* `backend-redis`: Redis database (`backend-redis-storage`)
* `system-redis`: Redis database (`system-redis-storage`)

### `system-mysql`

This is a relational database for storing the information about users, accounts, APIs, plans etc in the 3scale Admin Console. 

A subset of this information related to Services is synchronized to the `Backend` component and stored in `backend-redis`. `system-mysql` is the source of truth for this information.

### `system-storage`

This is for storing files to be read and written by the `System` component. They fall into two categories:

* Configuration files read by the System component at run-time
* Static files (HTML, CSS, JS, etc) uploaded to System by its CMS feature, for the purpose of creating a Developer Portal.

Note that `System` can be scaled horizontally with multiple pods uploading and reading said static files, hence the need for a `RWX` PersistentVolume.

### `zync-database`

This is a relational database for storing information related to the synchronization of identities between 3scale and an Identity provider.
This information is not duplicated in other components and this is the sole source of truth.

### `backend-redis`

This contains multiple data sets used by the `Backend` component:

* Usages: This is API usage information aggregated by `Backend`. It is used by `Backend` for rate-limiting decisions and by `System` to display analytics information in the UI or via API.
* Config: This is configuration information about Services, Rate-limits, etc that is synchronized from `System` via an internal API. This is NOT the source of truth of this info, `System` and `system-mysql` is.
* AuthKeys: Storage of OAuth keys created directly in `Backend`. This is the source of truth for this information.
* Queues: Queues of Background Jobs to be executed by worker processses. These are ephemeral and are deleted once processed.

### `system-redis`

This contains Queues for jobs to be processed in background. These are ephemeral and are deleted once processed.

## Backup Procedures

### `system-mysql`

1. Execute MySQL Backup Command

   ```bash
   oc rsh $(oc get pods -l 'deploymentConfig=system-mysql' -o json | jq -r '.items[0].metadata.name') bash -c 'export MYSQL_PWD=${MYSQL_ROOT_PASSWORD}; mysqldump --single-transaction -hsystem-mysql -uroot system' | gzip > system-mysql-backup.gz
   ```

### `system-storage`

1. Archive the system-storage files to another storage.

    ```bash
    oc rsync $(oc get pods -l 'deploymentConfig=system-app' -o json | jq '.items[0].metadata.name' -r):/opt/system/public/system ./local/dir
    ```

### `zync-database`

1. Execute Postgres Backup Command

   ```bash
    oc rsh $(oc get pods -l 'deploymentConfig=zync-database' -o json | jq '.items[0].metadata.name' -r) bash -c 'pg_dumpall -c --if-exists' | gzip > zync-database-backup.gz
    ```

### `backend-redis`

1. Backup the dump.rb file from redis

   ```bash
   oc cp $(oc get pods -l 'deploymentConfig=backend-redis' -o json | jq '.items[0].metadata.name' -r):/var/lib/redis/data/dump.rdb ./backend-redis-dump.rdb
   ```

### `system-redis`

1. Backup the dump.rb file from redis

   ```bash
   oc cp $(oc get pods -l 'deploymentConfig=system-redis' -o json | jq '.items[0].metadata.name' -r):/var/lib/redis/data/dump.rdb ./system-redis-dump.rdb
   ```

## Restore Procedures

### `system-mysql`

1. Copy the MySQL dump to the system-mysql pod

   ```bash
   oc cp ./system-mysql-backup.gz $(oc get pods -l 'deploymentConfig=system-mysql' -o json | jq '.items[0].metadata.name' -r):/var/lib/mysql
   ```

1. Decompress the Backup File

    ```bash
    oc rsh $(oc get pods -l 'deploymentConfig=system-mysql' -o json | jq -r '.items[0].metadata.name') bash -c 'gzip -d ${HOME}/system-mysql-backup.gz'
    ```

1. Restore the MySQL DB Backup file

   ```bash
   oc rsh $(oc get pods -l 'deploymentConfig=system-mysql' -o json | jq -r '.items[0].metadata.name') bash -c 'export MYSQL_PWD=${MYSQL_ROOT_PASSWORD}; mysql -R -hsystem-mysql -uroot system < ${HOME}/system-mysql-backup'
   ```

### `system-storage`

1. Restore the Backup file to system-storage

   ```bash
   oc rsync ./local/dir $(oc get pods -l 'deploymentConfig=system-app' -o json | jq '.items[0].metadata.name' -r):/opt/system/public/system
   ```

### `zync-database`

1. Copy the Zync Database dump to the zync-database pod

   ```bash
   oc cp ./zync-database-backup.gz $(oc get pods -l 'deploymentConfig=zync-database' -o json | jq '.items[0].metadata.name' -r):/var/lib/pgsql/
   ```

1. Decompress the Backup File

    ```bash
    oc rsh $(oc get pods -l 'deploymentConfig=zync-database' -o json | jq -r '.items[0].metadata.name') bash -c 'gzip -d ${HOME}/zync-database-backup.gz'
    ```

1. Restore the PostgreSQL DB Backup file

   ```bash
   oc rsh $(oc get pods -l 'deploymentConfig=zync-database' -o json | jq -r '.items[0].metadata.name') bash -c 'psql -f ${HOME}/zync-database-backup'
   ```

### `backend-redis`

* After restoring `backend-redis` a sync of the Config information from `System` should be forced, to ensure the information in `Backend` is consistent with that in `System` (the source of truth).

1. Edit the `redis-config` configmap

   ```bash
   oc edit configmap redis-config
   ```

1. Comment `SAVE` comands in the `redis-config` configmap

   ```bash
    #save 900 1
    #save 300 10
    #save 60 10000
   ```

1. Set `appendonly` to no in the `redis-config` configmap

   ```bash
   appendonly no
   ```

1. Re-deploy `backend-redis` to load the new configurations

   ```bash
   oc rollout latest dc/backend-redis
   ```

1. Rename the `dump.rb` file

   ```bash
   oc rsh $(oc get pods -l 'deploymentConfig=backend-redis' -o json | jq '.items[0].metadata.name' -r) bash -c 'mv ${HOME}/data/dump.rdb ${HOME}/data/dump.rdb-old'
   ```

1. Rename the `appendonly.aof` file

   ```bash
   oc rsh $(oc get pods -l 'deploymentConfig=backend-redis' -o json | jq '.items[0].metadata.name' -r) bash -c 'mv ${HOME}/data/appendonly.aof ${HOME}/data/appendonly.aof-old'
   ```

1. Move the Backup file to the POD

   ```bash
   oc cp ./backend-redis-dump.rdb $(oc get pods -l 'deploymentConfig=backend-redis' -o json | jq '.items[0].metadata.name' -r):/var/lib/redis/data/dump.rdb
   ```

1. Re-deploy `backend-redis` to load the backup

   ```bash
   oc rollout latest dc/backend-redis
   ```

1. Edit the `redis-config` configmap

   ```bash
   oc edit configmap redis-config
   ```

1. Uncomment `SAVE` comands in the `redis-config` configmap

   ```bash
    save 900 1
    save 300 10
    save 60 10000
   ```

1. Set `appendonly` to yes in the `redis-config` configmap

   ```bash
   appendonly yes
   ```

1. Re-deploy `backend-redis` to reload the default configurations

   ```bash
   oc rollout latest dc/backend-redis
   ```

### `system-redis`

1. Edit the `redis-config` configmap

   ```bash
   oc edit configmap redis-config
   ```

1. Comment `SAVE` comands in the `redis-config` configmap

   ```bash
    #save 900 1
    #save 300 10
    #save 60 10000
   ```

1. Set `appendonly` to no in the `redis-config` configmap

   ```bash
   appendonly no
   ```

1. Re-deploy `system-redis` to load the new configurations

   ```bash
   oc rollout latest dc/system-redis
   ```

1. Rename the `dump.rb` file

   ```bash
   oc rsh $(oc get pods -l 'deploymentConfig=system-redis' -o json | jq '.items[0].metadata.name' -r) bash -c 'mv ${HOME}/data/dump.rdb ${HOME}/data/dump.rdb-old'
   ```

1. Rename the `appendonly.aof` file

   ```bash
   oc rsh $(oc get pods -l 'deploymentConfig=system-redis' -o json | jq '.items[0].metadata.name' -r) bash -c 'mv ${HOME}/data/appendonly.aof ${HOME}/data/appendonly.aof-old'
   ```

1. Move the Backup file to the POD

   ```bash
   oc cp ./system-redis-dump.rdb $(oc get pods -l 'deploymentConfig=system-redis' -o json | jq '.items[0].metadata.name' -r):/var/lib/redis/data/dump.rdb
   ```

1. Re-deploy `system-redis` to load the backup

   ```bash
   oc rollout latest dc/system-redis
   ```

1. Edit the `redis-config` configmap

   ```bash
   oc edit configmap redis-config
   ```

1. Uncomment `SAVE` comands in the `redis-config` configmap

   ```bash
    save 900 1
    save 300 10
    save 60 10000
   ```

1. Set `appendonly` to yes in the `redis-config` configmap

   ```bash
   appendonly yes
   ```

1. Re-deploy `system-redis` to reload the default configurations

   ```bash
   oc rollout latest dc/system-redis
   ```

## Open Issues

* What about System services and sphinx (index)?
* How to handle backup/restore of job queues (of different types). They  can be lost or maybe done twice!
