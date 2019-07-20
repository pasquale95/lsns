# lsns

A plugin for `Volatility` developed by Convertini Pasquale for the **Forensics** course at EURECOM led by Prof. Balzarotti Davide.

## Table of content
- [Namespaces description](#namespaces-description)
- [Plugin description](#plugin-description)
  - [General usage](#general-usage)
  - [Option -n](#option--n---ns)
  - [Option -p](#option--p---pid)
  - [Option -t](#option--t---table)
    - [Option -i](#option--i---inode)
- [Example Setup](#setup)
  - [Needed tools](#needed-tools)
    - [Install vagrant](#install-vagrant)
    - [Install Virtualbox and vboxmanage](#install-virtualbox-and-vboxmanage)
  - [Prepare and generate the dump](#prepare-and-generate-the-dump)
  - [Generate the profile for volatility](#generate-the-profile-for-volatility)
    - [Notes on profile](#notes-on-profile)
  - [Test the dump file](#test-the-dump-file)
- [Troubleshooting](#troubleshooting)
- [Contribute](#contribute)
- [Appendix](#Appendix)
  - [Namespaces offset](#namespaces-offset)
  - [Inode numbers](#inode-numbers)

## Namespaces description
Namespace are a way to detach processes from a specific kernel layer assigning them to a new one. They are widely used to split resources between different groups of processes.

Currently Linux supports:

| Namespace | from Kernel | Description |
|:---------------:|:------------:|:--------:|
| UTS | 2.6.19 | domain and hostname |
| IPC | 2.6.19 | queues, semaphores, shared memory... |
| MNT | 2.6.19 | filesystems |
| PID | 2.6.19 | pid |
| NET | 2.4.24 | IP, routes, network devices... |
| USER | 3.8 | Uid, Gid |
| CGROUP | 4.4 | memory, cpu |

## Plugin description

The plugin analyzes the kernel memory to find and list all namespaces and relative subscribed processes.

The plugin works on **Linux**, both on **x86** and **x64** architectures.

### General usage

1. Git clone the [Volatility](https://github.com/volatilityfoundation/volatility) repository or [Download a Release](http://www.volatilityfoundation.org/#!releases/component_71401);
2. Git clone this repository;
3. Copy `lsns.py` under the path `./volatility/plugins/linux`.

```bash
$ export VOLATILITY_PATH=<path_for_volatility>
$ export LSNS_PATH=<path_for_lsns>
$ mkdir $VOLATILITY_PATH
$ git clone https://github.com/volatilityfoundation/volatility $VOLATILITY_PATH
$ git clone https://github.com/Pasquale95/lsns.git $LSNS_PATH
$ cp $LSNS_PATH/lsns.py $VOLATILITY_PATH/volatility/plugins/linux/
```

4. Then run the command:

```bash
$ cd $VOLATILITY_PATH
$ python vol.py --profile=<profile> -f <path-to-file> lsns
```

An example of output is:

```
Volatility Foundation Volatility Framework 2.6.1
NSPACE_Offset      NS         TYPE   NSPROC PID   COMMAND         ARGUMENTS
------------------ ---------- ------ ------ ----- --------------- ----------------------------------------------------------------------------------------------------
0xffffffff81efd580 4026531957 net       175     1 init            /sbin/init splash
0xffffffff81e9cb40 ---------- ipc       178     1 init            /sbin/init splash
0xffffffff81e63220 4026531835 cgroup    181     1 init            /sbin/init splash
0xffffffff81e50140 4026531836 pid       178     1 init            /sbin/init splash
0xffffffff81e49360 4026531837 user      178     1 init            /sbin/init splash
0xffffffff81e152e0 4026531838 uts       181     1 init            /sbin/init splash
0xffff88011b130c00 ---------- mnt       180     1 init            /sbin/init splash
0xffff88011abd7800 ---------- mnt         1    11 kdevtmpfs       [kdevtmpfs]
0xffff88011a7cd200 ---------- ipc         1  3194 WebExtensions   /usr/lib/firefox/firefox -contentproc -childID 3 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff88011a4b1a00 ---------- ipc         1  3241 Web Content     /usr/lib/firefox/firefox -contentproc -childID 4 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff88011817d600 ---------- ipc         1  3133 Web Content     /usr/lib/firefox/firefox -contentproc -childID 1 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff8800aa25c390 4026532253 user        1  3241 Web Content     /usr/lib/firefox/firefox -contentproc -childID 4 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff8800aa25c260 4026532306 user        1  3194 WebExtensions   /usr/lib/firefox/firefox -contentproc -childID 3 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff8800aa25c000 4026532198 user        1  3133 Web Content     /usr/lib/firefox/firefox -contentproc -childID 1 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff880089144980 4026532362 net         1  3241 Web Content     /usr/lib/firefox/firefox -contentproc -childID 4 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff880089143100 4026532309 net         1  3194 WebExtensions   /usr/lib/firefox/firefox -contentproc -childID 3 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff880089141880 4026532255 net         3  3314 oxide-renderer  /usr/lib/x86_64-linux-gnu/oxide-qt/oxide-renderer...u-amazon-default.oxide --disable-gpu-compositing
0xffff880089140000 4026532201 net         1  3133 Web Content     /usr/lib/firefox/firefox -contentproc -childID 1 ...a -appdir /usr/lib/firefox/browser 3051 true tab
0xffff8800890d8000 4026532252 pid         3  3314 oxide-renderer  /usr/lib/x86_64-linux-gnu/oxide-qt/oxide-renderer...u-amazon-default.oxide --disable-gpu-compositing
```

where:
- `NSPACE_Offset`: the offset where the namespace C struct is located;
- `NS`: inode number of the namespace;
- `TYPE`: namespace type;
- `NSPROC`: number of process registered in the namespace;
- `PID`: pid of the process which created the namespace;
- `COMMAND`: short command of the process which created the namespace;
- `ARGUMENTS`: long command version of the process which created the namespace.

**Note:**: the plugin requires un updated `module.c` to retrieve the maximum amount of information from the memory dump. Since the default one in Volatility is not up-to-date, you can copy `module.c` you find in this repository under the path `tools/linux/` after cloning the Volatility project.

```bash
$ cp $LSNS_PATH/module.c $VOLATILITY_PATH/tools/linux/
```

### Option -n (--ns)

Each namespace can contain more registered processes. To list them use the option **-n** as follows:

```bash
 $ python vol.py --profile=<profile> -f <path-to-file> lsns -n <nspace_offset>
```

The output shows, in _tree_ format, the list of processes belonging to that namespace. An example is:

```
python vol.py --profile=LinuxUbuntu14extx64 -f ~/vm.elf lsns -n ffff880089141880
Volatility Foundation Volatility Framework 2.6.1

NAMESPACE: 0xffff880089141880L (TYPE: net) (INODE: 4026532255)
PID    PPID   COMMAND
3314   3313   /usr/lib/x86_64-linux-gnu/oxide-qt/oxide-renderer --type=zygote --shared-memory-override-path=/dev/shm/ubuntu-amazon-default.oxide --disable-gpu-compositing
3317   3314   └─ /usr/lib/x86_64-linux-gnu/oxide-qt/oxide-renderer --type=zygote --shared-memory-override-path=/dev/shm/ubuntu-amazon-default.oxide --disable-gpu-compositing
3344   3317      └─ /usr/lib/x86_64-linux-gnu/oxide-qt/oxide-renderer --type=renderer --disable-accelerated-video-decode --enable-smooth-scrolling --enable-use-zoom-for-dsf=fals
```

Notice that the option requires the `nspace_offset` (**1st** column of `lsns` output) since the inode value is not always available in the profiles.

### Option -p (--pid)

If you want to have a look to the namespaces for a specific pid, then use the option **-p** as follows:

```bash
$ python vol.py --profile=<profile> -f <path-to-file> lsns -p <pid>
```

The output shows, in _tabular_ format, the list of all the namespaces of the process identified by that pid:

```
$ python vol.py --profile=LinuxUbuntu14extx64 -f ~/vm.elf lsns -p 1
Volatility Foundation Volatility Framework 2.6.1

PID: 1
NS_offset          NS         TYPE   NSPROC PID   COMMAND
------------------ ---------- ------ ------ ----- -------------------
0xffffffff81e152e0 4026531838 uts    181    1     /sbin/init splash
0xffffffff81e9cb40 ---------- ipc    178    1     /sbin/init splash
0xffff88011b130c00 ---------- mnt    180    1     /sbin/init splash
0xffffffff81e50140 4026531836 pid    178    1     /sbin/init splash
0xffffffff81efd580 4026531957 net    175    1     /sbin/init splash
0xffffffff81e63220 4026531835 cgroup 181    1     /sbin/init splash
0xffffffff81e49360 4026531837 user   178    1     /sbin/init splash
```

### Option -t (--table)

If you want to have a complete overview of the namespaces of all processes, then you can run the option **-t** as follows:

```bash
$ python vol.py --profile=<profile> -f <path-to-file> lsns -t
```

The output shows, in _tabular_ format, the list of all processes with all the offsets of the respective namespaces:

```
python vol.py --profile=LinuxUbuntu14extx64 -f ~/vm.elf lsns -t
Volatility Foundation Volatility Framework 2.6.1
PROCESS         PID   uts_ns             ipc_ns             mnt_ns             pid_ns             net_ns             cgroup_ns          user_ns
--------------- ----- ------------------ ------------------ ------------------ ------------------ ------------------ ------------------ ------------------
init                1 0xffffffff81e152e0 0xffffffff81e9cb40 0xffff88011b130c00 0xffffffff81e50140 0xffffffff81efd580 0xffffffff81e63220 0xffffffff81e49360
kthreadd            2 0xffffffff81e152e0 0xffffffff81e9cb40 0xffff88011b130c00 0xffffffff81e50140 0xffffffff81efd580 0xffffffff81e63220 0xffffffff81e49360
rcu_bh              8 0xffffffff81e152e0 0xffffffff81e9cb40 0xffff88011b130c00 0xffffffff81e50140 0xffffffff81efd580 0xffffffff81e63220 0xffffffff81e49360
migration/0         9 0xffffffff81e152e0 0xffffffff81e9cb40 0xffff88011b130c00 0xffffffff81e50140 0xffffffff81efd580 0xffffffff81e63220 0xffffffff81e49360
watchdog/0         10 0xffffffff81e152e0 0xffffffff81e9cb40 0xffff88011b130c00 0xffffffff81e50140 0xffffffff81efd580 0xffffffff81e63220 0xffffffff81e49360
kdevtmpfs          11 0xffffffff81e152e0 0xffffffff81e9cb40 0xffff88011abd7800 0xffffffff81e50140 0xffffffff81efd580 0xffffffff81e63220 0xffffffff81e49360
Web Content      3133 0xffffffff81e152e0 0xffff88011817d600 0xffff88011b130c00 0xffffffff81e50140 0xffff880089140000 0xffffffff81e63220 0xffff8800aa25c000
WebExtensions    3194 0xffffffff81e152e0 0xffff88011a7cd200 0xffff88011b130c00 0xffffffff81e50140 0xffff880089143100 0xffffffff81e63220 0xffff8800aa25c260
Web Content      3241 0xffffffff81e152e0 0xffff88011a4b1a00 0xffff88011b130c00 0xffffffff81e50140 0xffff880089144980 0xffffffff81e63220 0xffff8800aa25c390
unity-webapps-s  3267 0xffffffff81e152e0 0xffffffff81e9cb40 0xffff88011b130c00 0xffffffff81e50140 0xffffffff81efd580 0xffffffff81e63220 0xffffffff81e49360
webapp-containe  3284 0xffffffff81e152e0 0xffffffff81e9cb40 0xffff88011b130c00 0xffffffff81e50140 0xffffffff81efd580 0xffffffff81e63220 0xffffffff81e49360
...
```

#### Option -i (--inode)
If you want the tabular format of the option **-t** but with the inode numbers, then use the option **-i**:

```bash
$ python vol.py --profile=<profile> -f <path-to-file> lsns -ti
```

The output shows, in _tabular_ format, the list of all processes with all the inodes (if available) of the respective namespaces:

```
python vol.py --profile=LinuxUbuntu14extx64 -f ~/vm.elf lsns -ti
Volatility Foundation Volatility Framework 2.6.1
PROCESS         PID   uts_ns     ipc_ns     mnt_ns     pid_ns     net_ns     cgroup_ns  user_ns
--------------- ----- ---------- ---------- ---------- ---------- ---------- ---------- ----------
init                1 4026531838 ---------- ---------- 4026531836 4026531957 4026531835 4026531837
kthreadd            2 4026531838 ---------- ---------- 4026531836 4026531957 4026531835 4026531837
rcu_bh              8 4026531838 ---------- ---------- 4026531836 4026531957 4026531835 4026531837
migration/0         9 4026531838 ---------- ---------- 4026531836 4026531957 4026531835 4026531837
watchdog/0         10 4026531838 ---------- ---------- 4026531836 4026531957 4026531835 4026531837
kdevtmpfs          11 4026531838 ---------- ---------- 4026531836 4026531957 4026531835 4026531837
Web Content      3133 4026531838 ---------- ---------- 4026531836 4026532201 4026531835 4026532198
WebExtensions    3194 4026531838 ---------- ---------- 4026531836 4026532309 4026531835 4026532306
Web Content      3241 4026531838 ---------- ---------- 4026531836 4026532362 4026531835 4026532253
unity-webapps-s  3267 4026531838 ---------- ---------- 4026531836 4026531957 4026531835 4026531837
webapp-containe  3284 4026531838 ---------- ---------- 4026531836 4026531957 4026531835 4026531837
...
```

## Setup

To run properly this plugin on a dump file, we are going to use a VM and dump it to get an `.elf` file which we will use to perform our forensics analysis. To let `vol.py` plugins running over the dump file we need to setup a matching **profile**.

Since the profile changes according to OS version, distribution and kernel we are going to setup a common environment in order to prevent incompatibility issues.

### Needed tools

I am working on a machine with the following specifications:
- OS: **Ubuntu 18.04.2 LTS**;
- arch: **x64**;
- kernel **4.15.0-50-generic**.

The tools we are going to use in order to setup a common environment are the following:

- `vagrant`: tool to run a VM almost automatically by specifing just few important points (`VM_NAME`, `VAGRANT_BOX`...). Since the image is automatically downloaded by the tool from the cloud, in this way we ensure we are running the same identical image;
- `virtualbox`: the real VM. `vagrant` needs it to run the downloaded image. You can also specify other `providers`, but in our case we are going to use `virtualbox` since then it's easier retrieving a dump of the entire operating system;
- `vboxmanage`: this tool allows to get a complete dump of a VM by specifying its name.

#### Install vagrant

vagrant is officially available with Ubuntu's **APT** tool. Thus, just run:

```bash
$ sudo apt-get install vagrant
```

To check everything went fine, run:

```bash
$ vagrant --version
```

and see what is the version you get. In my case is **Vagrant 2.0.2**.

#### Install Virtualbox and vboxmanage

Even though many of you already own a `Virtualbox` release, `vagrant` can have some problems with its last version.
In my case version **6.0** of VirtualBox was not compatible with valgrand, which is the reason why I downgraded to version **5.2.26**.

You can install this version by using Ubuntu's package manager or by downloading directly from [Virtualbox website](https://www.virtualbox.org/wiki/Download_Old_Builds_5_2).

If you want to install it using the first option, make sure to have added Oracle's keys to the package manager. [Here](https://tecadmin.net/install-oracle-virtualbox-on-ubuntu/) a useful guide.

Once you've done that, just run:

```bash
$ sudo apt-get install virtualbox-5.2
```

and the binary should be installed and located under `/usr/bin/`.

To check that everything is working fine, run:

```bash
$ virtualbox --help
```

and you should get in the first row the version of the machine.

If everything worked fine, we need also to install the **Extension pack** for `Virtualbox` before running `vagrant`.

You can download it [here](https://www.virtualbox.org/wiki/Download_Old_Builds_5_2). Just double-check the version you download corresponds the `Virtualbox` one.

Once you have installed everything, you should have got for free also `vboxmanage`.

### Prepare and generate the dump

To generate the VM, we need to tell `vagrant` the proper configuration by using a **Vagrantfile**. Here the content of my file, but be sure you change the variables that need to be changed:

```ruby
# encoding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :
# Box / OS
VAGRANT_BOX = 'ubuntu/trusty64'
# Memorable name for your
VM_NAME = 'ubuntu-14'
# VM User — 'vagrant' by default
VM_USER = 'vagrant'
# Vagrant folder
VAGRANT_FOLDER = 'vagrant_VMs'
# Host folder to sync
HOST_PATH = '~' + '/' + VAGRANT_FOLDER + '/' + VM_NAME
# Where to sync to on Guest — 'vagrant' is the default user name
GUEST_PATH = '/home/' + VM_USER + '/' + VM_NAME
# # VM Port — uncomment this to use NAT instead of DHCP
# VM_PORT = 8080

Vagrant.configure(2) do |config|
  # Vagrant box from Hashicorp
  config.vm.box = VAGRANT_BOX
  # Actual machine name
  config.vm.hostname = VM_NAME
  # Set VM name in Virtualbox
  config.vm.provider "virtualbox" do |v|
    v.name = VM_NAME
    v.memory = 2048
  end

  #DHCP — comment this out if planning on using NAT instead
  config.vm.network "private_network", type: "dhcp"
  # # Port forwarding — uncomment this to use NAT instead of DHCP
  # config.vm.network "forwarded_port", guest: 80, host: VM_PORT

  # Sync folder
  config.vm.synced_folder HOST_PATH, GUEST_PATH
# Enable USB Controller on VirtualBox
   config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--usb", "on"]
    vb.customize ["modifyvm", :id, "--usbxhci", "on"]
    vb.customize ["usbfilter", "add", "0",
      "--target", :id,
      "--name", "ULT-Best Best USB Device [0100]",
      "--manufacturer", "ULT-Best",
      "--product", "Best USB Device",
      "--serialnumber", "12345882407D"]
   end
end
```

Then run:

```bash
$ mkdir -p ~/vagrant_VMs/ubuntu-14
$ mv Vagrantfile ~/vagrant_VMs/ubuntu-14
$ cd ~/vagrant_VMs/ubuntu-14
$ vagrant up
```

At this point `vagrant` will create the VM with **Ubuntu 14.04** installed by downloading the box `trusty64`, that is one of the most famous boxes for this tool.
In case it doesn't work, give a try to:

```bash
$ vagrant up --provider virtualbox
```

After this process, to access the machine run:

```bash
$ vagrant ssh
```

If everything works, we are now ready to generate the dump in `.elf` format. To get it run:

```bash
$ vbox manage debugvm ubuntu-14 dumpvcore --filename vm.elf
```

This will generate `vm.elf` in the current directory.

### Generate the profile for volatility

To understand and convert the needed symbols, Volatility needs a profile about the memory dump we want to analyze. Therefore here a brief guide to generate this profile correctly.
We are going to use the machine we just created in order to facilitate this process.

To generate the profile we must work **on the machine under testing**. The needed tools are already included in the volatility project, thus we need just to move them under the vagrant machine:

```bash
$ cp -r ~/volatility/tools/ ~/vagrant_VMs/ubuntu-14/tools
```

Once we have done that we can move to the vagrant machine and install the required packages and binaries:

```bash
$ cd ~/vagrant_VMs/ubuntu-14
$ vagrant up
$ vagrant ssh
```

At this point you should be connected to the Virtual machine. From now on the commands have to be run inside the virtual machine:

```bash
$ sudo apt-get install dwarfdump
$ sudo apt-get install build-essential
$ sudo apt-get install linux-headers-generic
$ cd ubuntu-14/tools/linux
$ sed -i 's/PWD/shell pwd/g' Makefile
$ sudo make
```

If you encountered errors please refer [here](#Troubleshooting), otherwise run:

```bash
$ cd ~/ubuntu-14
$ sudo zip Ubuntu14.zip ./tools/linux/module.dwarf /boot/System-img...
```

#### Notes on profile

Under the folder `tools/linux` you will find a modified version of `module.c`. I cannot guarantee for its perfection, but I have modified it adding the definitions for some structures that were not updated in the original version of the file (i.e. `module.c` pushed by **Volatility community**, which seems not supporting all structures in the linux kernel versions above v3.3.0).

If you find problems in generating the profile, just re-enable the standard Volatility `module.c` which I leave in the same folder for simplicity. In other words run:

```bash
$ mv module.c module_new.c
$ mv module_old.c module.c
$ sudo make
```

### Test the dump file

At this point we can test that everything works correctly by running a simple plugin of `vol.py` on the just created `.elf` file.

But before doing that, make sure you have the proper profile correctly copied under your `volatility` folder. If you followed the previous steps, then you should find the profile named `Ubuntu14.zip` under the folder `~/vagrant_VMs/ubuntu-14`. Then just copy it:

```bash
$ cp ~/vagrant_VMs/ubuntu-14/Ubuntu14.zip ~/volatility/volatility/plugins/overlays/linux/
```

If everything went fine, the following command should print the list of processes running on that machine at the time when the dump was generated:

```bash
$ python vol.py -f ~/vm.elf --profile=LinuxUbuntu14x64 linux_pslist
```

An example of expected output is:

```
Volatility Foundation Volatility Framework 2.6.1
Offset             Name                 Pid             PPid            Uid             Gid    DTB                Start Time
------------------ -------------------- --------------- --------------- --------------- ------ ------------------ ----------
0xffff88007c088000 init                 1               0               0               0      0x000000007b890000 0
0xffff88007c089800 kthreadd             2               0               0               0      ------------------ 0
0xffff88007c08b000 ksoftirqd/0          3               2               0               0      ------------------ 0
0xffff88007c08c800 kworker/0:0          4               2               0               0      ------------------ 0
0xffff88007c08e000 kworker/0:0H         5               2               0               0      ------------------ 0
0xffff88007c140000 kworker/u2:0         6               2               0               0      ------------------ 0
0xffff88007c141800 rcu_sched            7               2               0               0      ------------------ 0
0xffff88007c143000 rcuos/0              8               2               0               0      ------------------ 0
0xffff88007c144800 rcu_bh               9               2               0               0      ------------------ 0
...
```

## Troubleshooting

In some cases you can run into some errors due to the `dwarfdump` version installed automatically by the package manager. If you encountered the error:

```
dwarfdump ERROR:  dwarf_attrlist:  DW_DLE_UNKNOWN_FORM (242) Possibly corrupt DWARF data (242)
```

it is very likely the problem is related to the version of `dwarfdump` installed. To bypass the problem we can install the last version of the tool by compiling its last version, which requires `automake-1.15`:

```bash
$ cd /usr/local/bin/
$ wget https://ftp.gnu.org/gnu/automake/automake-1.15.tar.gz
$ tar xf automake-1.15.tar.gz
$ cd automake-1.15
$ ./configure --prefix=/usr/local
$ sudo make install
```

and `aclocal-1.14.1`

```bash
$ cd /usr/local/bin/
$ wget https://ftp.gnu.org/gnu/automake/automake-1.14.1.tar.gz
$ tar xf automake-1.14.1.tar.gz
$ cd automake-1.14.1
$ ./configure --prefix=/usr/local
$ sudo make install
```

At this point we can compile the last version of `dwarfdump`:

```bash
$ git clone https://github.com/tomhughes/libdwarf.git
$ sudo apt-get install libelf1 libelf-dev
$ cd libdwarf/
$ ./configure
$ sudo make install
$ cp dwarfdump/dwarfdump /usr/local/bin/
$ cp dwarfdump/dwarfdump.conf /usr/local/lib/
$ cp libdwarf/libdwarf.a /usr/local/lib/
```

Now try again to run `make` under `tools/linux` to get `module.dwarf`.

## Contribute

To contribute please fork the project and then make a pull request.

## Appendix

### Namespaces offset

This section is to state my work to let the plugin working.
The plugin tries to reconstruct the **namespace <-> process** hierarchy by looking at 2 structs named respectively `nsproxy` and `cred` referenced by `task_struct`, which is allocated by the kernel for each new process:

```
struct task_struct {
  ...
  const struct cred __rcu *cred;
  struct nsproxy *nsproxy;
  ...
};
```

By looking at the content of `nsproxy` we have a list of 5 (or 6 from kernel v4.4) out of 6 (or 7) namespaces that belong to that process:

```
struct nsproxy {
  atomic_t count;
  struct uts_namespace *uts_ns;
  struct ipc_namespace *ipc_ns;
  struct mnt_namespace *mnt_ns;
  struct pid_namespace *pid_ns_for_children;
  struct net *net_ns;
  struct cgroup_namespace *cgroup_ns; //from kernel v.4.4
};
```

The counter `atomic_t count` is an integer which indicates the number of processes that dereference the struct, but it does not give any idea about the number of processes in each of the namespaces.

The missing namespace (`user_namespace`) is in the `cred` struct:

```
struct cred{
  ...
  struct user_namespace *user_namespace;
  ...
}
```

The `nsproxy` struct is cloned every time a new process defines a new namespace (any of the namespaces in the struct), otherwise it inherits the same pointer of the parent process.

Each pointer then addresses a specific structure where more info about that namespace are stored. This structure is unique for each namespace, which means that by just looking at these pointers we can divide the processes according to their namespaces.

**N.B**: if you want to maximize the number of namespaces my suggestion is to run web browsers inside the machine before dumping. Web browsers (especially **Google Chrome**, **Mozilla Firefox** and **Chromium**) create a reasonable amount of namespaces to isolate their resources/sub-processes from the external. For this reason, the setup indicated [here](#prepare-and-generate-the-dump) is perfect for a quick test, but if you want to delve into namespaces, it is better to opt for an image with also the **GUI** installed.

### Inode numbers

If we want to look at the inode number, then this info is stored within the struct `ns_common`, referenced by the `namespace struct`:

```
struct ns_common {
	atomic_long_t stashed;
	const struct proc_ns_operations *ops;
	unsigned int inum;
};
```

The number we are looking for is stored as `inum`, which stands for `inode number`.

**N.B** :As you can see running the tool not always the program is able to retrieve the inode number of all the namespaces, which is the reason why the option **-n** still requires the address as input and not the inode number. The reason for that must be found in the _profile_ we use with Volatility. Indeed I've experimented how for some OS the profile does not have the declaration of some kernel structures (like **user_namespace**): the plugin is developed such that if a structure type is not declared inside the profile, then it just ignores it and goes on instead of exiting with an exception. But of course when the output is ready some values will be displayed as `----------` to indicate that it was not inside the profile.
