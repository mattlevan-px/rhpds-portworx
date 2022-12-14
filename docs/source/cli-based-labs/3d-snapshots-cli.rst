=========================================
Lab 07 - Application Consistent Snapshots
=========================================

3D Snapshots
------------

In the upcoming steps, we will set up 3D snapshots for the Portworx volumes consumed by these deployment and StatefulSets.

Deploy a MySQL Deployment and MongoDB statefulSet
-------------------------------------------------

We will deploy a couple of Databases. Once ready, inspect them.

.. code-block:: shell

  cat <<EOF > /tmp/create-mongo.yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: mongo  
    labels:
      name: mongo
  spec:
    ports:
    - port: 27017
      targetPort: 27017
    clusterIP: None  
    selector:
      role: mongo
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: mongo
  spec:
    serviceName: "mongo"
    selector:
      matchLabels:
        app: mongo
    replicas: 3
    template:
      metadata:
        labels:
          role: mongo
          environment: test
          app: mongo
      spec:
        terminationGracePeriodSeconds: 10
        containers:
          - name: mongo
            image: mongo:3.4
            command:
              - mongod
              - "--replSet"
              - rs0
              - "--bind_ip"
              - 0.0.0.0
              - "--smallfiles"
              - "--noprealloc"
            ports:
              - containerPort: 27017
            volumeMounts:
              - name: mongo-persistent-storage
                mountPath: /data/db
          - name: mongo-sidecar
            image: cvallance/mongo-k8s-sidecar
            env:
              - name: MONGO_SIDECAR_POD_LABELS
                value: "role=mongo,environment=test"
    volumeClaimTemplates:
    - metadata:
        name: mongo-persistent-storage
      spec:
        storageClassName: px-ha-sc
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: px-ha-sc
        resources:
          requests:
            storage: 2Gi
  EOF

.. code-block:: shell

  cat <<EOF > /tmp/create-mysql.yaml
  ---
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
      name: px-ha-sc
  provisioner: pxd.portworx.com
  parameters:
     repl: "3"
     io_profile: "db"
     io_priority: "high"
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
     name: px-mysql-pvc
  spec:
    storageClassName: px-ha-sc
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
  ---
  apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: null
    labels:
      app: mysql
    name: mysql
  spec:  
    ports:
    - port: 3306    
      protocol: TCP
      targetPort: 3306
    selector:
      app: mysql
  status:
    loadBalancer: {}
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
  EOF

.. code-block:: shell

  cat <<EOF > /tmp/testpolicy.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: SchedulePolicy
  metadata:
    name: testpolicy
    namespace: mysql-app
  policy:
    interval:
      intervalMinutes: 60
      retain: 5
    daily:
      time: "10:14PM"
      retain: 5
    weekly:
      day: "Thursday"
      time: "10:13PM"
      retain: 5
    monthly:
      date: 14
      time: "8:05PM"
      retain: 5
  EOF

.. code-block:: shell

  oc config set-context --current --namespace=default
  oc create -f /tmp/create-mysql.yaml
  sleep 5
  oc create -f /tmp/create-mongo.yaml
  oc create -f /tmp/testpolicy.yaml

Verify the creation of the MySQL and MongoDB pods are Ready
-----------------------------------------------------------

.. code-block:: shell

  oc get deployment

Wait until all MySQL nodes are ``Ready 1/1``

.. code-block:: shell

  oc get sts

Wait until all Mongo nodes are ``Ready 3/3``

Create a post-snapshot rule for MongoDB
---------------------------------------

Create a pre-snapshot rule called ``mysql-presnap-rule`` with the below specifications:

.. code-block:: shell

  cat <<EOF > /tmp/pre-mysql.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: Rule
  metadata:
    name: mysql-presnap-rule
  rules:
    - podSelector:
        app: mysql    
      actions:
      - type: command
        background: true
        # this command will flush tables with read lock
        value: mysql --user=root --password=$MYSQL_ROOT_PASSWORD -Bse 'flush tables with read lock;system ${WAIT_CMD};'
  EOF

