===================================
Lab 04 - Shared Volumes
===================================

.. include:: import-yaml.rst

In this step, we will create a Portworx volume (PVC) for nginx.

Create StorageClass
-------------------------

Take a look at the StorageClass definition for Portworx

.. code-block:: yaml
  :name: px-shared-sc.yaml
  
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
      name: px-shared-sc
  provisioner: pxd.portworx.com
  parameters:
     repl: "3"
     sharedv4: "true"

The parameters are declarative policies for your storage volume. See `here <https://docs.portworx.com/manage/volumes.html>`__ for a full list of supported parameters. In our case the key parameter is sharedv4 = true.

Create the storage class using:

Copy the above code block and paste it into the Import YAML.   

Create PersistentVolumeClaim
----------------------------------

Take a look at the Persistent Volume Claim

.. code-block:: yaml
  :name: px-shared-pvc.yaml

  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
     name: px-shared-pvc
  spec:
    storageClassName: px-shared-sc
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 1Gi

Here we're pointing at the storage class defined above and giving our volume a maximum size (Portworx thinly provisions volumes so that space will not be reserved up-front).

Create the PersistentVolumeClaim using:

Copy the above code block and paste it into the Import YAML.   

Now that we have the volumes created, let's deploy a few nginx instances and see how the shared volumes work!

In this step, we will deploy the nginx application using the ``PersistentVolumeClaim`` created before.

.. note:: Notice in the below specification we have set the securityContext.seLinuxOptions. Without this setting the pods may be assigned random selinux labels, in which case only the last pod to come online would have access to the shared volume.   

Step deploy 3 instances of nginx
--------------------------------

.. code-block:: yaml

  :name: deploy-webapps.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: webapp1
    labels:
      app: webapp1
  spec:
    selector:
      matchLabels:
        app: webapp1
    replicas: 1
    template:
      metadata:
        labels:
          app: webapp1
          group: webapp
      spec:
        securityContext:
          runAsNonRoot: true
          seLinuxOptions: 
            level: "s0:c1,c0"
          seccompProfile: 
            type: RuntimeDefault
        containers:
        - name: webapp1
          securityContext:
            allowPrivilegeEscalation: false
            seLinuxOptions: 
              level: "s0:c1,c0"
            capabilities:
              drop: ["ALL"]
          image: nginxinc/nginx-unprivileged
          ports:
          - containerPort: 8080
          volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: shared-data
        volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: px-shared-pvc
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: webapp2
    labels:
      app: webapp2
  spec:
    selector:
      matchLabels:
        app: webapp2
    replicas: 1
    template:
      metadata:
        labels:
          app: webapp2
          group: webapp
      spec:
        securityContext:
          runAsNonRoot: true
          seLinuxOptions: 
            level: "s0:c1,c0"
          seccompProfile: 
            type: RuntimeDefault
        containers:
        - name: webapp2
          securityContext:
            allowPrivilegeEscalation: false
            seLinuxOptions: 
              level: "s0:c1,c0"
            capabilities:
              drop: ["ALL"]
          image: nginxinc/nginx-unprivileged
          ports:
          - containerPort: 8080
          volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: shared-data
        volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: px-shared-pvc
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: webapp3
    labels:
      app: webapp3
  spec:
    selector:
      matchLabels:
        app: webapp3
    replicas: 1
    template:
      metadata:
        labels:
          app: webapp3
          group: webapp
      spec:
        securityContext:
          runAsNonRoot: true
          seLinuxOptions: 
            level: "s0:c1,c0"
          seccompProfile: 
            type: RuntimeDefault
        containers:
        - name: webapp3
          securityContext:
            allowPrivilegeEscalation: false
            seLinuxOptions: 
              level: "s0:c1,c0"
            capabilities:
              drop: ["ALL"]
          image: nginxinc/nginx-unprivileged
          ports:
          - containerPort: 8080
          volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: shared-data
        volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: px-shared-pvc
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: webapp1-svc
    labels:
      app: webapp1
  spec:
    ports:
    - port: 80
      targetPort: 8080
    selector:
      app: webapp1
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: webapp2-svc
    labels:
      app: webapp2
  spec:
    ports:
    - port: 80
      targetPort: 8080
    selector:
      app: webapp2
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: webapp3-svc
    labels:
      app: webapp3
  spec:
    ports:
    - port: 80
      targetPort: 8080
    selector:
      app: webapp3

Take a look at the yaml:

Observe the ``volumeMounts`` and ``volumes`` sections where we mount the PVC.

Copy the above code block and paste it into the Import YAML.   

Verify nginx pods are ready
---------------------------------

Wait till all three nginx pods are in ready state.

.. code-block:: shell

  Workloads -> Pods

When all three pods are in ``Running`` state then then hit ``ctrl-c`` to clear the screen.. Be patient, if it's staying in Pending state for a while it's because it has to fetch the docker image on each node.

In this step, we will use pxctl to inspect the volume

Inspect the Portworx volume
---------------------------------

