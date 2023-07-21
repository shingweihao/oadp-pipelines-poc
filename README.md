
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
3. 
   Create the cloud-credentials Secret necessary for your OADP DataProtectionApplication resource.  
   ```
   $ vi credentials-velero  

   [default]  
   aws_access_key_id=<your AWS access key ID>  
   aws_secret_access_key=<your AWS secret access key>

   $ oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=./credentials-velero
   ```

4. Create the DataProtectionApplication resource on OADP.  
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
             prefix: <FOLDER WHERE VELERO FILES WILL BE STORED>
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
