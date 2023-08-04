
# OADP (Velero) + OpenShift Pipelines (Tekton) POC

A short proof of concert involving the use of OADP (Velero), Restic and OpenShift Pipelines (Tekton) to automate the backup and restoration of namespace data, to and fro local NFS storage.  

Demo Part 1 (Backup): https://www.youtube.com/watch?v=nQsMf-Lc7FY  
Demo Part 2 (Restore): https://www.youtube.com/watch?v=ut_wI0EHzlk  
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
5. Creating a namespace dedicated for OpenShift Pipelines resources

   We will be creating a new namespace called **pipeline-ns, and all resources related to OpenShift Pipelines (e.g. tasks, pipeline, pipelineRun) will be saved here moving forward.** 
   ```
   $ oc new-project pipeline-ns
   ```
   
6. Creating ObjectBucket Secret  

   We will need to create a Secret object to store our bucket "ACCESS_KEY_ID" and "SECRET_ACCESS_KEY" (which is required on step 2 of our backup task).  
   ```
   $ oc create secret generic bucket-secret -n pipeline-ns \
   --from-literal=AWS_ACCESS_KEY_ID=<your AWS access key ID> \
   --from-literal=AWS_SECRET_ACCESS_KEY=<your AWS secret access key>
   ```
   
7. **(Optional)** Creating ConfigMap for NFS server  

   As my NFS server is an EC2 instance, I have created a ConfigMap object to mount my EC2 identity file into a Tekton workspace (to allow the Tekton pod to SCP into my NFS server).  
   ```
   $ oc create configmap --from-file=./nfsserver.pem -n pipeline-ns
   ```

8. Creating Backup tasks on OpenShift Pipelines (Tekton)  

   To setup the Tekton CLI
   ```
   $ wget https://github.com/tektoncd/cli/releases/download/v0.31.2/tkn_0.31.2_Linux_x86_64.tar.gz

   $ tar -xvzf tkn_0.31.2_Linux_x86_64.tar.gz
   LICENSE
   README.md
   tkn

   $ sudo chmod 775 tkn
   $ sudo cp tkn /usr/bin

   $ tkn version
   Client version: 0.31.2
   Pipeline version: v0.44.4
   Triggers version: v0.23.1
   Operator version: v0.65.1
   ```

   To allow the Pipelines ServiceAccount to work with OADP resources, create a new Role and RoleBinding with the appropriate permissions.
   ```
   # Error message without SCC
   from server for: "STDIN": backups.velero.io "backup" is forbidden: User "system:serviceaccount:pipeline-ns:pipeline" cannot get resource "backups" in API group "velero.io" in the namespace "openshift-adp"

   $ vi pipelines-oadp-role.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: pipelines-oadp-role
   rules:
     - apiGroups:
         - 'velero.io'
       resources:
         - backups
       verbs:
         - '*'
   
   $ vi pipelines-oadp-rb.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: pipelines-oadp-rb
   subjects:
     - kind: ServiceAccount
       name: pipeline
       namespace: pipeline-ns
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: pipelines-oadp-role

   $ oc apply -f pipelines-oadp-role.yaml
   $ oc apply -f pipelines-oadp-rb.yaml
   ```   
   ```
   $ vi step1-backup-createbackup

   # Task 1: Create Backup CR
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: step1-backup-createbackup
     namespace: pipeline-ns
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
     namespace: pipeline-ns
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
             aws s3 sync s3://$(params.bucket-name)$(workspaces.bucket-prefix.path)/backups/$(params.name-of-backup)/ $(workspaces.bucket-prefix.path)/backups/$(params.name-of-backup)/
             aws s3 sync s3://$(params.bucket-name)$(workspaces.bucket-prefix.path)/restic/ $(workspaces.bucket-prefix.path)/restic/$(params.namespace-to-backup)/

   $ oc apply -f step2-backup-s3toworkspace.yaml
   ```
   ```
   $ vi step3-backup-workspacetonfs.yaml
   # Task 3: SCP files from Tekton workspace into local NFS server
   # As pods are ephemeral by nature, we need to add the host fingerprint manually for every instance of a task pod (else the pipeline will fail due to permission issues).
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
       - name: nfs-user
       - name: nfs-server-ip
       - name: nfs-directory
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
             scp -ri "$(workspaces.nfsserver.pem.path)$(workspaces.nfsserver.pem.path)" $(workspaces.bucket-prefix.path)/ $(params.nfs-user)@$(params.nfs-server-ip):$(params.nfs-directory)

   $ oc apply -f step3-backup-workspacetonfs
   ```

