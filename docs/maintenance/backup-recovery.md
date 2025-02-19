---
status: new
---

# Backup And Recovery

## Overview

There are generally three parts of data to backup

- SeaTable tables data
- Databases
- Configuration files with private keys

If you setup your SeaTable server according to our manual, there should exist a directory layout like this:

```
/opt/seatable-server/seatable/
├── ccnet
├── conf
├── db-data
├── logs
├── pids
├── scripts
├── seafile-data
├── seahub-data
└── storage-data

```

All your data is stored under the `/opt/seatable-server/seatable/` directory.

#### Important sub-directories that contain user data:
- seafile-data: contains uploaded files for file and image columns
- seahub-data: contains data used by web front-end, such as avatars
- db-data: contains archived rows in bases
- storage-data: contains backups for the bases in dtable-db (added in Enterprise Edition 3.0.0); Since version 3.0.0, tables and snapshots are also stored in this directory.

#### Important MariaDB databases that contain user/metadata:
- ccnet_db: contains user and group information
- seafile_db: contains library metadata
- dtable_db: contains tables used by the web front end

??? info "Database structure"

    SeaTable stores the following data types in the MySQL database engine in the `mariadb` Docker container:

    * user actions and inputs in bases (e.g. new/modified/deleted rows, new/modified/deleted columns, new/modified, deleted views)
    * meta-information for bases (e.g. API-token, common datasets, links, row comments, snapshots, third-party accounts, webhooks)
    * statistical and log information (e.g. automation rules execution, row count)
    * user and group information (e.g. 2FA status, logins, user quota)
    * versioning information for the assets (e.g. files and images) saved in bases

## Backup

### Steps

1. Backup the MySQL databases
2. Backup the /opt/seatable-server/ directory
3. Backup the /opt/seatable-compose/ directory (with your seatable license and .env secrets)

Backup Order: Database First or Data Directory First

### 1. Backup the MySQL databases

```bash
# It's recommended to backup the database to a separate files each time. Don't overwrite older database backups for at least a week.
# replace <your_mysql_password> with your actual MySQL password (might be still present in /opt/seatable-compose/.env)
# beware that this method will expose your mysql password in the process list and shell history of the docker host

mkdir -p /opt/seatable-backup/databases && cd /opt/seatable-backup/databases
docker exec -it "mariadb" "mysqldump" -u"root" -p'<your_mysql_password>' --opt "ccnet_db" > "./ccnet_db.sql"
docker exec -it "mariadb" "mysqldump" -u"root" -p'<your_mysql_password>' --opt "seafile_db" > "./seafile_db.sql"
docker exec -it "mariadb" "mysqldump" -u"root" -p'<your_mysql_password>' --opt "dtable_db" > "./dtable_db.sql"
```

!!! warning

    The above commands do not work via cronjob. To create dumps of the database via cronjob, the parameters `-it` must be omitted.

### Backing up SeaTable data

You can use rsync to do incremental backup for data directories (assuming /opt/seatable-backup/ already exists)

```bash
mkdir -p /opt/seatable-backup/
rsync -az --exclude 'ccnet' --exclude 'logs' --exclude 'db-data' /opt/seatable-server /opt/seatable-backup
rsync -az /opt/seatable-compose /opt/seatable-backup
```

You may notice that `db-data` directory is not backed up. The data in this directory is backed up in a different way. Please refer to the next sub-section.

#### Setup automatic backup for dtable-db

_available since Enterprise Edition 3.0.0_

Data managed by dtable-db component is archived rows from bases. They should be backed up as well. Data for dtable-db sits in the `/opt/seatable-server/seatable/db-data` directory.

Unlike other components, dtable-db provides built-in automatic backup mechanism. It will take a snapshot for each base and upload to dtable-storage-server. dtable-db only make new backup for a base if it detects changes to it. This makes the backup more efficient. dtable-storage-server also compresses the backups to make it more storage-efficient.

To setup automatic backup for dtable-db:

1. Setup and run dtable-storage-server. It should be started by default. Find more details in [dtable-storage-server documentation](../configuration/dtable-storage-server-conf.md).
2. Set `[backup]` configuration options in dtable-db.conf as in [dtable-db documentation](../configuration/dtable-db-conf.md)

If you configure dtable-storage-server with local file system as backend, dtable-storage-server saves its data to the path specified in dtable-storage-server.conf. By default it's set to `/opt/seatable-server/seatable/storage-data`. If you set up your backup as in the last section, you should have already backed up this directory as well. Since storage-data directory has already contained the backups for dtable-db, data in db-data directory doesn't need to backup.

If you configure dtable-storage-server with object storage as backend, there will be no data saved to `/opt/seatable-server/seatable/storage-data`. So you don't have to backup storage-data directory either.

You can also manually execute the command to backup dtable-db data immediately

```
docker exec -it seatable-server /opt/seatable/scripts/seatable.sh backup-all
```

## Restore

### Restore the databases

```bash
# replace <your_mysql_password> with your actual MySQL password (might be still present in /opt/seatable-compose/.env)
# beware that this method will expose your mysql password in the process list and shell history of the docker host

docker exec -i "mariadb" "/usr/bin/mysql" -u"root" -p'<your_mysql_password>' ccnet_db < /opt/seatable-backup/databases/ccnet_db.sql
docker exec -i "mariadb" "/usr/bin/mysql" -u"root" -p'<your_mysql_password>' seafile_db < /opt/seatable-backup/databases/seafile_db.sql
docker exec -i "mariadb" "/usr/bin/mysql" -u"root" -p'<your_mysql_password>' dtable_db < /opt/seatable-backup/databases/dtable_db.sql

```

### Restore the SeaTable data

```
rsync -az /opt/seatable-backup/seatable-server /opt
rsync -az /opt/seatable-backup/seatable-compose /opt

```

### Restore the dtable-db data

```
docker exec -it seatable-server /opt/seatable/scripts/seatable.sh restore-all
```

## Database Backup (optional - without operation log)

Before running `clean_db_records`, you can make a backup by a shell script. The following tables with many records will be excluded:

- operation_log
- delete_operation_log
- session_log
- activities

Example of the `database_backup.sh` backup shell script：

```
#!/bin/bash
set -e
db_host='<IP address of database>'
db_user='root'
db_name='dtable_db'
backup_dir='/opt/seatable/db-backups'

echo 'Start backing up the database'

mysqldump -h$db_host -u$db_user -p --opt $db_name \
  --ignore-table=$db_name.activities \
  --ignore-table=$db_name.delete_operation_log \
  --ignore-table=$db_name.operation_log \
  --ignore-table=$db_name.session_log \
  > $backup_dir/seatable.sql.`date +"%Y-%m-%d"`

echo 'Database backup succeeded'
```

Run the shell script:

```
$ ./database_backup.sh
Start backing up the database
Enter password: xxxxx
Database backup succeeded
```
