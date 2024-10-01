<p align="center">
  <img src="assets/image-2.png" />
</p>


# Table of Content:

- [Introduction](#introduction-1)
- [Setup](#setup)
- [Migration](#migration)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)


# Introduction

### **What is the tool's purpose?**

The primary goal of `cattle-drive` tool is to migrate certain objects from source downstream RKE1 cluster to target RKE2 downstream cluster. These objects include:

   - Projects
        - Namespaces
        - ProjectRoleTemplateBindings
   - ClusterRoleTemplateBindings
   - Cluster Apps
   - Cluster Catalog Repos

### **How does the tool contribute to achieving its goals?**

The tool needs to interact with Rancher to obtain the resources of the source RKE1 downstream cluster such that it can migrate the required resources to the downstream RKE2 cluster. 

To communicate Rancher, you would need to download the `KUBECONFIG` file from the UI of the local cluster. If you are using the kubeconfig file of the upstream RKE1, RKE2 or k3s cluster, you would need to give it admin access by creating a `no-scoped` token using this [doc](https://ranchermanager.docs.rancher.com/api/quickstart) and update the kubeconfig file with the token. 

### **How to gain access to this tool?**

You can download this tool from the [releases](https://github.com/rancherlabs/cattle-drive/releases) page.

# Setup

### Pre-requisites:

   - Both the source and destination clusters must be running the same minor version of Kubernetes (e.g. migrating from an RKE1 cluster running `v1.26.9-rancher1-1` to an RKE2 cluster running `v1.26.14+rke2r1`).


### **Source Cluster:**

| RKE1 Version |  Cluster Name | 
| ------------ | ------------- | 
| `v1.28.13`    |  `source-rke` | 


![alt text](assets/image-8.png)


### **Destination Cluster:**

|   RKE2 Version  | Cluster Name  |  
| --------------- | ------------- |
| `v1.28.13+rke2r1`| `target-rke2` | 

![alt text](assets/image-9.png)

---

### **Bastion Host:**

This can be any host from where you would run `cattle-drive` tool to connect to the Rancher upstream cluster to perform migration. 

   - ### Download the `cattle-drive` tool

      ```bash
      stalin@bastion:~/cattle-drive$ curl -L  https://github.com/rancherlabs/cattle-drive/releases/download/v0.1.2/cattle-drive_Linux_x86_64.tar.gz | tar -zxvf -
        % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                       Dload  Upload   Total   Spent    Left  Speed
        0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
        0 12.7M    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0
      LICENSE
      README.md
      cattle-drive
      100 12.7M  100 12.7M    0     0  4327k      0  0:00:03  0:00:03 --:--:-- 8976k
      ```

   - ### `KUBECONFIG` file of Rancher.
      - Download the kubeconfig file from the Rancher UI of the local cluster. If you are using the upstream clusters kubeconfig file generated during the kubernetes cluster creation, then you would have to follow the procedure mentioned [here](#how-does-the-tool-contribute-to-achieving-its-goals).


     > Location of kubeconfig files in upstream clusters:

      - RKE1 - kubeconfig file named `kube_config_cluster.yml` is generated automatically in the same directory from where you ran the `rke up` command.
      - RKE2 - `/etc/rancher/rke2/rke2.yaml`.
      - K3S - `/etc/rancher/k3s/k3s.yaml`.

   - ### Test `cattle-drive` connectivity.
      - Once you have the **kubeconfig** file and **cattle-drive** tool ready, you can execute the below command to test if the tool is able to communicate with Rancher and fetch details about the status of the objects to migrate from source RKE1 cluster to target RKE2 cluster. 

      ```bash
      stalin@bastion:~/cattle-drive$ export KUBECONFIG=<Path_To_Kubeconfig_file>
      stalin@bastion:~/cattle-drive$ ./cattle-drive status -s source-rke -t target-rke2 --kubeconfig ~/cattle-drive/upstream-kubeconfig 
      Project status:
      - [demo-project] ✘
       -> namespaces:
              - [database] ✘
              - [demo-ns1] ✘
              - [demo-ns2] ✘
              - [demo-ns3] ✘
      Cluster users permissions:
      Catalog repos:
      ```
      - As you can see from the above output, the connection was successful and the tool returns a status displaying the resources that are marked as `✘` which denotes that they are yet to be migrated to the target cluster. 

# Migration

### Migrate Rancher objects: 
Let's try to migrate the Rancher objects from  `source-rke` cluster to `target-rke2` cluster. The tool contains two more commands:

- `migrate` - This command is used for migrating all the resources in one single shot. 

   ```bash
   stalin@bastion:~/cattle-drive$ ./cattle-drive migrate -s source-rke -t target-rke2 --kubeconfig ~/cattle-drive/upstream-kubeconfig 
   Migrating Objects from cluster [source-rke] to cluster [target-rke2]:
   - migrating Project [demo-project]... Done.
     - migrating Namespace [database]... Done.
     - migrating Namespace [demo-ns1]... Done.
     - migrating Namespace [demo-ns2]... Done.
     - migrating Namespace [demo-ns3]... Done.


   stalin@bastion:~/cattle-drive$ ./cattle-drive status -s source-rke -t target-rke2   --kubeconfig ~/cattle-drive/upstream-kubeconfig 
   Project status:
    - [demo-project] ✔
     -> namespaces:
            - [database] ✔
            - [demo-ns1] ✔
            - [demo-ns2] ✔
            - [demo-ns3] ✔
   Cluster users permissions:
   Catalog repos:
   ```

   - You can see that the `migrate` command does not give you a selective option to migrate resources. 
   - When re-running the `status` command you can see `✔` at the end of each resource which denotes that it was successfully migrated.
   
      
   

- `interactive` - This command is useful for selective migration.
   ```bash
   # You can see few namespaces are not migrated. We will migrate `demo-ns5` and `demo-ns7` to the target cluster using interactive method.

   stalin@bastion:~/cattle-drive$ ./cattle-drive status -s source-rke -t target-rke2 --kubeconfig ~/cattle-drive/upstream-kubeconfig 
   Project status:
    - [demo-project] ✔
     -> namespaces:
            - [database] ✔
            - [demo-ns1] ✔
            - [demo-ns2] ✔
            - [demo-ns3] ✔
            - [demo-ns4] ✘
            - [demo-ns5] ✘
            - [demo-ns7] ✘
            - [demo-ns8] ✘
   Cluster users permissions:
   Catalog repos:
   ```
   [![asciicast](https://asciinema.org/a/DcXn5zk23O2nzY2qNCim4KFM2.svg)](https://asciinema.org/a/DcXn5zk23O2nzY2qNCim4KFM2)

   - You can use the `m` key stroke within the interactive shell to migrate a specific resource of choice. 

# Troubleshooting

### Log Collection:
  - You can run the tool in interactive mode to collect logs by passing the `--log-file` option. 

  ```bash
  stalin@bastion:~/cattle-drive$ ./cattle-drive interactive --log-file cattle-drive.log -s source-rke -t target-rke2 --kubeconfig ~/cattle-drive/upstream-kubeconfig 

  stalin@bastion:~/cattle-drive$ ls -las cattle-drive.log 
4 -rw------- 1 stalin stalin 1379 Oct  1 07:46 cattle-drive.log

stalin@bastion:~/cattle-drive$ cat  cattle-drive.log 
[2024-09-25 20:28:51.389929773 +0000 UTC m=+18.715519570] successfully updated cluster [source-rke]
[2024-09-25 20:28:51.389964539 +0000 UTC m=+18.715554311] migrated object [namespace/web3 ✘]
[2024-09-26 07:15:15.831151411 +0000 UTC m=+30.502793972] successfully updated cluster [source-rke]
[2024-09-26 07:15:15.831467388 +0000 UTC m=+30.503109942] migrated object [namespace/web1 ✘]
[2024-09-26 07:18:20.18651487 +0000 UTC m=+42.080160755] successfully updated cluster [source-rke]
[2024-09-26 07:18:20.186552418 +0000 UTC m=+42.080198302] migrated object [namespace/web2 ✘]
[2024-09-30 21:06:17.482633087 +0000 UTC m=+28.400034167] successfully updated cluster [source-rke]
[2024-09-30 21:06:17.483143185 +0000 UTC m=+28.400544262] migrated object [namespace/demo-ns7 ✘]
[2024-09-30 21:25:12.724063838 +0000 UTC m=+16.037008211] successfully updated cluster [source-rke]
[2024-09-30 21:25:12.724387418 +0000 UTC m=+16.037331791] migrated object [namespace/demo-ns5 ✘]
[2024-09-30 21:25:25.65953553 +0000 UTC m=+28.972479911] successfully updated cluster [source-rke]
[2024-09-30 21:25:25.659552207 +0000 UTC m=+28.972496580] migrated object [namespace/demo-ns7 ✘]
[2024-10-01 07:46:33.136280954 +0000 UTC m=+14.643524373] successfully updated cluster [source-rke]
[2024-10-01 07:46:33.136314595 +0000 UTC m=+14.643558015] migrated object [namespace/demo-ns4 ✘]
  ```
  - Collecting the Rancher logs would also be helpful to understand any migration failures. 

### Connectivity issues:

- You might see the below error message when using the tool to connect to Rancher using the incorrect kubeconfig file.
```bash
stalin@bastion:~/rancher-devcluster$ ~/cattle-drive/cattle-drive status -s source-rke -t target-rke2 --kubeconfig kube_config_cluster.yml 
initiating source [source-rke] and target [target-rke2] clusters objects.. |E1001 07:55:53.807829   20104 memcache.go:265] couldn't get current server API group list: the server could not find the requested resource
E1001 07:55:53.810012   20104 memcache.go:265] couldn't get current server API group list: the server could not find the requested resource
exiting tool: the server could not find the requested resource
```
- This is because the used kubecconfig file does not have the required token to authenticate with Rancher. 
- To fix this, login to Rancher UI as `admin` user and download the kubeconfig file from the Rancher UI of the local cluster. This kubeconfig file would have a token which will be used by the cattle-drive tool.
**`OR`**
- If you are using the kubeconfig file which was automatically generated by the RKE1, RKE2 or K3S kubernetes cluster, you would need create an API key with no-scope token as describe [here](https://ranchermanager.docs.rancher.com/api/quickstart) and update your existing kubeconfig file and use it with the tool. 

# FAQ

- Can I use cattle-drive to migrate Rancher?
  - No. The purpose of cattle-drive tool is to only migrate objects from RKE1 to RKE2 downstream clusters. 

- Is it possible to only migrate a specific resource like projects, clusterruserpermissions etc?
  - Yes. Check the interactive option in [this](#migration) section. 


