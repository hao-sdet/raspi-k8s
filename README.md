# Getting Started
Sources: https://microk8s.io/docs

## Requirements
+ 2 or more Raspberry Pi (the Raspberry Pi 3B is recommended)
+ 2 or more SD Cards
+ 5V and 2.5A Power Supply for each Pi

## Setup
### Installing Ubuntu Server 20.04 LTS (64-bit server OS)
1. Insert the SD card then run 
```
$ diskutil list
```
2. Unmount the target disk 
```
$ diskutil unmountDisk /dev/diskN (where N is the target disk)
```
3. Copy the image
```
$ sudo dd bs=1m if=path/to/your/image.img of=/dev/rdiskN; sync (where N is the target disk)
```
4. Eject the SD card
```
$ sudo diskutil eject /dev/rdiskN (where N is the target disk)
```

### Configuring your Raspberry Pi
Goal: 
```
k8s-master
k8s-worker-01
k8s-worker-02
k8s-worker-03
```
1. Enable SSH
```
$ sudo apt update
$ sudo apt install openssh-server
$ sudo systemctl status ssh
```
2. Change the hostname
```
$ nano /etc/hostname
```
3. Edit the hosts file:
```
$ nano /etc/hosts
```
Replace the "127.0.0.1 localhost" line with the new hostname so: "172.0.0.1 k8s-master localhost" for example.
4. Update
```
$ apt update && apt upgrade
```
5. Reboot

### Installing MicroK8s
0. SSH into your Pi
```
$ ssh pi@192.168.1.X
```
1. Enable c-groups (to make kubelet work out of the box)
```
$ sudo nano /boot/firmware/cmdline.txt
```
2. Add the following options
```
cgroup_enable=memory cgroup_memory=1
```
Expected:
```
cgroup_enable=memory cgroup_memory=1 net.ifnames=0 dwc_otg.lpm_enable=0 console=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait
```
3. Reboot
4. Install MicroK8s snap
```
sudo snap install microk8s --classic
```
5. Repeat step 0-4 for the remaining Pis

## Usage
### MicroK8s Add-Ons
See: https://microk8s.io/docs/addons
```
$ microk8s enable dns 
$ microk8s enable storage 
$ microk8s enable metallb (You’ll be prompted for a range of IP addresses here, pick one on the same subnet as the Kubenetes nodes)
$ microk8s enable ingress 
$ microk8s enable dashboard
$ microk8s enable registry
```

### Adding/Removing a node
!You may need to enable your firewall to allow communication between nodes
```
sudo ufw allow in on cni0 && sudo ufw allow out on cni0
sudo ufw default allow routed
```
1. Generate a connection string for master node
```
$ sudo microk8s.add-node
```
2. Join master from other nodes
```
$ microk8s.join <master_ip>:<port>/<token>
Ex: $ microk8s.join 10.55.60.14:25000/JHpbBYMIevZSAMnmjMHmFwanrOYCWZLu
```
3. Verify if new node has been joined/added
! To this on master node !
```
$ microk8s.kubectl get node
```
4. Repeat step 2 for the remaining nodes you want to add to the cluster
5. Remove a node from master
```
$ sudo microk8s remove-node <node name>
```
6. Alternatively, to leave the cluster from any node
```
sudo microk8s.leave
```

### Deploying the Application into Kubernetes
! Master node ONLY!
1. Install Docker
```
$ apt-get install docker.io
$ sudo systemctl enable docker
$ sudo systemctl start docker
```
2. Create the Docker File for your application
! The Raspberry Pi is an ARM based processor architecture, therefore you can only run images that are compiled for ARM, you can't run x86/x64 architecture images.
```
$ touch DockerFile (with your application)
```
3. Create an Docker image from DockerFile
```
$ docker build -f Dockerfile -t my-application:local .
```
4. Export Docker image
```
$ docker save hello-python > hello-python.tar
```
5. Import the Docker image into Kuerbetes Repository
```
$ microk8s ctr image import hello-python.tar
```
6. Verify that the Docker image has successfully imported
```
$ microk8s ctr images ls
```
7. Create a deployment.yaml and service.yaml files
```
$ touch deployment.yaml
$ touch service.yaml 
```
8. Deploy your application
```
$ kubectl apply -f deployment.yaml
$ kubectl apply -f service.yaml
```
9. Check pods status
```
$ microk8s kubectl get pods
```
10. Show services
```
$ microk8s kubectl get services
```
11. Remove the Application (Deployment and Service)
```
$ microk8s kubectl delete -f deployment.yaml
$ microk8s kubectl delete -f service.yaml
```
12. Updating an Application or Service on the Fly
While the application is running you can modify the content inside the service.yaml file then run the following command to update the changes
```
$ microk8s kubectl apply -f service.yaml
```

### Working with Kubernetes Image Repository
!You may need to allow Insecure Registry for private secure registry
See: https://microk8s.io/docs/registry-private
1. Enable Kubernetes Image Repository
!Once enabled, the registry should be lived on localhost:32000
```
$ microk8s enable registry
```
2. Build the Docker image with tag
```
$ docker build . -t localhost:32000/my-application:registry
```
3. Push image to Kubernetes Image Repository
```
$ docker push localhost:32000/my-application
```
4. Deploy your container using Registry Image
Update deployment.yaml file 
```
image: 192.168.1.164:32000/my-application:registry
```
5. Save and deploy
```
$ kubectl apply -f deployment.yaml
```
