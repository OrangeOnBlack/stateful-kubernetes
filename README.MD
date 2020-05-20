# Prerequisites
Create image file to be used for Gluster storage (10 GB)

``$ dd if=/dev/zero of=/home/user/image bs=1M count=10000``

Mount image as loop device

``$ sudo losetup /dev/loop8 /home/user/image``

Load kernel module necessary for Gluster

``$ sudo modprobe dm_thin_pool``

Install required packages (use default on prompts ENTER)

``$ sudo add-apt-repository ppa:gluster/glusterfs-5``

``$ sudo apt update``

``$ sudo apt-get install -y software-properties-common``

``$ sudo apt-get install -y glusterfs-client``


Automatically load kernel module and mount image on reboot

```
$ echo '[Unit]
Description=Create the loopback device for GlusterFS
DefaultDependencies=false
Before=local-fs.target
After=systemd-udev-settle.service
Requires=systemd-udev-settle.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c "modprobe dm_thin_pool && [ -b /dev/loop8 ] || losetup /dev/loop8 /home/user/image"

[Install]
WantedBy=local-fs.target
' | sudo tee /etc/systemd/system/loop_gluster.service
```

``$ sudo systemctl enable /etc/systemd/system/loop_gluster.service``

# Install Gluster using Heteki (Version 9.0)
``$ git clone https://github.com/heketi/heketi``

``$ cd heketi/extras/kubernetes``

``$ git checkout 891a9966``

``$ kubectl create -f glusterfs-daemonset.json``

Label Kubernetes nodes to be Gluster nodes

``$ kubectl label node <hostname> storagenode=glusterfs``

``$ kubectl create -f heketi-service-account.json``

``$ kubectl create clusterrolebinding heketi-gluster-admin --clusterrole=edit --serviceaccount=default:heketi-service-account``

``$ kubectl create secret generic heketi-config-secret --from-file=./heketi.json``

``$ kubectl create -f heketi-bootstrap.json``


Install Heketi-CLI Tool

``$ wget https://github.com/heketi/heketi/releases/download/v9.0.0/heketi-client-v9.0.0.linux.amd64.tar.gz``

``$ tar -xzvf ./heketi-client-v9.0.0.linux.amd64.tar.gz``

``$ sudo cp ./heketi-client/bin/heketi-cli /usr/local/bin/``

``$ rm -rf heketi-client-v9.0.0.linux.amd64.tar.gz``


Set user, password and endpoint for ``hetekei-cli``

``$ export HEKETI_CLI_SERVER=http://$(kubectl get endpoints heketi | awk '{print $2}' | sed -n 2p)``

``$ curl $HEKETI_CLI_SERVER/hello``

``$ export HEKETI_CLI_USER=admin``

``$ export HEKETI_CLI_KEY="My Secret"``

**user and password can be changed in ``heketi/extras/kubernetes/heketi.json``**

In ``topology.json`` edit hostnames and IP addresses accordingly

Load topology file

``$ heketi-cli topology load --json=topology.json``

Initialise storage

``$ kubectl create -f heketi-storage.json``

``$ kubectl delete all,service,jobs,deployment,secret --selector="deploy-heketi"``

Start to permanent heketi deployment

``$ cd heketi/extras/kubernetes``

``$ kubectl create -f heketi-deployment.json``

Update endpoint

``$ export HEKETI_CLI_SERVER=http://$(kubectl get endpoints deploy-heketi | awk '{print $2}' | sed -n 2p)``

In ``storage-class`.yaml`` edit ``Resturl`` to match the endpoint, which can be obtained with the following command

``$ kubectl get endpoints deploy-heketi | awk '{print $2}' | sed -n 2p``

Create Kubernetes Storage Class

``$ kubectl -f storage-class.yaml``

Create Kubernetes Persistent Volume Claim

``$ kubectl -f persistent-volume-claim.yaml``

Optional: Launch example application using Gluster Volume

``$ kubectl -f example-nginx.yaml``



# Reset 

Delete metadata (repeat on every node)

``$ sudo rm -rf /var/lib/glusterd/vols/heketidbstorage``

``$ sudo rm -rf /var/lib/glusterd/peers/*``

``$ sudo glusterd restart``

Additionally zero-fill image files again an re-mount with ``losetup``
