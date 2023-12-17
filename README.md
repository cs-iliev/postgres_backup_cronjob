# Backup postgres to Azure Blob Storage

This document outlines our approach to implementing scheduled backups and on-demand restoration of data in a Kubernetes environment. We utilized Kubernetes CronJobs for automated, scheduled backups, and Kubernetes Jobs for on-demand restoration.

## Scheduled Backups with CronJob

### Objective

To ensure data consistency and reliability, we implemented a scheduled backup system using Kubernetes CronJobs. This setup automates the process of backing up our data at regular intervals.

### Implementation
##### CronJob Configuration
The CronJob is configured to run a backup script at scheduled times. Here's an example of the CronJob configuration:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup-cronjob
spec:
  schedule: "0 * * * *" # Runs every hour
  jobTemplate:
    spec:
      template:
        metadata:
          name: postgres-backup
        spec:
          containers:
          - name: upload-backup-container
            image: python:3.9-slim
            envFrom:
            - secretRef:
                name: postgres-backup-secrets
            - configMapRef:
                name: postgres-backup-config
            command: ["/bin/bash", "-c"]
            args:
              - |
                #!/bin/bash

                pip install azure-storage-blob
                python /script/upload_backup_to_blob.py
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
            - name: script
              mountPath: "/script"

          initContainers:
          - name: postgres-backup-container
            image: postgres:latest
            envFrom:
            - secretRef:
                name: postgres-backup-secrets
            - configMapRef:
                name: postgres-backup-config
            command: ["/script/backup_script.sh"]
            volumeMounts:
            - name: backup-volume
              mountPath: /backup-volume
            - name: script
              mountPath: "/script"

          restartPolicy: OnFailure
          volumes:
          - name: script
            configMap:
              name: postgres-backup-config
              defaultMode: 0500
          - name: backup-volume
            emptyDir: {}
```
This CronJob runs every hour and consists of two containers that execute consecutively. They share two volumes. The scripts are stored within the **script** volume while **backup-volume** is where the backup is stored. each container executes a script. The first container executes *backup_script.sh* which stores a backup in the form *wiki-db-\<time-stamp\>.sql* and stores it in */backup* directory inside **backup-volume**. The scripts we use are stored inside *configmap.yaml*

```bash
#!/bin/bash

POSTGRES_HOST="db-postgresql-service.wiki.svc.cluster.local"
TIMESTAMP="$(date +%Y%m%d_%H%M%S)"
BACKUP_FILE="/backup-volume/$POSTGRES_DB-$TIMESTAMP.sql"
pg_dump -U $POSTGRES_USER -d $POSTGRES_DB -h $POSTGRES_HOST > $BACKUP_FILE
```
Next we execute *upload_backup_to_blob.py* which utilizes Azure Python SDK to connect to the Azure Storage Blob and upload the backup.
```python
import sys
import os
import base64
import binascii
from datetime import datetime, timedelta
from azure.storage.blob import BlobServiceClient, generate_account_sas, ResourceTypes, AccountSasPermissions

# Retrieve environment variables
storage_account_name = os.getenv('AZURE_STORAGE_ACCOUNT_NAME')
storage_account_key = os.getenv('AZURE_STORAGE_ACCOUNT_KEY')
container_name = os.getenv('CONTAINER_NAME')
backup_dir = "/backup"

# Find the most recent backup file
most_recent_backup = max(
    (f for f in os.listdir(backup_dir) if f.startswith('wiki-db-') and f.endswith('.sql')), 
    key=lambda x: os.path.getmtime(os.path.join(backup_dir, x)),
    default=None
)

# Generate SAS token
sas_token = generate_account_sas(
    account_name=storage_account_name,
    account_key=storage_account_key,
    resource_types=ResourceTypes(object=True),
    permission=AccountSasPermissions(write=True),
    expiry=datetime.utcnow() + timedelta(hours=1)  # Adjust the expiry time
)

# Use SAS token in BlobServiceClient initialization
blob_service_client = BlobServiceClient(account_url=f"https://{storage_account_name}.blob.core.windows.net", credential=sas_token)

