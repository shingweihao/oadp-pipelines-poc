
# OADP (Velero) + OpenShift Pipelines (Tekton) POC

A short proof of concert involving the use of OADP (Velero), Restic and OpenShift Pipelines (Tekton) to automate the backup and restoration of namespace data into local NFS storage.  

Part 1: https://www.youtube.com/watch?v=nQsMf-Lc7FY  
Part 2: https://www.youtube.com/watch?v=ut_wI0EHzlk  
## Backup Architecture
![Screenshot 2023-07-22 004822](https://github.com/shingweihao/oadp-pipelines-poc/assets/122070690/b92f707e-b057-448f-9a2c-ae76dbf50dc9)
### Flow Of Architecture:  
1. Create Backup CR
2. OADP retrieves data and resources from target namespace, Velero backup data files will be created.
3. OADP stores Velero backup data files into S3 bucket (defined in our DataProtectionApplication resource).
4. Retrieve Velero backup data files from S3 bucket and save into Tekton workspace temporarily.
5. Retrieve Velero backup data files from Tekton workspace into local NFS storage.

## Backup Pipeline Instructions
1. Install the OADP (stable-1.2) and OpenShift Pipelines (1.10.4) operators from the Operator Hub.  
   ![image](https://github.com/shingweihao/oadp-pipelines-poc/assets/122070690/a27fdf8d-e5ba-467c-8ada-72f93536f366)![image](https://github.com/shingweihao/oadp-pipelines-poc/assets/122070690/bc22d7d5-d2f5-45d9-9bf0-cdc65e7a1794)

2. Ensure that you have an S3 bucket (or any compatible S3 storage) setup, as OADP only allows for an ObjectBucket storage.
   
   Create the cloud-credentials Secret necessary for your OADP DataProtectionApplication resource.  
   ```
   $ vi credentials-velero  

   [default]  
   aws_access_key_id=<your AWS access key ID>  
   aws_secret_access_key=<your AWS secret access key>

   $ oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=./credentials-velero
   ```

3. Create the DataProtectionApplication resource on OADP.  
   ```
   $ vi dpa.yaml

   apiVersion: oadp.openshift.io/v1alpha1
   kind: DataProtectionApplication
   metadata:
     name: default-oadp
     namespace: openshift-adp
   spec:
     backupLocations:
       - velero:
           config:
             profile: default
             region: ap-southeast-1
           credential:
             key: cloud
             name: cloud-credentials
           default: true
           objectStorage:
             bucket: <YOUR BUCKET NAME HERE>
             prefix: sno #Folder where Velero files will be stored, in this example I will be using /sno for all tasks. Change the bucket-prefix in the tasks whenever necessary.
           provider: aws
     configuration:
       restic:
         enable: true
       velero:
         defaultPlugins:
           - openshift
           - csi
           - aws
         resourceTimeout: 10m
     snapshotLocations:
       - velero:
           config:
             profile: default
             region: ap-southeast-1
           provider: aws

   $ oc apply -f dpa.yaml
   ```
   Ensure that your BackupStorageLocation shows up as Available.  
   (If it doesn't, something is configured wrongly and DPA resource is unable to communicate with your bucket.)
   ```
   $ oc get BackupStorageLocation -n openshift-adp         
   NAME             PHASE       LAST VALIDATED   AGE   DEFAULT
   default-oadp-1   Available   60s              11m   true
   ```
4. **(Optional)** Deploying an application  

   I will be deploying a stateful MySQL application in the **mysql-persistent** namespace.  
   This same application will be used for the backup and restoration process in this example.
   
   (Credits to https://github.com/openshift/oadp-operator/tree/master/tests/e2e/sample-applications/mysql-persistent)
   ```
   $ vi mysql-persistent.yaml
   
   apiVersion: v1
   kind: List
   items:
     - kind: Namespace
       apiVersion: v1
       metadata:
         name: mysql-persistent
         labels:
           app: mysql
     - apiVersion: v1
       kind: ServiceAccount
       metadata:
         name: mysql-persistent-sa
         namespace: mysql-persistent
         labels:
           component: mysql-persistent
     - apiVersion: v1
       kind: PersistentVolumeClaim
       metadata:
         name: mysql
         namespace: mysql-persistent
       spec:
         accessModes:
         - ReadWriteOnce
         resources:
           requests:
             storage: 1Gi
         storageClassName: <YOUR STORAGECLASS NAME>
     - kind: SecurityContextConstraints
       apiVersion: security.openshift.io/v1
       metadata:
         name: mysql-persistent-scc
       allowPrivilegeEscalation: true
       allowPrivilegedContainer: true
       runAsUser:
         type: RunAsAny
       seLinuxContext:
         type: RunAsAny
       fsGroup:
         type: RunAsAny
       supplementalGroups:
         type: RunAsAny
       volumes:
       - '*'
       users:
       - system:admin
       - system:serviceaccount:mysql-persistent:mysql-persistent-sa
     - apiVersion: v1
       kind: Service
       metadata:
         annotations:
           template.openshift.io/expose-uri: mariadb://{.spec.clusterIP}:{.spec.ports[?(.name=="mysql")].port}
         name: mysql
         namespace: mysql-persistent
         labels:
           app: mysql
           service: mysql
       spec:
         ports:
         - protocol: TCP
           name: mysql
           port: 3306
         selector:
           app: mysql
     - apiVersion: apps/v1
       kind: Deployment
       metadata:
         annotations:
           template.alpha.openshift.io/wait-for-ready: 'true'
         name: mysql
         namespace: mysql-persistent
         labels:
           e2e-app: "true"
       spec:
         selector:
           matchLabels:
             app: mysql
         strategy:
           type: Recreate
         template:
           metadata:
             labels:
               e2e-app: "true"
               app: mysql
           spec:
             securityContext:
               runAsGroup: 27
               runAsUser: 27
               fsGroup: 27
               privileged: true
             serviceAccountName: mysql-persistent-sa
             containers:
             - image: registry.redhat.io/rhel8/mariadb-105:latest
               name: mysql
               securityContext:
                 runAsGroup: 27
                 runAsUser: 27
                 fsGroup: 27
                 privileged: false
               env:
                 - name: MYSQL_USER
                   value: changeme
                 - name: MYSQL_PASSWORD
                   value: changeme
                 - name: MYSQL_ROOT_PASSWORD
                   value: root
                 - name: MYSQL_DATABASE
                   value: todolist
               ports:
               - containerPort: 3306
                 name: mysql
               resources:
                 limits:
                   memory: 512Mi
               volumeMounts:
               - name: mysql-data
                 mountPath: /var/lib/mysql
             volumes:
             - name: mysql-data
               persistentVolumeClaim:
                 claimName: mysql
     - apiVersion: v1
       kind: Service
       metadata:
         name: todolist
         namespace: mysql-persistent
         labels:
           app: todolist
           service: todolist
           e2e-app: "true"
       spec:
         ports:
           - name: web
             port: 8000
             targetPort: 8000
         selector:
           app: todolist
           service: todolist
     - apiVersion: apps.openshift.io/v1
       kind: DeploymentConfig
       metadata:
         name: todolist
         namespace: mysql-persistent
         labels:
           app: todolist
           service: todolist
           e2e-app: "true"
       spec:
         replicas: 1
         selector:
           app: todolist
           service: todolist
         strategy:
           type: Recreate
         template:
           metadata:
             labels:
               app: todolist
               service: todolist
               e2e-app: "true"
           spec:
             containers:
             - name: todolist
               image: quay.io/konveyor/todolist-mariadb-go:v2_4
               env:
                 - name: foo
                   value: bar
               ports:
                 - containerPort: 8000
                   protocol: TCP
             initContainers:
             - name: init-myservice
               image: registry.access.redhat.com/ubi8/ubi:latest
               command: ['sh', '-c', 'sleep 10; until getent hosts mysql; do echo waiting for mysql; sleep 5; done;']
     - apiVersion: route.openshift.io/v1
       kind: Route
       metadata:
         name: todolist-route
         namespace: mysql-persistent
       spec:
         path: "/"
         to:
           kind: Service
           name: todolist

   $ oc apply -f mysql-persistent.yaml

   # If the application is unable to deploy due to permission issues, assign privileged SCC to mysql-persistent ServiceAccount.
   $ oc adm policy add-scc-to-user privileged system:serviceaccount:mysql-persistent:mysql-persistent-sa
   ```
4. Creating ObjectBucket Secret  

   We will need to create a Secret object to store our bucket's "ACCESS_KEY_ID" and "SECRET_ACCESS_KEY", as required on step 2 of our backup task.  

   Ensure that your Secret is in:
   - The same namespace as where your Pipelines will be running
   - Different namespace from your application that you're backing up

   I will be saving it under the "default" namespace, as my Pipelines will be running in the "default" namespace as well.  
   ```
   $ oc create secret generic bucket-secret --from-literal=AWS_ACCESS_KEY_ID=<your AWS access key ID> --from-literal=AWS_SECRET_ACCESS_KEY=<your AWS secret access key> -n default
   ```
   
6. **(Optional)** Creating ConfigMap for NFS server  

   As my NFS server is an EC2 instance, I have created a ConfigMap object to mount my SSH identity file onto another Tekton workspace.
   
   Ensure that your ConfigMap is in:
   - The same namespace as where your Pipelines will be running
   - Different namespace from your application that you're backing up

    I will be saving it under the "default" namespace, as my Pipelines will be running in the "default" namespace as well.    
   ```
   $ oc create configmap --from-file=./nfsserver.pem -n default
   ```

7. Creating Backup tasks on OpenShift Pipelines (Tekton)

   ```
   $ vi step1-backup-createbackup

   # Task 1: Create Backup CR
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: step1-backup-createbackup
   spec:
     params:
       - name: name-of-backup
       - name: namespace-to-backup
       - name: oadp-dpa-location
     steps:
       - name: create-backup
         image: image-registry.openshift-image-registry.svc:5000/openshift/cli:latest
         command: ["/bin/bash", "-c"]
         args:
           - |-
             cat << EOF | oc apply -f -
             apiVersion: velero.io/v1
             kind: Backup
             metadata:
               name: $(params.name-of-backup)
               labels:
                 velero.io/storage-location: default
               namespace: openshift-adp
             spec:
               defaultVolumesToRestic: true
               includedNamespaces:
               - $(params.namespace-to-backup)
               storageLocation: $(params.oadp-dpa-location)
             EOF

   $ oc apply -f backup-tasks/step1-backup-createbackup
   ```
   ```
   $ vi step2-backup-s3toworkspace.yaml

   # Task 2: Retrieve Velero backup file from S3 and save into Tekton workspace
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: step2-backup-s3toworkspace
   spec:
     workspaces:
       - name: bucket-prefix
         mountPath: /sno
     params:
       - name: name-of-backup
       - name: namespace-to-backup
       - name: bucket-name
       - name: bucket-secret
     steps:
       - name: s3-to-workspace
         image: amazon/aws-cli
         envFrom:
           - secretRef:
               name: $(params.bucket-secret)
         command: ["/bin/bash", "-c"]
         args:
           - |-
             aws s3 sync s3://$(params.bucket-name)/sno/backups/$(params.name-of-backup)/ $(workspaces.bucket-prefix.path)/backups/$(params.name-of-backup)/
             aws s3 sync s3://$(params.bucket-name)/sno/restic/ $(workspaces.bucket-prefix.path)/restic/$(params.namespace-to-backup)/

   $ oc apply -f step2-backup-s3toworkspace.yaml
   ```
   ```
   $ vi step3-backup-workspacetonfs.yaml
   
   # Task 3: SCP files from Tekton workspace into local NFS server
   # Note: As pods are ephemeral for each TaskRun, we need to add the host fingerprint manually in the task. If not, the pipeline will fail due to permission issues.
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: step3-backup-workspacetonfs
   spec:
     workspaces:
       - name: bucket-prefix
         mountPath: /sno
       - name: nfsserver.pem
         mountPath: /nfsserver.pem
     params:
       - name: name-of-backup
       - name: bucket-name
       - name: bucket-secret
       - name: ssh-fingerprint
         default: <your-nfs-server-fingerprint-here>
     steps:
       - name: workspace-to-nfs
         image: quay.io/devfile/base-developer-image:ubi8-latest
         envFrom:
           - secretRef:
               name: $(params.bucket-secret)
         command: ["/bin/bash", "-c"]
         args:
           - |-
             mkdir ~/.ssh/
             echo "$(params.ssh-fingerprint)" > ~/.ssh/known_hosts
             scp -ri "$(workspaces.nfsserver.pem.path)/nfsserver.pem" $(workspaces.bucket-prefix.path)/ <user>@<your-nfs-server-ip>:<directory>

   $ oc apply -f step3-backup-workspacetonfs
   ```
8. Creating Pipeline and PipelineRun resources
   
   You may refer to the PoC video (https://www.youtube.com/watch?v=nQsMf-Lc7FY) @ 7:55 onwards, to observe how the Pipeline can be created via GUI.  
   (Ensure that the Pipeline and PipelineRun resources are created on the same namespace as your Bucket Secret and ConfigMap objects.)  

9. Upon successful execution of the Pipeline, the Velero backup data files should show up on your local NFS server.
   ```
   [ec2-user@ip-192-168-0-90 ~]$ cd sno

   [ec2-user@ip-192-168-0-90 sno]$ pwd
   /home/ec2-user/sno
   
   [ec2-user@ip-192-168-0-90 sno]$ ls
   backups  restic
   
   [ec2-user@ip-192-168-0-90 sno]$ ls backups/backup/
   backup-csi-volumesnapshotclasses.json.gz   backup-resource-list.json.gz
   backup-csi-volumesnapshotcontents.json.gz  backup-results.gz
   backup-csi-volumesnapshots.json.gz         backup.tar.gz
   backup-itemoperations.json.gz              backup-volumesnapshots.json.gz
   backup-logs.gz                             velero-backup.json
   backup-podvolumebackups.json.gz
   
   [ec2-user@ip-192-168-0-90 sno]$ ls restic/mysql-persistent/
   config  data  index  keys  snapshots
   ```
## Restoration Pipeline Instructions   
The restoration pipeline would be the exact same steps as the backup pipeline, but in a reverse manner.  

It is made with the assumptions listed below:  
1. Your Velero backup data files exist only in your local NFS server, and not in your S3 bucket.  
2. Performed in a new cluster (it is doable in the same cluster chances are you will encounter the bug listed below).  

Creating Backup tasks on OpenShift Pipelines (Tekton)
   ```
   # Task 1: SCP files from NFS server into Tekton workspace
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: step1-restore-nfstoworkspace
   spec:
     workspaces:
       - name: bucket-prefix
         mountPath: /sno
       - name: nfsserver.pem
         mountPath: /nfsserver.pem
     params:
       - name: name-of-backup
       - name: bucket-name
       - name: bucket-secret
       - name: ssh-fingerprint
         default: <your-nfs-server-fingerprint-here>
     steps:
       - name: nfs-to-workspace
         image: quay.io/devfile/base-developer-image:ubi8-latest
         envFrom:
           - secretRef:
               name: $(params.bucket-secret)
         command: ["/bin/bash", "-c"]
         args:
           - |-
             mkdir ~/.ssh/
             echo "$(params.ssh-fingerprint)" > ~/.ssh/known_hosts
             scp -ri "$(workspaces.nfsserver.pem.path)/nfsserver.pem" <user>@<your-nfs-server-ip>:<directory> $(workspaces.bucket-prefix.path)/
   ```
   ```
   # Task 2: Retrieve Velero backup file from Tekton workspace and save into S3
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: step2-restore-workspacetos3
   spec:
     workspaces:
       - name: bucket-prefix
         mountPath: /sno
     params:
       - name: name-of-backup
       - name: bucket-name
       - name: bucket-secret
     steps:
       - name: workspace-to-s3
         image: amazon/aws-cli
         envFrom:
           - secretRef:
               name: $(params.bucket-secret)
         command: ["/bin/bash", "-c"]
         args:
           - |-
             aws s3 sync $(workspaces.bucket-prefix.path)/ s3://$(params.bucket-name)/
             sleep 60 #Buffer time for OADP operator to detect Velero backup data files
   ```
   ```
   # Task 3: Create Restore CR
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: step3-restore-createrestore
   spec:
     params:
       - name: name-of-restore
       - name: name-of-backup
       - name: namespace-to-restore
       - name: oadp-dpa-location
     steps:
       - name: create-backup
         image: image-registry.openshift-image-registry.svc:5000/openshift/cli:latest
         command: ["/bin/bash", "-c"]
         args:
           - |-
             cat << EOF | oc apply -f -
             apiVersion: velero.io/v1
             kind: Restore
             metadata:
               name: $(params.name-of-restore)
               namespace: openshift-adp
             spec:
               backupName: $(params.name-of-backup)
               restorePVs: true
             EOF
   ```
**End result: The namespace that you've backed up should be restored to the state when the backup was created.**
![image](https://github.com/shingweihao/oadp-pipelines-poc/assets/122070690/2de01da8-1751-46d9-8397-659480d30a5e)


## Bugs Encountered
If you create a Restic `Backup` CR for a namespace, empty the object storage bucket, and then recreate the `Backup` CR for the same namespace, the recreated `Backup` CR fails. 

The `velero` pod log displays the following error message: `stderr=Fatal: unable to open config file: Stat: The specified key does not exist.\nIs there a repository at the following location?`.  

**Cause**  
Velero does not recreate or update the Restic repository from the `ResticRepository` manifest if the Restic directories are deleted from object storage. See [Velero issue 4421](https://github.com/vmware-tanzu/velero/issues/4421) for more information.  

**Solution**  
Remove the related Restic repository from the namespace by running the following command:  
```
$ oc delete backuprepositories.velero.io -n openshift-adp <name_of_backup_repository>
```
