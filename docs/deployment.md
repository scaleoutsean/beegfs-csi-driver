# BeeGFS CSI Driver Deployment

<a name="contents"></a>
## Contents

* [Deploying to Kubernetes](#deploying-to-kubernetes)
  * [Kubernetes Node Preparation](#kubernetes-node-preparation)
  * [Kubernetes Deployment](#kubernetes-deployment)
  * [Air-Gapped Kubernetes Deployment](#air-gapped-kubernetes-deployment)
  * [Deployment to Kubernetes Clusters with Mixed Nodes](#mixed-kubernetes-deployment)
  * [Deployment to Kubernetes Using the Operator](#operator-deployment)
* [Example Application Deployment](#example-application-deployment)
* [Managing BeeGFS Client Configuration](#managing-beegfs-client-configuration)
  * [General Configuration](#general-configuration)
  * [Kubernetes Configuration](#kubernetes-configuration)
  * [BeeGFS Client Parameters](#beegfs-client-parameters)
* [Notes for Kubernetes Administrators](#kubernetes-administrator-notes)
  * [Security and Networking Considerations](#security-considerations)
* [Removing the Driver from Kubernetes](#removing-the-driver-from-kubernetes)

<a name="deploying-to-kubernetes"></a>
## Deploying to Kubernetes

<a name="kubernetes-node-preparation"></a>
### Kubernetes Node Preparation

The following MUST be completed on any Kubernetes node (master OR worker) that
runs a component of the driver:
* Download the [BeeGFS repository
  file](https://doc.beegfs.io/latest/advanced_topics/manual_installation.html)
  for your Linux distribution to the appropriate directory.
* Using the package manager for your Linux distribution (i.e. yum, apt, zypper)
  install the following packages: 
  * beegfs-client-dkms
  * beegfs-helperd
  * beegfs-utils  
* Start and enable beegfs-helperd using systemd: `systemctl start beegfs-helperd
  && systemctl enable beegfs-helperd`

IMPORTANT: By default the driver uses the beegfs-client.conf file at
*/etc/beegfs/beegfs-client.conf* for base configuration. Modifying the location
of this file is not currently supported without changing kustomization files.

<a name="kubernetes-deployment"></a>
### Kubernetes Deployment

Deployment manifests are provided in this repository under *deploy/k8s/* along with
a default BeeGFS Client ConfigMap. The driver is deployed using `kubectl apply
-k` (kustomize). For more detailed information on how the manifests are organized or 
version specific upgrade information, see the Kubernetes [deployment 
README](../deploy/k8s/README.md).

Steps:
* On a machine with kubectl and access to the Kubernetes cluster where you want
  to deploy the BeeGFS CSI driver clone this repository: `git clone
  https://github.com/NetApp/beegfs-csi-driver.git`.
* Create a new kustomize overlay (changes made to the default overlay will be 
  overwritten in subsequent driver versions): `cp -r deploy/k8s/overlays/default 
  deploy/k8s/overlays/my-overlay`.
* If you wish to modify the default BeeGFS client configuration fill in the
  empty ConfigMap at *deploy/k8s/overlays/my-overlay/csi-beegfs-config.yaml*.
  * An example ConfigMap is provided at
    *deploy/k8s/overlays/examples/csi-beegfs-config.yaml*. Please see the 
    section on [Managing BeeGFS Client
    Configuration](#managing-beegfs-client-configuration) for full details. 
* If you are using [BeeGFS Connection Based Authentication](https://doc.beegfs.io/latest/advanced_topics/authentication.html) 
  fill in the empty Secret config file at 
  *deploy/k8s/overlays/my-overlay/csi-beegfs-connauth.yaml*.
  * An example Secret config file is provided at 
    *deploy/k8s/overlays/examples/csi-beegfs-connauth.yaml*. Please see 
    the section on [ConnAuth Configuration](#connauth-configuration) for full 
    details. 
* Change to the BeeGFS CSI driver directory (`cd beegfs-csi-driver`) and run:
  `kubectl apply -k deploy/k8s/overlays/my-overlay`.
  * Note by default the beegfs-csi-driver image will be pulled from
    [DockerHub](https://hub.docker.com/r/netapp/beegfs-csi-driver). See [this
    section](#air-gapped-kubernetes-deployment) for guidance deploying in
    offline environments.
  * Note that some supported Kubernetes versions may require modified 
    deployment manifests. Modify the `bases` field in 
    *deploy/k8s/overlays/my-overlay* as necessary before deployment 
    (e.g. `../../versions/v1.18`).
* Verify all components are installed and operational: `kubectl get pods -n
  kube-system | grep csi-beegfs`.

Example command outputs: 

```bash
-> kubectl cluster-info
Kubernetes control plane is running at https://some.fqdn.or.ip:6443

-> kubectl apply -k deploy/k8s/overlays/my-overlay
serviceaccount/csi-beegfs-controller-sa created
clusterrole.rbac.authorization.k8s.io/csi-beegfs-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/csi-beegfs-provisioner-binding created
configmap/csi-beegfs-config-h5f2662b6c created
secret/csi-beegfs-connauth-9gkbdgchg9 created
statefulset.apps/csi-beegfs-controller created
daemonset.apps/csi-beegfs-node created
csidriver.storage.k8s.io/beegfs.csi.netapp.com created

-> kubectl get pods -n kube-system | grep csi-beegfs
csi-beegfs-controller-0                   2/2     Running   0          2m27s
csi-beegfs-node-2h6ff                     3/3     Running   0          2m27s
csi-beegfs-node-dkcr5                     3/3     Running   0          2m27s
csi-beegfs-node-ntcpc                     3/3     Running   0          2m27s
```
Note the `csi-beegfs-node-#####` pods are part of a DaemonSet, so the number you
see will correspond to the number of Kubernetes worker nodes in your
environment.

Next Steps:
* If you want a quick example on how to get started with the driver see the
  [Example Application Deployment](#example-application-deployment) section. 
* For a comprehensive introduction see the [BeeGFS CSI Driver Usage](usage.md)
  documentation.

<a name="air-gapped-kubernetes-deployment"></a>
### Air-Gapped Kubernetes Deployment

This section provides guidance on deploying the BeeGFS CSI driver in
environments where Kubernetes nodes do not have internet access. 

Deploying the CSI driver involves pulling multiple Docker images from various
Docker registries. You must either ensure all necessary images (see
*deploy/k8s/overlays/default* for a complete list) are 
available on all nodes, or ensure they can be pulled from some internal 
registry.

If your air-gapped environment does not have a DockerHub mirror, one option is
pulling the necessary images from a machine with access to the internet
(example: `docker pull netapp/beegfs-csi-driver`) then save them as tar files
with [docker save](https://docs.docker.com/engine/reference/commandline/save/)
so they can be copied to the air-gapped Kubernetes nodes and loaded with [docker
load](https://docs.docker.com/engine/reference/commandline/load/).

Once the images are available, modify *deploy/k8s/overlays/my-overlay* to point 
to them. Adjust the `images[].newTag` fields as necessary to ensure they either 
match images that exist on the Kubernetes nodes or reference your internal 
registry. Then follow the above commands for Kubernetes deployment.

<a name="mixed-kubernetes-deployment"></a>
### Deployment to Kubernetes Clusters With Mixed Nodes

In some Kubernetes clusters, not all nodes are capable of running the BeeGFS 
CSI driver (or it may not be desirable for all nodes to do so). For example:
* A cluster may be shared by multiple departments and one department may not 
  want the BeeGFS client (and its kernel modules) to be installed on a subset 
  of nodes.
* Some nodes in a cluster may be running an OS that is not supported for the
  BeeGFS client (e.g. a specialized Linux distribution or Red Hat CoreOS in an 
  OpenShift cluster).
* Some nodes in a cluster may be capable of running the BeeGFS client, but the
  user installing the driver does not have the permissions required to install 
  it.

It is possible to patch the driver's deployment manifest so that the BeeGFS CSI 
driver's controller and node services only run on a subset of nodes. Follow 
these steps:
1. Either identify a label shared by nodes you want to install the driver on or 
   add a label to said nodes. For example:
   * Most Kubernetes distributions include the `node-role.kubernetes.io/master` 
     label on all master nodes.
   * OpenShift clusters include the `node.openshift.io/os_id=rhcos` or 
     `node.openshift.io/os_id=rhel` labels to distinguish between Red Hat 
     CoreOS and Red Hat Enterprise Linux nodes.
   * You may want to add a label like `node-role.your.domain/beegfs` to denote 
     BeeGFS capable nodes.
1. Open */deploy/k8s/overlays/my-overlay/patches/node-affinity.yaml* for 
   editing (where "my-overlay") is the overlay you created in 
   [Kubernetes Deployment](#kubernetes-deployment).
1. Edit or uncomment and edit the nodeAffinity field associated with the 
   controller service, the node service, or both. See the [Kubernetes 
   documentation](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/) 
   for more information about nodeAffinity configurations.
1. Deploy the driver: `kubectl apply -k deploy/k8s/overlays/my-overlay`.

NOTE: When the driver is installed in this way, all workloads (e.g. Pods, 
StatefulSets, Deployments) that depend on BeeGFS MUST be deployed with the same 
nodeAffinity assigned to the driver node service. Provide your users with the 
labels or nodes they must run their workloads on.

<a name="operator-deployment"></a>
### Deployment to Kubernetes Using the Operator

An [operator](https://operatorframework.io/what/) can be used to deploy the 
BeeGFS CSI driver to a cluster and manage its configuration/state within that 
cluster. The operator is designed for integration with 
[Operator Lifecycle Manager (OLM)](https://olm.operatorframework.io/) and is 
primarily intended as an easy way for OpenShift administrators or 
administrators with clusters running OLM to install the driver directly from 
[Operator Hub](https://operatorhub.io/) and keep it updated. It is also 
possible, though not recommended, to install the operator and to use the 
operator to install the driver directly from this repository.

See the [BeeGFS CSI Driver Operator](../operator/README.md) documentation for 
details.

<a name="example-application-deployment"></a>
## Example Application Deployment

Verify that a BeeGFS file system is accessible from the Kubernetes nodes.
Minimally, verify that the BeeGFS servers are up and listening from a
workstation that can access them:

```bash
-> beegfs-check-servers  # -c /path/to/config/file/if/necessary
Management
==========
mgmtd-node [ID: 1]: reachable at mgmtd.some.fdqn.or.ip:8008 (protocol: TCP)

Metadata
==========
meta-node [ID: 1]: reachable at meta.some.fqdn.or.ip:8005 (protocol: TCP)

Storage
==========
storage-node [ID: 1]: reachable at storage.some.fdqn.or.ip:8003 (protocol: TCP)
```

Modify *examples/k8s/dyn/dyn-sc.yaml* such that `parameters`:
- `sysMgmtdHost` is set to an appropriate value (e.g. `mgmtd.some.fqdn.or.ip` in
  the output above)
- `volDirBasePath` contains a k8s cluster `name` that is unique to this k8s
  cluster among all k8s clusters accessing this BeeGFS file system.

From the project directory, deploy the application files found in the
*examples/k8s/dyn/* directory, including a Storage Class, a PVC, and a pod which
mounts a volume using the BeeGFS CSI driver:

```bash
-> kubectl apply -f examples/k8s/dyn
storageclass.storage.k8s.io/csi-beegfs-dyn-sc created
persistentvolumeclaim/csi-beegfs-dyn-pvc created
pod/csi-beegfs-dyn-app created
```

Validate that the components deployed successfully:

```shell
-> kubectl get sc csi-beegfs-dyn-sc
NAME                PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
csi-beegfs-dyn-sc   beegfs.csi.netapp.com   Delete          Immediate           false                  61s


-> kubectl get pvc csi-beegfs-dyn-pvc
NAME                 STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS        AGE
csi-beegfs-dyn-pvc   Bound    pvc-7621fab2   100Gi      RWX            csi-beegfs-dyn-sc   84s

-> kubectl get pod csi-beegfs-dyn-app
NAME                 READY   STATUS    RESTARTS   AGE
csi-beegfs-dyn-app   1/1     Running   0          93s
```

Finally, inspect the application pod `csi-beegfs-dyn-app` which mounts a BeeGFS
volume:

```bash
-> kubectl describe pod csi-beegfs-dyn-app
Name:         csi-beegfs-dyn-app
Namespace:    default
Priority:     0
Node:         node3/10.193.161.165
Start Time:   Tue, 05 Jan 2021 09:29:57 -0600
Labels:       <none>
Annotations:  cni.projectcalico.org/podIP: 10.233.92.101/32
              cni.projectcalico.org/podIPs: 10.233.92.101/32
Status:       Running
IP:           10.233.92.101
IPs:
  IP:  10.233.92.101
Containers:
  csi-beegfs-app:
    Container ID:  docker://21b4404e9791de597ac60043ebf7ddd24d3f684e4228874b81c4e307e017f862
    Image:         alpine:latest
    Image ID:      docker-pullable://alpine@sha256:3c7497bf0c7af93428242d6176e8f7905f2201d8fc5861f45be7a346b5f23436
    Port:          <none>
    Host Port:     <none>
    Command:
      ash
      -c
      touch "/mnt/dyn/touched-by-${POD_UUID}" && sleep 7d
    State:          Running
      Started:      Tue, 05 Jan 2021 09:30:00 -0600
    Ready:          True
    Restart Count:  0
    Environment:
      POD_UUID:   (v1:metadata.uid)
    Mounts:
      /mnt/dyn from csi-beegfs-dyn-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-bsr5d (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  csi-beegfs-dyn-volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  csi-beegfs-dyn-pvc
    ReadOnly:   false
  default-token-bsr5d:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-bsr5d
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  20s   default-scheduler  Successfully assigned default/csi-beegfs-dyn-app to node3
  Normal  Pulling    18s   kubelet            Pulling image "alpine:latest"
  Normal  Pulled     17s   kubelet            Successfully pulled image "alpine:latest" in 711.793082ms
  Normal  Created    17s   kubelet            Created container csi-beegfs-dyn-app
  Normal  Started    17s   kubelet            Started container csi-beegfs-dyn-app
```

`csi-beegfs-dyn-app` is configured to create a file inside its own */mnt/dyn*
directory called *touched-by-<pod_uuid>*. Confirm that this file exists within
the pod:

```bash
-> kubectl exec csi-beegfs-dyn-app -- ls /mnt/dyn
touched-by-9154aace-65a3-495d-8266-38eb0b564ddd
``` 

Kubernetes stages the BeeGFS file system in the */var/lib/kubelet/plugins*
directory and publishes the specific directory associated with the persistent
volume to the pod in the */var/lib/kubelet/pods* directory. Verify that the
touched file exists in these locations if you have appropriate access to the
worker node:

```bash
-> ssh root@10.193.161.165
-> ls /var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-7621fab2/globalmount/mount/k8s/my-cluster-name/pvc-7621fab2/
touched-by-9154aace-65a3-495d-8266-38eb0b564ddd
-> ls /var/lib/kubelet/pods/9154aace-65a3-495d-8266-38eb0b564ddd/volumes/kubernetes.io~csi/pvc-7621fab2/mount/
touched-by-9154aace-65a3-495d-8266-38eb0b564ddd
```

Finally, verify that the file exists on the BeeGFS file system from a
workstation that can access it.

```bash
ls /mnt/beegfs/k8s/my-cluster-name/pvc-7621fab2/
touched-by-9154aace-65a3-495d-8266-38eb0b564ddd
```

Next Steps:
* For a comprehensive introduction see the [BeeGFS CSI Driver Usage](usage.md)
  documentation.
* For additional examples see *examples/k8s/README.md*. 

<a name="managing-beegfs-client-configuration"></a>
## Managing BeeGFS Client Configuration

Currently the only tested and supported container orchestrator (CO) for the
BeeGFS CSI driver is Kubernetes. Notes in the General Configuration section
below would apply to other COs if supported. For Kubernetes the preferred method
to apply desired BeeGFS Client configuration is using a Kubernetes ConfigMap and
Secret, as described in [Kubernetes Configuration](#kubernetes-configuration).

<a name="general-configuration"></a>
### General Configuration

The driver is ready to be used right out of the box, but many environments may
either require or benefit from additional configuration.

The driver loads a configuration file on startup which it uses as a template to
create the necessary configuration files to properly mount a BeeGFS file system.
A beegfs-client.conf file does NOT ship with the driver, so it applies the
values defined in its configuration file on top of the default
beegfs-client.conf that ships with each BeeGFS distribution. Each `config`
section may optionally contain parameters that override previous sections.

NOTE: The configuration file is specified by the `--config-path` command line 
argument. For Kubernetes, the deployment manifests handle this automatically.

Depending on the topology of your cluster, some nodes MAY need different
configuration than others. This requirement can be handled in one of two ways:
1. The administrator creates unique configuration files and deploys each to the
   proper node.
2. The administrator creates one global configuration file with a
   `nodeSpecificConfigs` section and specifies the `--node-id` CLI flag on each
   node when starting the driver.
     * Note: Deploying to Kubernetes using the provided manifests in deploy/k8s/
       handles setting this flag and deploying a ConfigMap representing this
       global configuration file.  

Kubernetes deployment greatly simplifies the distribution of a global
configuration file using the second approach and is the de-facto standard way to
deploy the driver. See [Kubernetes Configuration](#kubernetes-configuration) for
details.

In the example below, the `beegfsClientConf` section contains parameters taken
directly out of a beegfs-client.conf configuration file. In particular, the
beegfs-client.conf file contains a number of references to other files (e.g.
`connInterfacesFile`). The CSI configuration file instead expects a YAML list,
which it uses to generate the expected file. See [BeeGFS Client
Parameters](#beegfs-client-parameters) for more detail about
supported `beegfsClientConf` parameters.

NOTE: beegfs-client.conf values MUST be specified as strings, even if they 
appear to be integers or booleans (e.g. "8000", not 8000 and "true", not true).

The order of precedence for configuration option overrides is described by
"PRECEDENCE" comments in the example below. In general, precedence is as
follows: 

>config < fileSystemSpecificConfig < nodeSpecificConfig.config <
nodeSpecificConfig.fileSystemSpecificConfig 

When conflicts occur between configurations of equal precedence, configuration
set lower in the file takes precedence over configuration set higher in the
file.

NOTE: All configuration, and in particular `fileSystemSpecificConfigs` and
`nodeSpecificConfigs` configuration is OPTIONAL! In many situations, only the
outermost `config` is required.

```yaml
# when more specific configuration is not provided; PRECEDENCE 3 (lowest)
config:  # OPTIONAL
  connInterfaces:
    - <interface_name>  # e.g. ib0
    - <interface_name>
  connNetFilter:
    - <ip_subnet>  # e.g. 10.0.0.1/24
    - <ip_subnet>
  connTcpOnlyFilter:
    - <ip_subnet>  # e.g. 10.0.0.1/24
    - <ip_subnet>
  beegfsClientConf:
    <beegfs-client.conf_key>: <beegfs-client.conf_value>
    # All beegfs-client.conf values must be strings. Quotes are required on 
    # integers and booleans.
    # e.g. connMgmtdPortTCP: "9008"
    # SEE BELOW FOR RESTRICTIONS

fileSystemSpecificConfigs:  # OPTIONAL
    # for a specific filesystem; PRECEDENCE 2
  - sysMgmtdHost: <sysMgmtdHost>  # e.g. 10.10.10.1
    config:  # as above

    # for a specific filesystem; PRECEDENCE 2
  - sysMgmtdHost: <sysMgmtdHost>  # e.g. 10.10.10.100
    config:  # as above

nodeSpecificConfigs:  # OPTIONAL
  - nodeList:
      - <node_name>  # e.g. node1
      - <node_name>
    # default for a specific set of nodes; PRECEDENCE 1
    config:  # as above:
    # for a specific node AND filesystem; PRECEDENCE 0 (highest)
    fileSystemSpecificConfigs:  # as above

  - nodeList:
      - <node_name>  # e.g. node1
      - <node_name>
    # default for a specific set of nodes; PRECEDENCE 1
    config:  # as above:
    # for a specific node AND filesystem; PRECEDENCE 0 (highest)
    fileSystemSpecificConfigs:  # as above
```

<a name="connauth-configuration"></a>
#### ConnAuth Configuration

For security purposes, the contents of BeeGFS connAuthFiles are stored in a
separate file. This file is optional, and should only be used if the
connAuthFile configuration option is used on a file system's other services.

NOTE: The connAuth configuration file is specified by the `--connauth-path`
command line argument. For Kubernetes, the deployment manifests handle this
automatically.

```yaml
- sysMgmtdHost: <sysMgmtdHost>  # e.g. 10.10.10.1
  connAuth: <some_secret_value>
- sysMgmtdHost: <sysMgmtdHost>  # e.g. 10.10.10.100
  connAuth: <some_secret_value>
```

NOTE: Unlike general configuration, connAuth configuration is only applied at a 
per file system level. There is no default connAuth and the concept of a node 
specific connAuth doesn't make sense.

<a name="kubernetes-configuration"></a>
### Kubernetes Configuration

When deployed into Kubernetes, a single Kubernetes ConfigMap contains the
configuration for all Kubernetes nodes. When the driver starts up on a node, it
uses the node's name to filter the global ConfigMap down to a node-specific
version. 

The instructions in the [Kubernetes Deployment](#kubernetes-deployment)
automatically create an empty ConfigMap and an empty Secret and pass them to the
driver on all nodes.

* To pass custom configuration to the driver, add the desired parameters from
[General Configuration](#general-configuration) to
  *deploy/k8s/overlays/my-overlay/csi-beegfs-config.yaml* before deploying. 
  The resulting deployment will automatically include a correctly formed 
  ConfigMap. See *deploy/k8s/overlays/examples/csi-beegfs-config.yaml* for an 
  example file.
* To pass connAuth configuration to the driver, modify
  *deploy/k8s/overlays/my-overlay/csi-beegfs-connauth.yaml* before deploying. 
  The resulting deployment will automatically include a correctly formed
  Secret. See *deploy/k8s/overlays/examples/beegfs-config-connauth.yaml* for an 
  example file.

To update configuration after initial deployment, modify
*deploy/k8s/overlays/my-overlay/csi-beegfs-config.yaml* or 
*deploy/k8s/overlays/my-overlay/csi-beegfs-connauth.yaml* and repeat the 
kubectl deployment step from [Kubernetes Deployment](#kubernetes-deployment). 
Kustomize will automatically update all components and restart the driver on 
all nodes so that it picks up the latest changes.

NOTE: To validate the BeeGFS Client configuration file used for a specific PVC, 
see the [Troubleshooting Guide](troubleshooting.md#k8s-determining-the-beegfs-client-conf-for-a-pvc)

<a name="beegfs-client-parameters"></a>
### BeeGFS Client Parameters (beegfsClientConf)

The following beegfs-client.conf parameters appear in the BeeGFS v7.2
[beegfs-client.conf
file](https://git.beegfs.io/pub/v7/-/blob/7.2/client_module/build/dist/etc/beegfs-client.conf).
Other parameters may exist for newer or older BeeGFS versions. The list a
parameter falls under determines its level of support in the driver.

<a name="no-effect"></a>
#### No Effect

These parameters are specified elsewhere (a Kubernetes StorageClass, etc.) or
are determined dynamically and have no effect when specified in the
`beeGFSClientConf` configuration section.

* `sysMgmtdHost` (This is specified in a `fileSystemSpecificConfigs[i]` or by
  the volume definition itself.)
* `connClientPortUDP` (An ephemeral port, obtained by binding to port 0, allows
  multiple filesystem mounts. On Linux the selected ephemeral port is
  constrained by the values of [IP
  variables](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html#ip-variables).
  [Ensure that firewalls allow UDP
  traffic](https://doc.beegfs.io/latest/advanced_topics/network_tuning.html#firewalls-network-address-translation-nat)
  between BeeGFS file system nodes and ephemeral ports on BeeGFS CSI Driver
  nodes.)
* `connPortShift`

<a name="unsupported"></a>
#### Unsupported

These parameters are specified elsewhere and may exhibit undocumented behavior
if specified here.

* `connAuthFile` - Overridden in the 
  [connAuth configuration file](#connauth-configuration).
* `connInterfacesFile` - Overridden by lists in the 
  [driver configuration file](#general-configuration).
* `connNetFilterFile` - Overridden by lists in the driver configuration file.
* `connTcpOnlyFilterFile` - Overridden by lists in the driver configuration 
  file.

<a name="tested"></a>
### Tested

These parameters have been tested and verified to have the desired effect.

* `quotaEnabled` - Documented in [Quotas](quotas.md).

<a name="untested"></a>
#### Untested

These parameters SHOULD result in the desired effect but have not been tested.

* `connHelperdPortTCP`
* `connMgmtdPortTCP`
* `connMgmtdPortUDP`
* `connPortShift`
* `connCommRetrySecs`
* `connFallbackExpirationSecs`
* `connMaxInternodeNum`
* `connMaxConcurrentAttempts`
* `connUseRDMA`
* `connRDMABufNum`
* `connRDMABufSize`
* `connRDMATypeOfService`
* `logClientID`
* `logHelperdIP`
* `logLevel`
* `logType`
* `sysCreateHardlinksAsSymlinks`
* `sysMountSanityCheckMS`
* `sysSessionCheckOnClose`
* `sysSyncOnClose`
* `sysTargetOfflineTimeoutSecs`
* `sysUpdateTargetStatesSecs`
* `sysXAttrsEnabled`
* `tuneFileCacheType`
* `tunePreferredMetaFile`
* `tunePreferredStorageFile`
* `tuneRemoteFSync`
* `tuneUseGlobalAppendLocks`
* `tuneUseGlobalFileLocks`
* `sysACLsEnabled`

<a name="kubernetes-administrator-notes"></a>
## Notes for Kubernetes Administrators

<a name="security-considerations"></a>
### Security and Networking Considerations

**The driver must be allowed to mount and unmount file systems.**

* When the driver binary is run directly, it must be run as a user with
  permission to make the mount and unmount system calls and with permission to
  create and delete directories on the staging target path and target path.
* When the driver is run in its container, the container must be granted the
  CAP_SYS_ADMIN capability or the container must be privileged. The provided
  Kubernetes deployment manifests run the driver container as privileged to
  avoid SELinux and other concerns.

**The driver must be allowed to reserve arbitrary UDP ports in the primary
network namespace.**

For each volume to be mounted on a node, the driver identifies an
available UDP port. Mounting the volume causes the BeeGFS client to listen for
UDP traffic on this port. While the driver may be running in a container, the
BeeGFS client is not. The provided Kubernetes deployment manifests run the
driver container in the host network instead of an isolated container network
to allow the driver to correctly identify available ports.

**The network must allow BeeGFS traffic.**

[By default](https://doc.beegfs.io/latest/advanced_topics/network_tuning.html#firewalls-network-address-translation-nat),
outbound traffic from all BeeGFS clients to ports 8003, 8005, and 8008 on nodes
serving up BeeGFS file systems must be allowed. A BeeGFS file system can be
configured to use different ports and the driver [can be
configured](#general-configuration) to accomodate.

Inbound UDP traffic from nodes serving up BeeGFS file systems to arbitrary
ports on all BeeGFS clients must be allowed. Each volume requires its own port
and it is not currently possible to configure an allowed port range.

<a name="removing-the-driver-from-kubernetes"></a>
## Removing the Driver from Kubernetes

If you're experiencing any issues, find functionality lacking, or our
documentation is unclear, we'd appreciate if you let us know:
https://github.com/NetApp/beegfs-csi-driver/issues. 

The driver can be removed using `kubectl delete -k` (kustomize) and the original
deployment manifests. On a machine with kubectl and access to the Kubernetes
cluster you want to remove the BeeGFS CSI driver from: 

* Remove any Storage Classes, Persistent Volumes, and Persistent Volume Claims
  associated with the driver. 
* Set the working directory to the beegfs-csi-driver repository containing the
  manifests used to deploy the driver.
* Run the following command to remove the driver: `kubectl delete -k
  deploy/k8s/overlays/my-overlay`