# Get BlobClient for the specified blob
blob_client = blob_service_client.get_blob_client(container=container_name, blob=most_recent_backup)

try:
    # Upload the backup file
    with open(os.path.join(backup_dir, most_recent_backup), "rb") as data:
    blob_client.upload_blob(data, overwrite=True)
    print(f"Upload successful: {most_recent_backup}")
except Exception as e:
    print(f"Error uploading file: {e}")

sys.exit()
```

Done! We've successfully backuped our database. This workflow is going to get triggered every hour. Now, we'll take a look at how we can perform a restore.

##### Display 5 most recent backups

We can utilize *list-recent-backups-job.yaml* to list the 5 most recent backups,

```bash
kubectl apply -f list-recent-backups-job.yaml
kubctl logs <pod-name>
```
This will provide with the names of the five most recent backups. This is needed when we need to refer a specific backup when we call *restore-job.yaml* Of course this number can be adjustd to allow to view older backups.

#### Restore database

We can utilize *restore-job.yaml* to restore the database on demand. Whenever we need to restore we can just execute the following command:

```bash
kubectl apply -f restore-job.yaml
```

Let's take a closer look at the yaml definition.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-restore-job
spec:
  template:
    metadata:
      name: postgres-restore
    spec:
      containers:
      - name: restore-backup-container
        image: postgres:latest
        envFrom:
        - secretRef:
            name: postgres-backup-secrets
        - configMapRef:
            name: postgres-backup-config
        command: ["/script/restore_script.sh"]
        volumeMounts:
        - name: backup-volume
          mountPath: /backup
        - name: script
          mountPath: "/script"

      initContainers:
      - name: download-backup-container
        image: python:3.9-slim
        envFrom:
        - secretRef:
            name: postgres-backup-secrets
        - configMapRef:
            name: postgres-backup-config
        command: ["/bin/bash", "-c"]
        args:
          - |
            #!/bin/bash

            pip install azure-storage-blob
            python /script/download_backup.py wiki-db-20231217_081301.sql # Modify before applying
        volumeMounts:
        - name: backup-volume
          mountPath: /backup
        - name: script
          mountPath: "/script"

      restartPolicy: OnFailure
      volumes:
      - name: script
        configMap:
          name: postgres-backup-config
          defaultMode: 0500
      - name: backup-volume
        emptyDir: {}

```

Whenever we like to restore a backup make sure to update the argument the python script receives. This ensures that we download the correct backup before we perform a restore operation. This job operates in a similar fashion as the cronjob. The first container executes a python script to download the backup in the shared volume, and the second container executes *backup_script.sh* which performs the backup.

```bash
#!/bin/bash

POSTGRES_HOST="db-postgresql-service.wiki.svc.cluster.local"
BACKUP_DIR="/backup"

# Find the most recent backup file
BACKUP_FILE=$(ls -t $BACKUP_DIR/wiki-db-*.sql | head -n 1)

echo "Restoring backup from $BACKUP_FILE..."
# Terminate existing connections
psql -U $POSTGRES_USER -h $POSTGRES_HOST -c "SELECT pg_terminate_backend(pg_stat_activity.pid) FROM pg_stat_activity WHERE pg_stat_activity.datname = '$POSTGRES_DB' AND pid <> pg_backend_pid();"
# Drop database
psql -U $POSTGRES_USER -h $POSTGRES_HOST -c "DROP DATABASE \"$POSTGRES_DB\""
# Create database
psql -U $POSTGRES_USER -h $POSTGRES_HOST -c "CREATE DATABASE \"$POSTGRES_DB\""
# Restore the backup
psql -U $POSTGRES_USER -d $POSTGRES_DB -h $POSTGRES_HOST -f "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "Backup restored successfully."
else
    echo "Failed to restore backup."
fi
```

### Conclusion
The combination of Kubernetes CronJobs and Jobs provides a robust solution for managing database backups and restorations. This setup ensures data safety and availability, allowing us to handle backups and restorations efficiently and reliably.