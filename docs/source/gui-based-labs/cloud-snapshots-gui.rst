=========================================
Lab 06 - Cloud Snapshots
=========================================

.. include:: import-yaml.rst

.. important:: We will make use of Minio object store in this Lab. We will use it as the endpoint for our cloud snapshots.

Deploy Minio as target for Portworx Cloud Snapshots
---------------------------------------------------

Create a storageClass for use by Minio

.. code-block:: yaml
  :name: px-ha-sc.yaml

  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-ha-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    io_priority: "high"
    group: "minio"

Copy the above code block and paste it into the Import YAML.   

Deploy Minio onto the OCP cluster from the jump host

.. code-block:: shell

  echo "Installing Helm"
  curl -L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash -s -- --version v3.8.2
  helm repo add stable https://charts.helm.sh/stable
  echo "Helm installed!"

.. code-block:: shell

  echo "Installing mc"
  export WORKER_NODE=$(oc get node -l node-role.kubernetes.io/worker= | grep -Eiv "infra|NAME" | awk '{print $1}' | head -1)
  oc debug node/$WORKER_NODE
  chroot /host
  curl --output /tmp/mc https://dl.minio.io/client/mc/release/linux-amd64/mc
  chmod +x /tmp/mc
  echo "mc installed!"

.. code-block:: shell

  echo "Setting Up Minio" 
  oc -n minio adm policy add-scc-to-user anyuid -z px-minio
  helm install px-minio stable/minio --namespace minio --create-namespace --set accessKey=ZZYYXXWWVVUUTT --setsecretKey=0ldSup3rS3cr3t --set persistence.storageClass=px-ha-sc --set resources.requests.memory=1Gi > /dev/null 2>&1
  until [[ `oc -n minio get pods | grep px-minio | grep Running | grep 1/1 | wc -l` -eq 2 ]]; do echo "Waiting for px-minio to be ready...."; sleep 1 ;done
  echo "Setup Complete ..."

.. code-block:: shell

  oc debug node/$WORKER_NODE
  chroot /host
  MINIO_ENDPOINT=http://$(oc --kubeconfig /var/lib/kubelet/kubeconfig -n minio get svc px-minio -o jsonpath='{.spec.clusterIP:9000')
  echo "Configure Minio's endpoints"
  /tmp/mc config host add px $MINIO_ENDPOINT ZZYYXXWWVVUUTT 0ldSup3rS3cr3t --api S3v4
  echo "Configuration Complete"

Once the set-up is complete, you can run the following command to make sure the minio servers are up:

.. code-block:: shell

  oc debug node/$WORKER_NODE
  chroot /host 
  /tmp/mc admin info px
  
To check for the backed up objects in the object store:

.. code-block:: shell

  oc debug node/$WORKER_NODE
  chroot /host 
  /tmp/mc ls px/

Create a new Portworx credential called ``my-cloud-credentials`` with the below parameters:

.. code-block:: 

  provider = s3
  s3 region = us-east-1
  access key = ZZYYXXWWVVUUTT
  secret key = 0ldSup3rS3cr3t

Run the below command to obtain the object store endpoint:

.. code-block:: shell

  MINIO_ENDPOINT=http://$(oc -n minio get svc px-minio -o jsonpath='{.spec.clusterIP}:9000'); echo $MINIO_ENDPOINT

.. dropdown:: Show Solution

  Get the minio endpoint from the ``px-minio`` service and use it to create portworx credential: 
  
  .. code-block:: shell

    oc debug node/$WORKER_NODE
    chroot /host 
    MINIO_ENDPOINT=http://$(oc --kubeconfig /var/lib/kubelet/kubeconfig -n minio get svc px-minio -o jsonpath='{.spec.clusterIP:9000')
    pxctl credentials create --providers3 --s3-access-key ZZYYXXWWVVUUTT --s3-secret-key 0ldSup3rS3cr3t --s3-endpoint $MINIO_ENDPOINT --s3-region us-east-1 my-cloud-credentials


Provision MySQL Database
------------------------

We will now create a MySQL database to use with Cloud Snapshots

.. code-block:: yaml
  :name: create-px-mysql-sc.yaml

  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-mysql-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    io_profile: "db"
    io_priority: "high"

.. code-block:: yaml
  :name: create-objects.yaml

  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: px-mysql-pvc
  spec:
    storageClassName: px-mysql-sc
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: mysql
  spec:
    selector:
      matchLabels:
        app: mysql
    replicas: 1
    template:
      metadata:
        labels:
          app: mysql
      spec:
        schedulerName: stork
        containers:
        - name: mysql
          image: mysql:5.6
          imagePullPolicy: "Always"
          env:
          - name: MYSQL_ALLOW_EMPTY_PASSWORD
            value: "1"
          ports:
          - containerPort: 3306
          volumeMounts:
          - mountPath: /var/lib/mysql
            name: mysql-data
        volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: px-mysql-pvc

Copy the above code blocks and paste it into the Import YAML.   

Create a demodb database

.. code-block:: shell
  
  Workloads -> Pods -> mysql pod -> Terminal

.. code-block:: shell

  mysql -u root -e "Create database demodb"

Take Cloud Snapshot
-------------------

We have deployed a mysql pod that uses PortWorx volume. Take a cloud snapshot of this PVC called ``mysql-snapshot``. The snapshot should be successfully backed up to the object store.

.. code-block:: yaml
  :name: cloud-snap.yaml

  apiVersion: volumesnapshot.external-storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    name: mysql-snapshot
    namespace: default
    annotations:
      portworx/snapshot-type: cloud
  spec:
    persistentVolumeClaimName: px-mysql-pvc

Copy the above code block and paste it into the Import YAML.   

If the cloud credentials and volume snapshot were set up correctly, you can check the status by checking the status of the volumeSnapshot:

.. code-block:: shell

  Home -> Search -> Resources 
  Search for VolumeSnapshots.volumesnapshot.external-storage.k8s.io/v1
  mysql-snapshot -> YAML

.. image:: images/mysql-snapshot-yaml.png 

To check for the backed up objects in the object store:

.. code-block:: shell

  oc debug node/$WORKER_NODE
  chroot /host 
  /tmp/mc ls px/

Clone PVC
---------

Create a clone PVC called ``px-mysql-clone-pvc`` by restoring data from the snapshot ``mysql-snapshot``.

.. code-block:: yaml
  :name: restore.yaml

  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: px-mysql-clone-pvc
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: mysql-snapshot
  spec:
    accessModes:
       - ReadWriteOnce
    storageClassName: stork-snapshot-sc
    resources:
      requests:
        storage: 1Gi

Copy the above code block and paste it into the Import YAML.

Verify the new persistentVolumeClaim becomes bound.