Portworx ships with a `pxctl <https://docs.portworx.com/control/status.html>`__ command line that can be used to manage Portworx.

Below we will use ``pxctl`` to inspect the underlying volume for our
PVC.

.. code-block:: text

    Workloads -> Pods -> 
    Select one of the pods named, portworx-cluster-XXXX
    Go to Terminal tab to review the volume status. Run: 

.. code-block:: shell

    /opt/pwx/bin/pxctl volume inspect <PVC-ID>

Make the following observations in the volume list \* ``Status`` indicates the volume is attached and shows the node on which it is attached. For shared volumes, this is the transaction coordinator node which all other nodes will go through to write the data. \* ``HA`` shows the number of configured replicas for this volume (shared volumes can be replicated of course, you can try it by modifying the storage class in step 2) \* ``Shared`` shows if the volume is shared \* ``IO Priority`` shows the relative priority of this volume's IO (high, medium, or low) \* ``Volume consumers`` shows which pods are accessing the volume

Now that we have our shared volumes created and mounted into all three nginx containers, let's proceed to write some data into the html folder of nginx and see how it gets read by all three containers.

In this step, we will check the state of our nginx servers.

Confirm our nginx servers are up
--------------------------------------

Connect to the Terminal on webapp1:

.. code-block:: shell

  Workloads -> Pods -> webapp1 -> Terminal

.. code-block:: shell

  curl webapp1-svc

You should see the following:

.. code:: html

   <html>
   <head><title>403 Forbidden</title></head>
   <body bgcolor="white">
   <center><h1>403 Forbidden</h1></center>
   <hr><center>nginx/xxx</center>
   </body>
   </html>

Create index.html nginx html folder on webapp1
----------------------------------------------------

Create index.html into webapp1's pod:

.. code-block:: shell

  Workloads -> Pods -> webapp1 -> Terminal

.. code-block:: shell

  cat <<EOF > /usr/share/nginx/html/index.html
   /$$$$$$$                       /$$                                                
  | $$__  $$                     | $$                                                
  | $$  \ $$ /$$$$$$   /$$$$$$  /$$$$$$   /$$  /$$  /$$  /$$$$$$   /$$$$$$  /$$   /$$
  | $$$$$$$//$$__  $$ /$$__  $$|_  $$_/  | $$ | $$ | $$ /$$__  $$ /$$__  $$|  $$ /$$/
  | $$____/| $$  \ $$| $$  \__/  | $$    | $$ | $$ | $$| $$  \ $$| $$  \__/ \  $$$$/ 
  | $$     | $$  | $$| $$        | $$ /$$| $$ | $$ | $$| $$  | $$| $$        >$$  $$ 
  | $$     |  $$$$$$/| $$        |  $$$$/|  $$$$$/$$$$/|  $$$$$$/| $$       /$$/\  $$
  |__/      \______/ |__/         \___/   \_____/\___/  \______/ |__/      |__/  \__/
  EOF

Now let's try all three URLs and see our hello world message is showing up on all three. This is because all three are attached to the same volume so updating one updates all three.

.. code-block:: shell

  Workloads -> Pods -> webapp1 -> Terminal

.. code-block:: shell

  curl webapp1-svc

.. code-block:: shell

  Workloads -> Pods -> webapp2 -> Terminal

.. code-block:: shell

  curl webapp1-svc

.. code-block:: shell

  Workloads -> Pods -> webapp3 -> Terminal

.. code-block:: shell

  curl webapp1-svc

In this step, we will play some file ping pong

Open some bash sessions in webapps 1-3
--------------------------------------------

Let's open a couple more terminals and have fun with shared volumes. You can navigate the terminals in the upper left corner of the screen:

Open a terminal for webapp1: *Terminal 1*.

.. code-block:: shell

  Workloads -> Pods -> webapp1 -> Terminal

.. code-block:: shell

  cd /usr/share/nginx/html/
  clear
  PS1="ping-pong-1# "
  echo "ping" > pingpong

Open a terminal for webapp2: *Terminal 2*.

.. code-block:: shell

  Workloads -> Pods -> webapp1 -> Terminal

.. code-block:: shell
  
  cd /usr/share/nginx/html/
  clear
  PS1="ping-pong-2# "
  echo "pong" > pingpong

Open a terminal for webapp3: *Terminal 3*.

.. code-block:: shell

  Workloads -> Pods -> webapp1 -> Terminal

.. code-block:: shell
  
  cd /usr/share/nginx/html/
  clear
  PS1="ping-pong-3# "
  echo "ping" > pingpong

Use the following command in *Terminal 3* to watch Ping - Pong Match between webapp1 and webapp2

.. code-block:: shell

  tail -f pingpong

*Terminal 1*: Start webapp1 as a pinger

.. code-block:: shell

  while sleep 2; do  echo "ping" >> pingpong; done

*Terminal 2*: Start webapp2 as a ponger

.. code-block:: shell

  while sleep 1; do  echo "pong" >> pingpong; done

You can have some more fun by using terminals 1,2,3 to see how they all share data in the mounted /usr/share/nginx/html folder.
