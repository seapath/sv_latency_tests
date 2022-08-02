
# Setup machines with ansible - RTE project

This repository contains different playbooks, used to automate the configuration of the machines used in this projetct.
The setup is detailled below.
To run a playbook, you can use the following command :
```
ansible-playbook -i inventories/rte.yaml playbooks/<your_playbook>
```

## Setup - inventory

### Machines
Three machines are used in this setup :
* A PTP master, used to synchronize the two other machines with the PTP protocol
  * This machine is preinstalled with a linux distribution
  * PTP is installed and enabled as a systemd service.
    When the PTP master is started, it serves its system time through the PTP protocol.
* A sender machine, used to send IEC61850 frames (Sampled Values)
  * This machine must be preinstalled with a linux distribution
  * PTP is running as a service (ptp4l running as a slave), and synchronizes the system clock (phc2sys)
  * RTE's OUISTITI docker container is installed
* A SEAPATH machine, used to host different VMs, which will receive the frames
  * This machine will be flashed and configured via ansible playbooks

### Fetch generated images
Images are generated on the build server `jean`.
Two playbooks allow to fetch Host and Guest images from this build server.

### SEAPATH machine flash

#### PXE Repository
In order to automate the flash, the machine which will run the SEAPATH distribution is booted via PXE.
The container repository of the Seapath project must be used.  
Gerrit repository : https://g1.sfl.team/admin/repos/rte/votp/containers,general  
Github repository : https://github.com/seapath/containers  
A DHCP server will set an IP that you need to provide in the rte.yaml inventory.
Then seapath-flash-pxe image will be loaded on the machine, and booted.

#### Flash
When the machine has booted via PXE, use the playbook `flash_disk.yaml` to flash the seapath-host image.

#### Host-setup
* Before rebooting into SEAPATH, you need to set a fixed IP. Run `setup_static_ip.yaml`, and `reboot.yaml`. `setup_static_ip` sets a fixed IP address on the interface that will be used by SSH.
* `reboot` waits until the machine has rebooted into SEAPATH.
* When the machine has booted into SEAPATH, OVS interfaces must be configured. Run `setup_network.yaml`
* The, PTP services must be installed on the host with `setup_host_ptp.yaml`. This playbook disables the usual NTP synchronization (done by systemd-timesyncd) and replaces it with PTP.

### VM Deployement and setup
* VMs can be deployed using templates provided in `templates/vm`. Different templates allow to test different configurations. The images used are Seapath debug images, which allow connection with the root user.
* Before being managed with ansible, VM's must be assigned a fixed IP. To do so, a receipe was added to the `seapath-guest-efi-dbg` image in the Yocto Project, adding a fixed IP to the interface `enp0s2`. On the first boot of a VM, a systemd service launches and adds the following ip address to the interface : `192.168.203.<last byte of the MAC address>`.
* PTP synchronization must be enabled with `setup_vm_ptp.yaml`. This playbooks loads the `kvm_ptp` module, which creates a virtual PTP clock under `/dev/ptp0` and starts phc2sys service to synchronize the systemclock with this paravirtualized PTP device.

### Deploy SVTools
The SV receiver is provided with a docker container (where the libiec61850 is pre-installed). Build it using the SVTool repository : https://github.com/vermeulenthi/svtools  
Before using deploy_svtools.yaml, a local registry containing the image must be created on your machine :
```
docker run -d -p 5000:5000 --name registry registry:2
docker tag svtools_dkimg:latest localhost:5000/svtools_dkimg:latest
docker push localhost:5000/svtools_dkimg:latest
```
(Following the docker documentation : https://docs.docker.com/registry/)
