---
id: minio
title: OpenEBS for Minio
sidebar_label: Minio
---
------

<img src="/docs/assets/o-minio.png" alt="OpenEBS and Minio" style="width:400px;">

<br>

## Introduction

<br>

MinIO is a high performance distributed object storage server, designed for large-scale private cloud infrastructure. MinIO is designed in a cloud-native manner to scale sustainably in multi-tenant environments. Orchestration platforms like Kubernetes provide a perfect cloud-native environment to deploy and scale MinIO. MinIO can be provisioned with OpenEBS volumes using various OpenEBS storage engines such as Local PV, cStor or Jiva based on the application requirement.



**Advantages of using OpenEBS for this Object storage solution:**

- PVCs to Minio are dynamically provisioned out of a dedicated or shared storage pool. You don't need to have dedicated disks On-Premise or on cloud to launch Object storage solution
- Adding more storage to the Kubernetes cluster with OpenEBS is seamless and done along with adding a Kubernetes node. Auto scaling of Minio instances is easy to do On-Premise as you do it on cloud platforms.
- Storage within a node or pool instance can also be scaled up on demand. 
- Complete management of disks under Minio are managed by OpenEBS, leading to production grade deployment of object storage

<br>

<hr>


## Deployment model

<br>

<img src="/docs/assets/svg/minio-deployment.svg" alt="OpenEBS and Minio" style="width:100%;">

<br>

<hr>


## Configuration workflow

<br>

1. Identify the installation method of MinIO
2. Install openebs
3. Select OpenEBS storage engine 
4. Create a new StorageClass or use default StorageClass based on the storage engine
5. Launch MinIO application by using the corresponding StorageClass
   
<h3><a class="anchor" aria-hidden="true" id="install-openebs"></a>Identify the installation method of MinIO</h3>

MinIO can be installed in 2 approaches:
- Standalone mode
  This will install one single application which is a `Deployment` kind. In standalone mode, MinIO will not maintain the replication at the application level. So if a user need replication at storage level, then user need to identify the OpenEBS storage engine which supports storage level replication. 

- Distributed mode
  MinIO can do the replication by itself in distributed mode. This will install Minio application which is a `StatefulSet` kind. It requires a minimum of 4 Nodes to setup MinIO in distributed mode. A distributed MinIO setup with 'n' number of disks/storage will have your data safe as long as `n/2` or more disks/storage are online. User should maintain a minimum  `(n/2 + 1)` disks/storage to create new objects. So based on the requirement, user can choose the appropriate OpenEBS storage engine to run MinIO i distributed mode.

