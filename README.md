
# OADP (Velero) + OpenShift Pipelines (Tekton) POC

A short proof of concert involving the use of OADP (Velero), Restic and OpenShift Pipelines (Tekton) to automate the backup and restoration of namespace data into local NFS storage.  

Part 1: https://www.youtube.com/watch?v=nQsMf-Lc7FY  
Part 2: https://www.youtube.com/watch?v=ut_wI0EHzlk  
## Backup Architecture
![Screenshot 2023-07-22 004822](https://github.com/shingweihao/oadp-pipelines-poc/assets/122070690/b92f707e-b057-448f-9a2c-ae76dbf50dc9)
### Flow of architecture:  
1. Create Backup CR
2. OADP retrieves data and resources from target namespace, Velero backup data files will be created.
3. OADP stores Velero backup data files into S3 bucket (defined in our DataProtectionApplication resource).
4. Retrieve Velero backup data files from S3 bucket and save into Tekton workspace temporarily.
5. Retrieve Velero backup data files from Tekton workspace into local NFS storage.

## Instructions
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
4. (Optional) Deploying an application  

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
5. Creating Backup tasks on OpenShift Pipelines (Tekton)

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
         default: backup
       - name: namespace-to-backup
         default: mysql-persistent
       - name: oadp-dpa-location
         default: default-oadp-1
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
         default: backup
      - name: namespace-to-backup
         default: mysql-persistent
       - name: bucket-name
         default: s3-oadp
       - name: bucket-secret
         default: bucket-secret
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
         default: backup
       - name: bucket-name
         default: s3-oadp
       - name: bucket-secret
         default: bucket-secret
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
             echo "192.168.0.177 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBC/aSgXr0SFkpxgizzhGCIjdnCj3w/89IzWzwiterN32RE4SplWqRlA+5PtxH9fPX64//bI5Pc7e25JZhJTL8eE=" > ~/.ssh/known_hosts #Change to your host fingerprint manually
             scp -ri "$(workspaces.nfsserver.pem.path)/nfsserver.pem" $(workspaces.bucket-prefix.path)/ ec2-user@192.168.0.177:/home/ec2-user/

   $ oc apply -f step3-backup-workspacetonfs
   ```
