# tink-workflow

This is the step by step guide on how to setup tink workflow use tinkerbell to provision baremetal server.

## Getting Started


### Prerequisites
1. Get two machine, one is provisioner which it could be VM, the other one if the bare metal server you'd like to be 
provisoned by tinkerbell, here we call it worker node.

2. You need setup the tinkerbell provision engine before working on the workflow.

Here are some quick steps.

```
$ wget https://raw.githubusercontent.com/tinkerbell/tink/master/setup.sh && chmod +x setup.sh
$ ./setup.sh
```

### Build workflow action docker images

```
docker build -t 192.168.1.1/ubuntu:base 00-base/
docker push 192.168.1.1/ubuntu:base
docker build -t 192.168.1.1/disk-wipe:v1 01-disk-wipe/ --build-arg REGISTRY=192.168.1.1
docker push 192.168.1.1/disk-wipe:v1
docker build -t 192.168.1.1/disk-partition:v101 02-disk-partition/ --build-arg REGISTRY=192.168.1.1
docker push 192.168.1.1/disk-partition:v1
docker build -t 192.168.1.1/install-root-fs:v101 03-install-root-fs/ --build-arg REGISTRY=192.168.1.1
docker push 192.168.1.1/install-root-fs:v1
docker build -t 192.168.1.1/install-grub:v1 04-install-grub/ --build-arg REGISTRY=192.168.1.1
docker push 192.168.1.1/install-grub:v101
docker build -t 192.168.1.1/cloud-init:v1 05-cloud-init/ --build-arg REGISTRY=192.168.1.1
docker push 192.168.1.1/cloud-init:v1
```

### Build provision images for worker
Build rootfs/kernel/modules/initrd images, please take packet image build as reference 
https://github.com/packethost/packet-images


```
root@tink-provisioner:/var/tinkerbell/nginx/misc/osie/current/ubuntu_18_04# ls -l *tar.gz
-rw-r--r-- 1 root root 688419310 May 15 19:50 image.tar.gz
-rw-r--r-- 1 root root  25406438 May 15 19:50 initrd.tar.gz
-rw-r--r-- 1 root root   7927566 May 15 19:50 kernel.tar.gz
-rw-r--r-- 1 root root  65400456 May 15 19:50 modules.tar.gz
```
After you have all the images, you need copy all of them to the OSIE directory
```
root@tink-provisioner:/var/tinkerbell/nginx/misc/osie/current/ubuntu_18_04# pwd
/var/tinkerbell/nginx/misc/osie/current/ubuntu_18_04
```

### Configure tink workflow

1. Download this repo to your provisioner
2. Modify the generate.sh file to create the hardware.json file for your baremetal server
```
root@tink-provisioner:~/tink-workflow# vim generate.sh
#!/bin/bash

export UUID=$(uuidgen|tr "[:upper:]" "[:lower:]")  #UUID will be generated by uuidgen
export MAC=b8:59:9f:e0:f6:8c     # Change this MAC address to your worker node PXE port mac address, it has to match.
cat hardware.json | envsubst  > hw1.json 

echo wrote hw1.json - $UUID
```
3. Login into tink-cli client and push hw1.json and ubuntu.tmpl into tink
You need copy both hw1.json and ubuntu.tmpl file to tink cli before you can push them into tink
3.1 Create hardware
```
root@tink-provisioner:~/tink-workflow# docker exec -it deploy_tink-cli_1 sh
# push the hardware information to tink database
/tmp # tink hardware push --file /tmp/hw1.json
```
3.2 Create workflow template
```
/tmp # tink template create -n 'ubuntu' -p /tmp/ubuntu.tmpl
```
3.3 Create workflow
```
/tmp# tink workflow create -t <template-uuid> -r '{"device_1": $ <MAC of your worker PXE port>}'
```

### Reboot worker
Reboot worker node will trigger workflow starts and you can monitoring the workflow events
```
/tmp #  tink workflow events  f588090f-e64b-47e9-b8d0-a3eed1dc5439
+--------------------------------------+-----------------+-----------------+----------------+---------------------------------+--------------------+
| WORKER ID                            | TASK NAME       | ACTION NAME     | EXECUTION TIME | MESSAGE                         |      ACTION STATUS |
+--------------------------------------+-----------------+-----------------+----------------+---------------------------------+--------------------+
| 90e16ddd-a4ce-4591-bb91-3ec1eddd0e2b | os-installation | disk-wipe       |              0 | Started execution               | ACTION_IN_PROGRESS |
| 90e16ddd-a4ce-4591-bb91-3ec1eddd0e2b | os-installation | disk-wipe       |              7 | Finished Execution Successfully |     ACTION_SUCCESS |
| 90e16ddd-a4ce-4591-bb91-3ec1eddd0e2b | os-installation | disk-partition  |              0 | Started execution               | ACTION_IN_PROGRESS |
| 90e16ddd-a4ce-4591-bb91-3ec1eddd0e2b | os-installation | disk-partition  |             12 | Finished Execution Successfully |     ACTION_SUCCESS |
| 90e16ddd-a4ce-4591-bb91-3ec1eddd0e2b | os-installation | install-root-fs |              0 | Started execution               | ACTION_IN_PROGRESS |
| 90e16ddd-a4ce-4591-bb91-3ec1eddd0e2b | os-installation | install-root-fs |              8 | Finished Execution Successfully |     ACTION_SUCCESS |
| 90e16ddd-a4ce-4591-bb91-3ec1eddd0e2b | os-installation | install-grub    |              0 | Started execution               | ACTION_IN_PROGRESS |
| 90e16ddd-a4ce-4591-bb91-3ec1eddd0e2b | os-installation | install-grub    |              5 | Finished Execution Successfully |     ACTION_SUCCESS |
+--------------------------------------+-----------------+-----------------+----------------+---------------------------------+--------------------+
```

## Authors

* **Xin Wang** - *Initial work* - [tink-workflow](https://github.com/wangxin311/tink-workflow)

See also the list of [contributors](https://github.com/wangxin311/tink-workflow/graphs/contributors) who participated in this project.

## License


## Acknowledgments

* Hat tip to anyone whose code was used
* Inspiration
* etc