9. Creating Pipeline and PipelineRun resources  

   Description of OADP parameters:
   - name-of-backup: Name of backup CR (don't create multiple backups of same name, Pipeline will not take effect)
   - namespace-to-backup: Name of namespace to be backed up
   - oadp-dpa-location: DPA Location name (oc get BackupStorageLocation -n openshift-adp)
   - bucket-name: Name of S3 bucket to store your Velero data files
   - bucket-secret: Secret Object name (which consists of your bucket access key id, secret key etc.)
   - ssh-fingerprint: SSH fingerprint of NFS server (public key)
   - nfs-user: Name of user of NFS server
   - nfs-server-ip: IP address of NFS server
   - nfs-directory: Directory of NFS server, where the Velero files will be stored in

   To edit the default values of the Pipeline parameters, you can change it directly from the YAML file or via the Pipeline Builder GUI in the OpenShift console (Actions > Edit Pipeline).
   ```
   $ vi backup-pipeline.yaml
   apiVersion: tekton.dev/v1beta1
   kind: Pipeline
   metadata:
     name: backup-pipeline
     namespace: pipeline-ns
   spec:
     params:
       - name: name-of-backup
         default: backup
       - name: namespace-to-backup
         default: mysql-persistent
       - name: oadp-dpa-location
         default: default-oadp-1
       - name: bucket-name
         default: s3-oadp
       - name: bucket-secret
         default: bucket-secret
       - name: ssh-fingerprint
         default: >-
           ssh-rsa
           AAAAB3NzaC1yc2EAAAADAQABAAABgQC0FTxXx5a6q3eKBBHNE8Kzd8MojdCsNjxHK3iPjnorxhddURwQsnXY6Dfu+FN2dI9GfHetv+2dSdooYDzlD61kf07BLDbA2wmIIGjruxLOUIsiXnRnYz1Df6Inhja1CYNDDuOpc0puO9ZhAMbQaCkq4EIBt6WPVTNQbybKeyTqnbzI2ksKjcIN46tJXpP7x89sYhFm0PKs9+52LdDmapOKLgTy2LvJpyN7txBJXP30l06s0zckAf4SHVUlhJNJimup3TMC1K2CXMkTQga0hQn/rnujgg6HrWLZxspG/laAamzj22ylPcfPyNMy3qvShQusUv5kJb6SNDXrK8kZRNsepyhMBSMydqCCcXO/XcmqbGv0AihQ2xB5rUYgf9tdwFTtL2AJVyt6FnilV5fFnI+1Twy7pRCDvkmfsStYiO6P79FOhGs0H7wCUG3Fg1inQyxP7MYq8wouejWniWFZmkNSabD/xOr8zErF/wHxm+7H8FriboHyZ85FEECrGVKAGD0=
           root@ip-192-168-1-90.ap-southeast-1.compute.internal
       - name: nfs-user
         default: ec2-user
       - name: nfs-server-ip
         default: 192.168.1.90
       - name: nfs-directory
         default: /home/ec2-user
     workspaces:
       - name: bucket-prefix
         optional: false
       - name: nfsserver.pem
         optional: false
     tasks:
       - name: step1-backup-createbackup
         taskRef:
           kind: Task
           name: step1-backup-createbackup
         params:
           - name: name-of-backup
             value: $(params.name-of-backup)
           - name: namespace-to-backup
             value: $(params.namespace-to-backup)
           - name: oadp-dpa-location
             value: $(params.oadp-dpa-location)
       - name: step2-backup-s3toworkspace
         runAfter:
           - step1-backup-createbackup
         taskRef:
           kind: Task
           name: step2-backup-s3toworkspace
         params:
           - name: name-of-backup
             value: $(params.name-of-backup)
           - name: namespace-to-backup
             value: $(params.namespace-to-backup)
           - name: bucket-name
             value: $(params.bucket-name)
           - name: bucket-secret
             value: $(params.bucket-secret)
         workspaces:
           - name: bucket-prefix
             workspace: bucket-prefix
       - name: step3-backup-workspacetonfs
         runAfter:
           - step2-backup-s3toworkspace
         taskRef:
           kind: Task
           name: step3-backup-workspacetonfs
         params:
           - name: name-of-backup
             value: $(params.name-of-backup)
           - name: bucket-name
             value: $(params.bucket-name)
           - name: bucket-secret
             value: $(params.bucket-secret)
           - name: ssh-fingerprint
             value: $(params.ssh-fingerprint)
           - name: nfs-user
             value: $(params.nfs-user)
           - name: nfs-server-ip
             value: $(params.nfs-server-ip)
           - name: nfs-directory
             value: $(params.nfs-directory)
         workspaces:
           - name: bucket-prefix
             workspace: bucket-prefix
           - name: nfsserver.pem
             workspace: nfsserver.pem

   $ oc apply -f backup-pipeline.yaml
   ```
   ```
   $ vi backup-pipelinerun.yaml

   apiVersion: tekton.dev/v1
   kind: PipelineRun
   metadata:
     name: backup-pipelinerun
     namespace: pipeline-ns
   spec:
     params:
     - name: name-of-backup
       value: backup
     - name: namespace-to-backup
       value: mysql-persistent
     - name: oadp-dpa-location
       value: default-oadp-1
     - name: bucket-name
       value: s3-oadp
     - name: bucket-secret
       value: bucket-secret
     - name: ssh-fingerprint
       value: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC0FTxXx5a6q3eKBBHNE8Kzd8MojdCsNjxHK3iPjnorxhddURwQsnXY6Dfu+FN2dI9GfHetv+2dSdooYDzlD61kf07BLDbA2wmIIGjruxLOUIsiXnRnYz1Df6Inhja1CYNDDuOpc0puO9ZhAMbQaCkq4EIBt6WPVTNQbybKeyTqnbzI2ksKjcIN46tJXpP7x89sYhFm0PKs9+52LdDmapOKLgTy2LvJpyN7txBJXP30l06s0zckAf4SHVUlhJNJimup3TMC1K2CXMkTQga0hQn/rnujgg6HrWLZxspG/laAamzj22ylPcfPyNMy3qvShQusUv5kJb6SNDXrK8kZRNsepyhMBSMydqCCcXO/XcmqbGv0AihQ2xB5rUYgf9tdwFTtL2AJVyt6FnilV5fFnI+1Twy7pRCDvkmfsStYiO6P79FOhGs0H7wCUG3Fg1inQyxP7MYq8wouejWniWFZmkNSabD/xOr8zErF/wHxm+7H8FriboHyZ85FEECrGVKAGD0=
         root@ip-192-168-1-90.ap-southeast-1.compute.internal
     - name: nfs-user
       value: ec2-user
     - name: nfs-server-ip
       value: 192.168.1.90
     - name: nfs-directory
       value: /home/ec2-user
     pipelineRef:
       name: backup-pipeline
     taskRunTemplate:
       serviceAccountName: pipeline
     timeouts:
       pipeline: 1h0m0s
     workspaces:
     - name: bucket-prefix
       volumeClaimTemplate:
         spec:
           accessModes:
           - ReadWriteOnce
           resources:
             requests:
               storage: 1Gi
           storageClassName: gp3-csi
           volumeMode: Filesystem
     - configMap:
         name: nfsserver.pem
       name: nfsserver.pem

   $ oc apply -f backup-pipelinerun.yaml
   ```

11. Upon successful execution of the Pipeline, the Velero backup data files should appear in your local NFS server.
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
The restoration pipeline would be the exact same steps as the backup pipeline, but in reverse order.  

Pipeline is executed with the assumptions listed below:  
- Your Velero backup data files exist only in your local NFS server.  
- Your S3 bucket is empty.  
- Restoration is performed in another cluster. (Pipeline can be executed in the same cluster, but delete the existing backupRepository before restoring - see [Bugs Encountered](#bugs-encountered)).

1. Creating Backup tasks on OpenShift Pipelines (Tekton)
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
       - name: nfs-user
       - name: nfs-server-ip
       - name: nfs-directory
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
             scp -ri "$(workspaces.nfsserver.pem.path)$(workspaces.nfsserver.pem.path)" $(params.nfs-user)@$(params.nfs-server-ip):$(params.nfs-directory) $(workspaces.bucket-prefix.path)/
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
             sleep 60 # Buffer time for OADP operator to detect Velero backup data files
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
             EOF
   ```
2. Creating restore Pipeline and PipelineRun CR
   ```
   $ vi restore-pipeline.yaml
   
   apiVersion: tekton.dev/v1beta1
   kind: Pipeline
   metadata:
     name: restore-pipeline
     namespace: pipeline-ns
   spec:
     params:
       - name: name-of-backup
         default: backup
       - name: bucket-name
         default: s3-oadp
       - name: bucket-secret
         default: bucket-secret
       - name: ssh-fingerprint
         default: >-
           AAAAB3NzaC1yc2EAAAADAQABAAABgQC0FTxXx5a6q3eKBBHNE8Kzd8MojdCsNjxHK3iPjnorxhddURwQsnXY6Dfu+FN2dI9GfHetv+2dSdooYDzlD61kf07BLDbA2wmIIGjruxLOUIsiXnRnYz1Df6Inhja1CYNDDuOpc0puO9ZhAMbQaCkq4EIBt6WPVTNQbybKeyTqnbzI2ksKjcIN46tJXpP7x89sYhFm0PKs9+52LdDmapOKLgTy2LvJpyN7txBJXP30l06s0zckAf4SHVUlhJNJimup3TMC1K2CXMkTQga0hQn/rnujgg6HrWLZxspG/laAamzj22ylPcfPyNMy3qvShQusUv5kJb6SNDXrK8kZRNsepyhMBSMydqCCcXO/XcmqbGv0AihQ2xB5rUYgf9tdwFTtL2AJVyt6FnilV5fFnI+1Twy7pRCDvkmfsStYiO6P79FOhGs0H7wCUG3Fg1inQyxP7MYq8wouejWniWFZmkNSabD/xOr8zErF/wHxm+7H8FriboHyZ85FEECrGVKAGD0=         
           root@ip-192-168-1-90.ap-southeast-1.compute.internal
       - name: nfs-user
         default: ec2-user
       - name: nfs-server-ip
         default: 192.168.1.90
       - name: nfs-directory
         default: /home/ec2-user/
       - name: name-of-restore
         default: restore
       - name: namespace-to-restore
         default: mysql-persistent
       - name: oadp-dpa-location
         default: default-oadp-1
     workspaces:
       - name: bucket-prefix
         optional: false
       - name: nfsserver.pem
         optional: false
     tasks:
       - name: step1-restore-nfstoworkspace
         taskRef:
           kind: Task
           name: step1-restore-nfstoworkspace
         params:
           - name: name-of-backup
             value: $(params.name-of-backup)
           - name: bucket-name
             value: $(params.bucket-name)
           - name: bucket-secret
             value: $(params.bucket-secret)
           - name: ssh-fingerprint
             value: $(params.ssh-fingerprint)
           - name: nfs-user
             value: $(params.nfs-user)
           - name: nfs-server-ip
             value: $(params.nfs-server-ip)
           - name: nfs-directory
             value: $(params.nfs-directory)
         workspaces:
           - name: bucket-prefix
             workspace: bucket-prefix
           - name: nfsserver.pem
             workspace: nfsserver.pem
       - name: step2-restore-workspacetos3
         runAfter:
           - step1-restore-nfstoworkspace
         taskRef:
           kind: Task
           name: step2-restore-workspacetos3
         params:
           - name: name-of-backup
             value: $(params.name-of-backup)
           - name: bucket-name
             value: $(params.bucket-name)
           - name: bucket-secret
             value: $(params.bucket-secret)
         workspaces:
           - name: bucket-prefix
             workspace: bucket-prefix
       - name: step3-restore-createrestore
         runAfter:
           - step2-restore-workspacetos3
         taskRef:
           kind: Task
           name: step3-restore-createrestore
         params:
           - name: name-of-restore
             value: $(params.name-of-restore)
           - name: name-of-backup
             value: $(params.name-of-backup)
           - name: namespace-to-restore
             value: $(params.namespace-to-restore)
           - name: oadp-dpa-location
             value: $(params.oadp-dpa-location)
   
   $ vi restore-pipelinerun.yaml
   apiVersion: tekton.dev/v1
   kind: PipelineRun
   metadata:
     name: restore-pipeline-qxbrbt
     namespace: pipeline-ns
   spec:
     params:
     - name: name-of-backup
       value: backup
     - name: bucket-name
       value: s3-oadp
     - name: bucket-secret
       value: bucket-secret
     - name: ssh-fingerprint
       value: AAAAB3NzaC1yc2EAAAADAQABAAABgQC0FTxXx5a6q3eKBBHNE8Kzd8MojdCsNjxHK3iPjnorxhddURwQsnXY6Dfu+FN2dI9GfHetv+2dSdooYDzlD61kf07BLDbA2wmIIGjruxLOUIsiXnRnYz1Df6Inhja1CYNDDuOpc0puO9ZhAMbQaCkq4EIBt6WPVTNQbybKeyTqnbzI2ksKjcIN46tJXpP7x89sYhFm0PKs9+52LdDmapOKLgTy2LvJpyN7txBJXP30l06s0zckAf4SHVUlhJNJimup3TMC1K2CXMkTQga0hQn/rnujgg6HrWLZxspG/laAamzj22ylPcfPyNMy3qvShQusUv5kJb6SNDXrK8kZRNsepyhMBSMydqCCcXO/XcmqbGv0AihQ2xB5rUYgf9tdwFTtL2AJVyt6FnilV5fFnI+1Twy7pRCDvkmfsStYiO6P79FOhGs0H7wCUG3Fg1inQyxP7MYq8wouejWniWFZmkNSabD/xOr8zErF/wHxm+7H8FriboHyZ85FEECrGVKAGD0=          root@ip-192-168-1-90.ap-southeast-1.compute.internal
     - name: nfs-user
       value: ec2-user
     - name: nfs-server-ip
       value: 192.168.1.90
     - name: nfs-directory
       value: /home/ec2-user/
     - name: name-of-restore
       value: restore
     - name: namespace-to-restore
       value: mysql-persistent
     - name: oadp-dpa-location
       value: default-oadp-1
     pipelineRef:
       name: restore-pipeline
     taskRunTemplate:
       serviceAccountName: pipeline
     timeouts:
       pipeline: 1h0m0s
     workspaces:
     - name: bucket-prefix
       volumeClaimTemplate:
         spec:
           accessModes:
           - ReadWriteOnce
           resources:
             requests:
               storage: 1Gi
           storageClassName: gp3-csi
           volumeMode: Filesystem
     - configMap:
         name: nfsserver.pem
       name: nfsserver.pem

   $ oc apply -f restore-pipeline.yaml
   $ oc apply -f restore-pipelinerun.yaml
   ```
**The namespace data (and also any applications that reside in it) should be restored to a desired state.**
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
