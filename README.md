**Q**. For each member in your team, provide 1 paragraph detailing what parts of the lab that member implemented / researched.

#### **_Purva's Contribution_**

Purva set up the GCP environment by resizing disk space, configuring VM resources, and installing kernel development dependencies. She cloned the Linux kernel repository, generated secure boot certificates, compiled the kernel, and documented the process. Purva also collaborated on analyzing kernel exits and monitoring VM operations.

#### **Preetam's Contribution**

Preetam modified the kernel source code, updated the \_\_vmx_handle_exit function, rebuilt and installed kernel modules, and configured GRUB. He managed kernel reboot, analyzed exit frequencies, documented findings, and navigated challenges in nested VM testing.

**Q**. Describe in detail the steps you used to complete the assignment

Prerequisite: Ensure GCP Disk Size greater than 20 GB

Steps to increase size :

Increase Disk Size and core for VM instance in compute engine

Stop the instance using:

`gcloud compute instances stop <INSTANCE_NAME> --zone <ZONE>`

Resize the disk to have sufficient space by running:

`gcloud compute disks resize DISK_NAME --size=SIZE_GB --zone=ZONE`

Upgrade the machine type to increase CPU cores using:

`gcloud compute instances set-machine-type <INSTANCE_NAME> --zone <ZONE> --machine-type n1-standard-2`

Restart instance by running:

`gcloud compute instances start <INSTANCE_NAME> --zone <ZONE>`

Step 1: Kernel Build Test

A. Fork the Linux GitHub Repository(https://github.com/torvalds/linux) into your own GitHub account.

B.  Clone the Repository in Your Outer VM. Then, SSH into your outer VM. Install Git using `sudo apt install git -y` and then, Clone your forked repository and run `cd linux`.

C. Configure the Kernel Build by firstly installing the necessary tools to build the Linux kernel using commands( `sudo apt update` and `sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev`)

D. Then, run `make menuconfig`. Install bc (Basic Calculator Tool): Run the following command to install bc using `sudo apt update` and `sudo apt install bc -y`.

E.  Build the Kernel by generating a dummy certificate:

`mkdir -p debian/certs`

Generate Certificate:

`openssl req -new -x509 -newkey rsa:2048 -sha256 -nodes -days 365 -keyout debian/certs/debian-uefi-certs.key -out debian/certs/debian-uefi-certs.pem -subj "/CN=Dummy Debian Secure Boot Certificate"`

Generate a Signing Key

Generate a private key and CSR for signing:

`openssl req -new -newkey rsa:2048 -days 3650 -nodes -keyout debian/certs/debian-uefi.key -out debian/certs/debian-uefi.csr`

Self-sign the certificate for 10 years:

`openssl x509 -req -days 3650 -in debian/certs/debian-uefi.csr -signkey debian/certs/debian-uefi.key -out debian/certs/debian-uefi-certs.pem`

Install lz4 using apt using `sudo apt update` and `sudo apt install lz4 -y` and build the kernel by running commands:

`make -j$(nproc)`

`sudo make modules_install`

`sudo make install`

Generate the Initial RAM Disk by running this command : `sudo update-initramfs -c -k $(make kernelrelease)` . The kernel files will be installed in /boot.

F. Before rebooting, take a snapshot of your outer VM and use your cloud platform or virtualization tool's snapshot feature.

G. Reboot the new kernel for testing by running commands:

`sudo update-grub`

`sudo reboot`

Verify the running kernel version using `uname -r` .

H. Ensure it matches the version of the kernel you built and necessary KVM modules are present by running `ls /lib/modules/$(uname -r)/kernel/arch/x86/kvm/` .

Step 2: Modify the KVM Code

A. : Locate the Exit Handler

The function that handles VM exits in the KVM source code is architecture-dependent.

Note : For AMD processors (SVM), the exit handler is typically svm_handle_exit

For Intel processors (VMX), the equivalent function is likely \_vmx_handle_exit, found in arch/x86/kvm/vmx.c.

Use grep to locate the function handle_exit or a similarly named function. This function processes VM exits.

B. Update the Exit Handler Logic

In the \_\_vmx_handle_exit function, insert your logic to:

Increment counters for the total exits and specific exit types.
Log statistics every 10,000 exits.

To make the logs more informative, map common exit reasons to human-readable names:

C. Rebuild and Install the Kernel

i. Clean up the build directory to ensure no stale files:

`make clean`

ii. Rebuild the Kernel:

`make -j$(nproc)`

iii. Install Kernel Modules:

`sudo make modules_install`

iv. Install the Kernel:

`sudo make install`

`sudo update-grub`

`sudo reboot`

Step 3 : Test the Changes

A.  Boot the Inner VM using your modified KVM code:

`sudo qemu-system-x86_64 -enable-kvm -hda focal-server-cloudimg-amd64.img -m 512 -nographic -netdev user,id=net0,hostfwd=tcp::2222-:22 -device e1000,netdev=net0`

B.  Generate VM Exits in the Inner VM:

Perform activities like:

○ High CPU usage: while true; do :; done

○ Disk I/O: dd if=/dev/zero of=testfile bs=1M count=100 rm testfile

○ Network traffic: ping -c 100 google.com

C.  Check Kernel Logs: On the outer VM, view the statistics in the kernel logs:

`cd /linux`

`sudo dmesg | grep "KVM Exit"`




**Q**. Comment on the frequency of exits – does the number of exits increase at a stable rate? Or are there more exits performed during certain VM operations? Approximately how many exits does a full VMboot entail?

Some types of exits are observed more frequently at certain stages of VM operation than at others, so certain types are observed more frequently than others. MSR WRITE operations produce high volumes of exits as well as periodic external interrupts output moderate amounts of exits. Configuration and violation related exits are initiated during urgency of the memory management processes.

The basic VM boot describes around one million total exits as a clear indication of the intricate relations between the virtual machine and its host hardware. These exits consist of, first, hundreds of thousands of MSR WRITE events, second, several thousands external interrupts, and third, a variety of other infrequent exit type contributions which indicate that virtualization is not a simple uniform concept.

This continuity of VM exits emblematises the complexity of system initiation and administration. various operation scenarios produce specific types of exits, starting from simple interrupt handling to infrequent conditions, such as unknown or halt exits. Such fluctuations explain why the management and even the simple tracking of virtual machine performance for different computational processes is very challenging.


**Q**. Of the exit types, which are the most frequent? Least?

MSR WRITE exits are the most common, ranging from hundreds of thousands during VM operations. Accounting for the exit of the firms from the industry these system configuration transitions predominate.

External interrupts comprise a moderate frequency category where several thousands of exits occur and reflect one of the less intense forms of interaction.

Most infrequent are UNKNOWN EXIT and HLT exits which seem to reflect systems status which seldom tends to occur under virtualization.
