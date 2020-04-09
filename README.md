# K8S NFS-Client Provisioner

The nfs-client-provisioner, as presented by [kubernetes-incubator](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client), is an automatic provisioner that use your existing and already configured NFS server to support dynamic provisioning of Kubernetes Persistent Volumes via Persistent Volume Claims. 

The original version of this tool provisions persistent volumes dynamically with the following name:

``${namespace}-${pvcName}-${pvName}``. 

## Available functionalities

The nfs-client-provisioner is a go binary that offers two functions:
- Create: The function creates the PV with the proprer spec, and the data directory named as follow:``${namespace}-${pvcName}-${pvName}``.
- Delete: If reclaim policy is `Delete`, then this function will be called to delete the PV along side with the mounted volume. If ``archiveOnDelete`` is set to ``true``, the data directory will be renamed as follow: ``archived-${original_name}``.

## What are the limits ?

When moving on with the tests of different use cases using this provisioner, we had the following problems:

- The created data directory is not located exactly where we want it to be. It is always created under another directory named ``${namespace}-${pvcName}-${pvName}``.
- For production measures, we don't use Delete or Recycle reclaim policies, but we would like the PV to be deleted when PVC is no longer needed.
- For the same measures, we like to keep our data directory even after deleting the PV and PVC.

### Side effects of these issues

I will give two different use cases to show you the purpose of this work:

#### First Usecase: Data directory

- Deploy an application that uses a PVC called pvc-nfs.
- Specify the target mounted directory on NFS share, by using ``subPath`` attribute under your containers spec.
- Create The PVC pvc-nfs that uses a storage class pointing to our nfs-client-provisioner.
- A new PV is dynamically created and will connect to the nfs provisioner to add the data directory.
- Data directory is created, but under the name ``${namespace}-${pvcName}-${pvName}``. This name has, in part of it, the PV name (autogenerated).
- Delete and recreate the PV PVC. 

Results:
- The new PV will create a new data directory having its name on it (in a part of it).
- The connection to the old data directory will be lost.

#### Second Usecase: Delete PVC doesn't delete PV automatically

**Using Retain claim policy**
- Deploy an application that uses a PVC called pvc-nfs.
- Create The PVC pvc-nfs that uses a storage class with Retain policy pointing to our nfs-client-provisioner.
- A new PV is dynamically created.
- PVC is destroyed.
- PV changes status to Released, and the only way to make it available is by updating it manually [reference](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)

**Using Delete or Recycle policy**
- PVC destroyed.
- PV deleted and the data directory along side with it.

**Observations:**
- By using Retain policy, the PV is Released but not usable, and the disk space is not recycled.
- By using Delete and Recycle policies, the PV is destroyed and the data directory is lost.
- In the Delete policy, the ``archiveOnDelete`` option archives the data directory, but this could be difficult to handle in a high availability context.

## How to fix them ?

**The Idea**
- Make sure the mounted directory has the same name set in the subPath field on your application container's spec.
- Make sure that, when PVC is deleted, PV is deleted as well, but the mounted data directory is still available for later use.

**The Solution**
I updated the go source code in ``cmd/nfs-client-provisioner/provisioner.go`` as follows:

- Create method: Create the data directory with only using the name given by ``subPath`` attribute.
- Delete method: Delete the PV if the claim is destroyed, without touching the data directory.

## How to deploy nfs-client to your cluster.

You must *already* have an NFS Server.

**Step 1: Get connection information for your NFS server**. Make sure your NFS server is accessible from your Kubernetes cluster and get the information you need to connect to it. At a minimum you will need its hostname.

**Step 2:** Make sure nfs-common package is installed on your cluster nodes.
**Step 3: Get the NFS-Client Provisioner files**. To setup the provisioner you will download a set of YAML files, edit them to add your NFS server's connection information and then apply each with the ``kubectl`` command. 

Get all of the files in the [deploy](https://github.com/ajammali/nfs-client-provisioner/tree/master/deploy) directory of this repository. These instructions assume that you have cloned the [nfs-client-provisioner](https://github.com/ajammali/nfs-client-provisioner) repository and have a bash-shell open in the project's root directory.

**Step 4: Setup authorization**. If your cluster has RBAC enabled you must authorize the provisioner. If you are in a namespace/project other than "default" edit `deploy/rbac.yaml`.

**Step 5: Deploy provisioner.** If your are using an ARM based servers you must use ``deploy/deployment-arm.yml``. Otherwise, you can use ``deploy/deployment.yml``.

Commands:

```sh
$ kubectl create -f deploy/rbac.yml
$ kubectl create -f deploy/deployment.yml
$ kubectl create -f deploy/class.yaml
```

**Step 6: Configure the NFS-Client provisioner**

Note: To deploy to an ARM-based environment, use: `deploy/deployment-arm.yaml` instead, otherwise use `deploy/deployment.yaml`.

Next you must edit the provisioner's deployment file to add connection information for your NFS server. Edit `deploy/deployment.yml` and replace the two occurences of <YOUR NFS SERVER HOSTNAME> with your server's hostname.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: ajammali/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: cluster.local/nfs-provisioner 
            - name: NFS_SERVER
              value: <YOUR NFS SERVER HOSTNAME>
            - name: NFS_PATH
              value: /mnt/nfs
      volumes:
        - name: nfs-client-root
          nfs:
            server: <YOUR NFS SERVER HOSTNAME>
            path: /mnt/nfs
```

You may also want to change the PROVISIONER_NAME above from ``cluster.local/nfs-provisioner`` to something more descriptive, but if you do remember to also change the PROVISIONER_NAME in the storage class definition below:

This is `deploy/class.yml` which defines the NFS-Client's Kubernetes Storage Class:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: cluster.local/nfs-provisioner
parameters:
  archiveOnDelete: "false" # When set to "false" your PVs will not be archived
                           # by the provisioner upon deletion of the PVC.
```

**Step 7: Finally, test your environment!**

Note: You need to make sure that you are using ``subPath`` option in your container's spec. The data directory will have the name of the subPath attribute's value.

Now we'll test your NFS provisioner.

Deploy:

```sh
$ kubectl create -f deploy/test-claim.yaml -f deploy/test-pod.yaml
```

Now check your NFS Server for the file `SUCCESS`.

```sh
kubectl delete -f deploy/test-pod.yaml -f deploy/test-claim.yaml
```

Now check the folder has been deleted.

**Step 8: Deploying your own PersistentVolumeClaims**. To deploy your own PVC, make sure that you have the correct `storage-class` as indicated by your `deploy/class.yaml` file.

For example:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