Rules:

.. code-block:: 

  Pod Selector:app=mysql,
  type: command,
  background: true,
  value: mysql --user=root --password=$MYSQL_ROOT_PASSWORD
  -Bse 'flush tables with read lock;system ${WAIT_CMD};'

.. dropdown:: Show Solution
  
  We have created a solution file for you under ``/tmp/pre-mysql.yaml``.
  Run: 
  
  .. code-block:: shell
    
    oc apply -f /tmp/pre-mysql.yaml


Create an application consistent snapshot of MySQL
--------------------------------------------------

Create a new volume snapshot called ``mysql-3d-snapshot`` which makes use of the pre-snapshot rule ``mysql-presnap-rule'`` with PVC ``px-mysql-pvc``.

.. code-block:: shell

  cat <<EOF > /tmp/vs.yaml
  apiVersion: volumesnapshot.external-storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    name: mysql-3d-snapshot
    annotations:
      stork.rule/pre-snapshot: mysql-presnap-rule
  spec:
    persistentVolumeClaimName: px-mysql-pvc
  EOF

.. dropdown:: Show Solution
  
  We have created a solution file for you under ``/tmp/vs.yaml``
  Run:
  
 .. code-block:: shell
  
    oc apply -f /tmp/vs.yaml

Create a pre-snapshot rule for MongoDB
--------------------------------------

Create a pre-snapshot rule called ``mongodb-presnap-rule`` with the
below specifications:

.. code-block:: shell

  cat <<EOF > /tmp/pre-mongo.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: Rule
  metadata:
    name: mongodb-presnap-rule
  rules:
    - podSelector:      
        role: mongo
      actions:
      - type: command      
        value: mongo --eval "printjson(db.fsyncLock())"
  EOF

Rules:

.. code-block:: 

  Pod Selector:role=mongo
  type: command
  value: mongo --eval "printjson(db.fsyncLock())"

.. dropdown:: Show Solution
  
  We have created a solution file for you under ``/tmp/pre-mongo.yaml`` 
  Run:

  .. code-block:: shell

    oc apply -f /tmp/pre-mongo.yaml


Create a post-snapshot rule for MongoDB
---------------------------------------

Create a pre-snapshot rule called ``mongodb-postsnap-rule`` with the below specifications:

.. code-block:: shell

  cat <<EOF > /tmp/post-mongo.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: Rule
  metadata:
    name: mongodb-postsnap-rule
  rules:
    - podSelector:      
        role: mongo
      actions:
      - type: command      
        value: mongo --eval "printjson(db.fsyncUnLock())"
  EOF

Rules:

.. code-block:: 

  Pod Selector:role=mongo
  type: command
  value: mongo --eval "printjson(db.fsyncUnLock())"

.. dropdown:: Show Solution
  
  We have created a solution file for you under ``/tmp/post-mongo.yaml``.
  Run: 

  .. code-block:: shell
    
    oc apply -f /tmp/post-mongo.yaml

Create an application consistent snapshot of MongoDB
----------------------------------------------------

Create a new group volume snapshot called ``mongodb-3d-snapshot`` which makes use of the pre and snapshot rules ``mongodb-presnap-rule`` and ``mongodb-postsnap-rule``.

.. code-block:: shell

  cat <<EOF > /tmp/gvs.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: GroupVolumeSnapshot
  metadata:  
    name: mongodb-3d-snapshot
    annotations:
      stork.rule/pre-snapshot: mongodb-presnap-rule
      stork.rule/post-snapshot: mongodb-postsnap-rule
  spec:
    pvcSelector:
      matchLabels:
        app : mongo
  EOF

Spec:

.. code-block:: shell

  pvcSelector: role=mongo
  pre-snapshot rule: mongodb-presnap-rule
  post-snapshot rule: mongodb-postsnap-rule

.. dropdown:: Show Solution

      We have created a solution file for you under ``/tmp/gvs.yaml`` 
      Run: 

      .. code-block:: shell
        
        oc apply -f /tmp/gvs.yaml