# A PostgreSQL cluster in OpenShift with the CN (cloud native) Postgres Operator

## Intro

To easily deploy PostgreSQL you can use the postgres cnpg operator.
This will create a postgres database cluster in the namespace you apply it to. Below is a sample basic postgres cluster definition.
Since there is no replication it is not a production ready database. See the section on high availability - below - for production use. Since your quota is limited be smart when and where to 
use replication.

## Create a postgres cluster:
```
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-pgdb
spec:
  instances: 1
  enablePDB: false
  imageName: ghcr.io/cloudnative-pg/postgresql:18.1-standard-bullseye
  primaryUpdateStrategy: unsupervised
  storage:
    size: 1Gi
    storageClass: rook-ceph-block
  walStorage:
    size: 1Gi
    storageClass: rook-ceph-block
  resources:
    requests:
      cpu: 500m
      memory: 200Mi
    limits:
      cpu: 1
      memory: 500Mi  
  
  
```

### Notes on the yaml:
- instances: 1 #only one instance is created
- enablePDB: false # no pod disruption budget is created. This is not useful for a single instance database.
- kind: Cluster # the CRD for a CN Postgres Cluster 
- imageName: ghcr.io/cloudnative-pg/postgresql:13.6  # if you want to use a specific postgres image 
- storage: size: 1Gi # 1GB of storage is created
- storageClass: rook-ceph-block # block storage is typically used for databases
- resources: requests: cpu: 500m # 500m cpu is requested, the operator will keep this amount of cpu available
- resources: requests: memorry: 200Mi # 200Mi memory is requested, the operator will keep this amount of memory available
- limits: cpu or memory: 1 # the operator will never use more than 1 cpu or 500Mi of memory, this is useful the temporary scale in case of a spike in traffic
- primaryUpdateStrategy: unsupervised # the operator will not update the primary pod automatically, it will wait for you to change the image
- walStorage: # Postgres uses Write ahead logging to ensure data consistency and replication. This storage is used for this purpose.


### what you get

The cloundnative PG operator creates a postgres cluster in the namespace you apply it to.
It will also create what you need.

#### pods
* Postgres Pods, typically named like:
  my-db-1, my-db-2, etc. 
* Each Pod usually has:
  The main PostgreSQL container
  Sidecar(s) for:
    * Backup/archiving (barman/wal-g depending on config)
    * Monitoring / metrics (if enabled)
    * Possibly a log sidecar depending on version/config

You see only the Pods, but the operator manages orchestration, promotions, failover, etc.

----
**Backups**
Are not possible yet because we still need to fasilitate S3 storage for the backups. This will be added soon. Watch the announcements on Github.
----

#### PVCs

It will create the PVCs you requested. In this case:
```
oc get pvc -n <namespace>
NAME            STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
my-db-1         Bound    pvc-ba26b682......   1Gi        RWO            rook-ceph-block   <unset>                 23m
my-db-1-wal     Bound    pvc-95b3adb0......   1Gi        RWO            rook-ceph-block   <unset>                 23m
```

#### secrets

It creates 5 secrets in the namespace.
```
oc get secrets -n sam-sam2-test | grep my-db
my-db-app                  kubernetes.io/basic-auth   11     26m
my-db-ca                   Opaque                     2      26m
my-db-dockercfg-4zblf      kubernetes.io/dockercfg    1      26m
my-db-replication          kubernetes.io/tls          2      26m
my-db-server               kubernetes.io/tls          2      26m
```

1. my-db-app (The most important one for you)
   * **Type**: kubernetes.io/basic-auth
   * Purpose: This contains the credentials for your application user.
   * Usage: This is what your backend application (Java, Python, Node, etc.) uses to connect to the database. It contains the 
     * username
     * password
     * dbname
     * connection uri.
2. my-db-ca
   * **Type**: Opaque
   * Purpose: This is the Certificate Authority (CA) for your database cluster.
   * Usage: CNPG uses this to sign all the other certificates (server and replication). It ensures that all communication between the operator, the nodes, and your apps is trusted. If you want your app to connect via verified TLS, you would mount the ca.crt from this secret.
3. my-db-server
    * **Type**: kubernetes.io/tls
   * Purpose: The Server Certificate.
   * Usage: This is used by the PostgreSQL instance to enable SSL/TLS. When a client connects to the database, the database presents this certificate to prove its identity and encrypt the traffic.
4. my-db-replication
    * **Type**: kubernetes.io/tls
   * Purpose: The Streaming Replication Certificate.
   * Usage: PostgreSQL uses this for internal communication between the Primary and Replica nodes. It allows them to authenticate each other securely without needing a password, using certificate-based authentication instead.
5. my-db-dockercfg-qbjqh
    * **Type**: kubernetes.io/dockercfg
   * Purpose: Image Pull Secret.
   * Usage: This is actually an OpenShift-specific secret automatically linked to your ServiceAccount. It allows the Pods to pull the PostgreSQL container image from the registry. You generally don't need to touch this.

**NB**
You might notice there isn't a my-db-superuser secret in your list. 
By default, CNPG manages the postgres user internally and doesn't always export it to a secret unless you ask it to, 
because you are encouraged to use the app user for everything.

From your commandline (after you logged in using oc login) you can connect a bash terminal to the database. If you log in with
```
oc exec -it my-db-1 -- psql -U postgres
```
you are in the database as adminuser.

The operator will also create a couple of services for the database.
Common ones:

    my-db-rw
        Read-write service.
        Always points to the current primary instance.
        Apps that need writes connect here.

    my-db-ro
        Read-only service.
        Typically targets replicas.
        Good for reporting / read scaling.

    my-db-r
        General read service (can target both primary and replicas depending on config/version).

## links:
- https://cloudnative-pg.io/
- https://www.cncf.io/projects/cloudnativepg/