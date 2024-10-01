![alt text](assets/image-2.png)

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
| `v1.28.3`    |  `source-rke` | 


![alt text](assets/image-8.png)


### **Destination Cluster:**

|   RKE2 Version  | Cluster Name  |  
| --------------- | ------------- |
| `v1.28.3+rke2r1`| `target-rke2` | 

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
      - Download the kubeconfig file of the local cluster from the Rancher UI. If you are using the upstream clusters kubeconfig file, then you would have to follow the procedure mentioned [here](#how-does-the-tool-contribute-to-achieving-its-goals).

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

# FAQ