For more information on installation, see Minio [documentation](https://github.com/helm/charts/tree/master/stable/minio).


<h3><a class="anchor" aria-hidden="true" id="install-openebs"></a>Install OpenEBS</h3>

If OpenEBS is not installed in your K8s cluster, this can be done from [here](https://docs.openebs.io/docs/next/installation.html). If OpenEBS is already installed, go to the next step.

<h3><a class="anchor" aria-hidden="true" id="install-openebs"></a>Select OpenEBS storage engine </h3>

Choosing a storage engine depends completely on the application workload as well as it's current and future growth in capacity and/or performance. More details of the storage engines can be found [here](https://docs.openebs.io/docs/next/casengines.html#when-to-choose-which-cas-engine). You can refer to one of the storage engine sections below based on your selection.
- [Local PV](https://docs.openebs.io/docs/next/localpv.html)
- [cStor](https://docs.openebs.io/docs/next/cstor.html)
- [Jiva](https://docs.openebs.io/docs/next/jiva.html)

The following is a quick summary of choosing a storage engine for MinIO:

- If user need more performance and does not require replication at storage level, choose **Local PV**. If Local PV device based StorageClass has been used, then, only one MinIO application on the Node can be consumed. If it is Local PV hostpath based StorageClass, then other than MinIO, some more applciation can be provisioned on the hostpath.

- If user need storage level replication and application is not taking replication by itself, but more reqruiement on Day 2 Operations such that snapshot & clone, volume & storage pool capacity expansion, scale up / scale down of volume replicas, etc, then **cStor** can be chosen. If MinIO is running in distributed mode and requires medium performance but requires Day 2 operations, then cStor with Single replica of Volume can be chosen.

- If user is having small workload running on the cluster, uses very small capacity requriement and application is not taking replication by itself, then **Jiva** can be chosen.


<h3><a class="anchor" aria-hidden="true" id="install-openebs"></a>Create a new StorageClass or use default StorageClass</h3>

Based on the OpenEBS storage engine chosen, user have to create a Storage Class with required storage policies on it or they can use default StorageClass created as part of OpenEBS installation. For example, if selected OpenEBS storage engine is cStor and MinIO deploying in distributed mode, user can create a cStor storage pool on each of the 4 Nodes and then use the corresponding `StoragePoolClaim` name in the StorageClass with replicaCount as `1`. 

The steps for the creation of StorageClass can be found from below.

- [cStor](https://docs.openebs.io/docs/next/ugcstor.html#creating-cStor-storage-class)
- [Local PV hostpath](https://docs.openebs.io/docs/next/localpv.html#openebs-localpv-hostpath), [Local PV device](https://docs.openebs.io/docs/next/localpv.html#openebs-localpv-device), [Local PV with Customized hostpath](https://docs.openebs.io/docs/next/uglocalpv.html#using-storageclass)
- [Jiva](https://docs.openebs.io/docs/next/jivaguide.html#create-a-sc)


<h3><a class="anchor" aria-hidden="true" id="install-openebs"></a>Launch MinIO application by using the corresponding StorageClass</h3>

<h4><a class="anchor" aria-hidden="true" id="minio-localpv"></a>MinIO on Local PV</h4>

MinIo can be provisioned on OpenEBS Local PV in 2 ways; on hostpath and other on Device. Local PV can be chosen if high performance is a must requirement and if the application can handle the replication by itself in `distributed mode` and or application does not require replication at all in `standalone mode`.

**In Standalone mode:**

Local PV can be provisioned on 2 ways:

- Hostpath-based

  In this case, Local PV volume will be provisioned on the hostpath created on the Node where the application has been scheduled. A PV will be created with specified size inside `/var/openebs/local` directory on the same node when it uses default Storage Class  `openebs-hostpath`. 
  Customizing the basepath can also be done using the steps provided [here](https://docs.openebs.io/docs/next/uglocalpv.html#configure-hostpath). Using the customized basepath, user can be able to mount the device and use this mounted directory into the basepath field in the StorageClass.

  After creating the StorageClass with required information, MinIO application can be launched using Local PV hostpath by running below command:
  ```
  helm install --name=minio-test --set accessKey=minio,secretKey=minio123,persistence.storageClass=openebs-hostpath,service.type=NodePort,persistence.enabled=true stable/minio
  ```
  Or
  ```
  kubectl apply -f https://raw.githubusercontent.com/ranjithwingrider/solution-app/master/minio-standalone-localpv-hostpath-default.yaml
  ```

  This will create a MinIO application running with one replica of a PV with 10Gi on default Local PV hostpath `/var/openebs/local/` directory on the node where the application pod has scheduled, if basepath is not customized.

- LocalPV-Device-based 

  In this case, Local PV volume will be provisioned on the node where application has scheduled and any of the unclaimed and active blockdevices available on the same node . Local PV device will use the entire blockdevice for MinIO application. The blockdevice can be mounted or raw device on the node where your application is scheduled and this blockdevice cannot be used by another application. If you have limited blockdevices attached to some nodes, then users can use `nodeSelector` in the application YAML to provision application on particular node where the available blockdevice is present.

  MinIO application can be launched using Local PV device by running below command:
  ```
  helm install --name=minio-test --set accessKey=minio,secretKey=minio123,persistence.storageClass=openebs-device,service.type=NodePort,persistence.enabled=true stable/minio
  ``` 
  Or
  ```
  kubectl apply -f https://raw.githubusercontent.com/ranjithwingrider/solution-app/master/minio-standalone-localpv-device.yaml
  ```
  This will create a single MinIO application on a single disk which is attached to the same node where the application is scheduled.

<font size="5">Accessing MinIO</font>

In a web browser, MinIO application can be accessed using the below way:

https://<Node_external_ip>:<Node_port>

Note:
- Node external IP address can be obtained by running `kubectl get node -o wide`
- Node port can be obtained by running `kubectl get svc -l app=minio

**In Distributed mode:**

Local PV can be provisioned on 2 ways:

- Hostpath-based
  In this case, Local PV volume will be provisioned on the hostpath on the Node where the application is getting scheduled. A PV will be created with specified size inside `/var/openebs/local` directory on the all 4 nodes when it uses default Storage Class  `openebs-hostpath`. 
  Customizing the basepath can also be done using the steps provided [here](https://docs.openebs.io/docs/next/uglocalpv.html#configure-hostpath). Using the customized basepath, user can be able to mount the device and use this mounted directory into the basepath field in the StorageClass.
  MinIO application can be launched using hostpath based Local PV by running below command:
  ```
  helm install --name=minio-test --set mode=distributed,accessKey=minio,secretKey=minio123,persistence.storageClass=openebs-hostpath,service.type=NodePort,persistence.enabled=true stable/minio
  ```
  or
  ```
  kubectl apply -f https://github.com/ranjithwingrider/solution-app/blob/master/minio-distributed-localpv-hostpath-default-kubectl.yaml
  ```
  This will create a MinIO application with replica count 4 running with a single replica of PV with 10G for all the 4 application instances on default Local PV hostpath `/var/openebs/local/` directory on the nodes where application has been scheduled,if basepath is not customized.

- LocalPV-Device-based 
  In this case, Local PV volume will be provisioned on the node where application has scheduled and any of the unclaimed and active blockdevices available on the same node . Local PV device will use the entire blockdevice for MinIO application. The blockdevice can be mounted or raw device on the node where your application is scheduled and this blockdevice cannot be used by another application. If you have limited blockdevices attached to some nodes, then users can use `nodeSelector` in the application YAML to provision application on particular node where the available blockdevice is present. Since MinIO is in distributed mode, it requires a minimum of 4 Nodes and a single unclaimed external disk should be attached to each of these 4 nodes.
  
  MinIO application can be launched using device-based Local PV by running below command:

   ```
   helm install --name=minio-test --set mode=distributed,accessKey=myaccesskey,secretKey=mysecretkey,persistence.storageClass=openebs-device,service.type=NodePort,persistence.enabled=true stable/minio
  ```
  Or
  ```
  kubectl apply -f https://raw.githubusercontent.com/ranjithwingrider/solution-app/master/minio-distributed-localpv-device-default.yaml
  ```
  This will create a MinIO application running with one replica on each of the 4 Nodes. This means, one PVCs with 10Gi will be created on each of suitable blockdevice on these 4 nodes. 

<font size="5">Accessing MinIO</font>

In a web browser, MinIO application can be accessed using the below way:

https://<Node_external_ip>:<Node_port>

Note:
- Node external IP address can be obtained by running `kubectl get node -o wide`
- Node port can be obtained by running `kubectl get svc -l app=minio

  
<br>

<hr>

## See Also:

<br>

### [OpenEBS architecture](/docs/next/architecture.html)

### [OpenEBS use cases](/docs/next/usecases.html)

### [cStor pools overview](/docs/next/cstor.html#cstor-pools)



<br>

<hr>

<br>





<!-- Hotjar Tracking Code for https://docs.openebs.io -->

<script>
   (function(h,o,t,j,a,r){
       h.hj=h.hj||function(){(h.hj.q=h.hj.q||[]).push(arguments)};
       h._hjSettings={hjid:785693,hjsv:6};
       a=o.getElementsByTagName('head')[0];
       r=o.createElement('script');r.async=1;
       r.src=t+h._hjSettings.hjid+j+h._hjSettings.hjsv;
       a.appendChild(r);
   })(window,document,'https://static.hotjar.com/c/hotjar-','.js?sv=');
</script>


<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-92076314-12"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-92076314-12');
</script>
