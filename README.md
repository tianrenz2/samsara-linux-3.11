# Samsara

The main goal of Samsara is to implement a software approach that can take full advantage of the latest hardware features in commodity processors to record and replay memory access interleaving efficiently without introducing any hardware modifications.

## What is Samsara?

Deterministic replay, which provides the ability to travel backward in time and reconstruct the past execution flow of a
multiprocessor system, has many prominent applications. Prior research in this area can be classified into two categories:
hardware-only schemes and software-only schemes. While hardware-only schemes deliver high performance, they require significant
modifications to the existing hardware. In contrast, software-only schemes work on commodity hardware, but suffer from excessive
performance overhead and huge logs. In this article, we present the design and implementation of a novel system, Samsara, which
uses the hardware-assisted virtualization (HAV) extensions to achieve efficient deterministic replay without requiring any hardware
modification. Unlike prior software schemes which trace every single memory access to record interleaving, Samsara leverages HAV
on commodity processors to track the read-set and write-set for implementing a chunk-based recording scheme in software. By doing
so, we avoid all memory access detections, which is a major source of overhead in prior works.


To read more about the Samsara,check out [the project page](http://zhenxiao.com/replay/).


## Tutorial and Results (Written in 2025)

### Compile the Samsara kernel
The Samsara kernel is based on 3.11 which is very old, so today's version of tools cannot successfully compile it. There are two ways:

#### Using container (recommended)
1. Get the ubuntu 12.04 image, however, the packages source links in the official image are broken, so you could directly use the pre-built image below:
```
docker run -v <your samsara linux directory>:/samsara -ti tianrenz2/tianrenz:ubuntu1204 bash
```

To build from the ubuntu 12.04 official image, in the container, execute the following:
```
sed -i -e 's/archive.ubuntu.com\|security.ubuntu.com/old-releases.ubuntu.com/g' /etc/apt/sources.list
apt-get update
apt-get install libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev autoconf llvm gcc bc
```

2. Inside the container, start building:
```
cd /samsara; cp samsara-config .config; make -j32
```

3. After finishing compiling, exit the container:
```
cd <samsara linux directory>; make modules_install; make install
```

4. Reboot the machine

So far tested this process on (cloudlab c220g1, c220g2)[https://docs.cloudlab.us/hardware.html] and kernel could successfully run. However, some newer machines, e.g., c6420, cannot sucessfully boot the 3.11 kernel with the error
```
[    8.961322] Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b
```

#### Build on phsyical machine
The lowest gcc available on ubuntu 18.04 is gcc-4.8, so make sure it's installed:
```
apt update;apt install libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf llvm gcc-4.8
```

Also, cherry-pick this commit `425be5679fd292a3c36cb1fe423086708a99f11a`(Stop relying on magic jmp behavior for early_idt_handlers) from upstream kernel, otherwise it cannot boot, because newer gcc version has changed some assembly generation behavior that older kernel is not compatible with.


### Compile the Samsara QEMU
It used QEMU 1.2.2, which is also very old, it requires glib-2.12 to build, which is not available in package, so we build it from source:
```
wget https://download.gnome.org/sources/glib/2.12/glib-2.12.0.tar.gz
tar -xzf glib-2.12.0.tar.gz;cd glib-2.12.0
export CC=/usr/bin/gcc-4.8;./configure;sed -i 's/CFLAGS = /CFLAGS = -fcommon /' Makefile;make -j20;make install
```

After installing, build the QEMU:
```
git clone https://github.com/PKUCloud/samsara-qemu-1.2.2.git

cd samsara-qemu-1.2.2
mkdir build; cd build
../configure --target-list=x86_64-softmmu
```

### Test
Machine (Cloudlab c220g1):
```
CPU Two Intel E5-2660 v3 10-core CPUs at 2.60 GHz (Haswell EP)
RAM 160GB ECC Memory (10x 16 GB DDR4 2133 MHz dual rank RDIMMs)
Disk One Intel DC S3500 480 GB 6G SATA SSDs
Disk Two 1.2 TB 10K RPM 6G SAS SFF HDDs
NIC Dual-port Intel X520 10Gb NIC (PCIe v3.0, 8 lanes) (both ports available for experiment use)
NIC Onboard Intel i350 1Gb
```

QEMU Command:
```
./x86_64-softmmu/qemu-system-x86_64 -enable-kvm -smp 1 -m 2G -cpu qemu64 -machine pc -no-hpet -kernel <kernel image> -append "root=/dev/sda rw init=/lib/systemd/systemd tsc=reliable console=ttyS0" -hda <rootfs disk> -monitor stdio -vnc :0
```

In the monitor console, Start record:
```
record on
```

Stop record:
```
record off
```

#### Problem 1: Cannot record with more than 2G memory
When configure 4G or more memory, the system could initially run normally, but as far as recording starts, the host KVM throws out the backtrace below and the guest cannot run anymore:
```
[  866.952307] BUG: unable to handle kernel paging request at ffff883409780158
[  866.960108] IP: [<ffffffffa03fc91d>] __rr_walk_spt+0x13d/0x3e0 [kvm]
[  866.967244] PGD 1fcb067 PUD 0
[  866.970678] Oops: 0002 [#1] SMP
[  866.974303] Modules linked in: nfsv3 nfs_acl nfs lockd fscache x86_pkg_temp_thermal coretemp kvm_intel kvm crc32_pclmul ghash_clmulni_intel aesni_intel aes_x86_64 lrw gf128mul joydev glue_helper ablk_helper gpio_ich cryptd shpchp lpc_ich binfmt_misc acpi_pad acpi_power_meter mac_hid ib_iser rdma_cm ib_addr iw_cm sunrpc ib_cm ib_sa ib_mad ib_core iscsi_tcp libiscsi_tcp libiscsi scsi_transport_iscsi ip_tables x_tables autofs4 ixgbe igb hid_generic usbhid dca ptp hid mxm_wmi mpt3sas pps_core ahci raid_class libahci scsi_transport_sas mdio i2c_algo_bit wmi
[  867.029616] CPU: 3 PID: 1661 Comm: qemu-system-x86 Not tainted 3.11.0+ #2
[  867.037188] Hardware name: Cisco Systems Inc UCSC-C220-M4S/UCSC-C220-M4S, BIOS C220M4.4.1.2c.0.0202211901 02/02/2021
[  867.048940] task: ffff8827f5639770 ti: ffff88280c948000 task.ti: ffff88280c948000
[  867.057289] RIP: 0010:[<ffffffffa03fc91d>]  [<ffffffffa03fc91d>] __rr_walk_spt+0x13d/0x3e0 [kvm]
[  867.067132] RSP: 0018:ffff88280c949b18  EFLAGS: 00010287
[  867.073065] RAX: 000000000000002b RBX: ffff8813fd854728 RCX: 00000000001000e5
[  867.081037] RDX: ffff883409780000 RSI: 000000000000002c RDI: ffff881403e00000
[  867.089006] RBP: ffff88280c949b70 R08: 000000003b9aca00 R09: 0000000000000000
[  867.096975] R10: 0000000100000000 R11: 000000000000a000 R12: 00000000000000e5
[  867.104945] R13: 00100013f435cf77 R14: 0000000000000001 R15: ffff881403e00000
[  867.112914] FS:  00007f7b1d809700(0000) GS:ffff88142f8c0000(0000) knlGS:0000000000000000
[  867.121951] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  867.128371] CR2: ffff883409780158 CR3: 000000140d912000 CR4: 00000000001427e0
[  867.136333] Stack:
[  867.138570]  ffffffffa03c39e0 ffffffffa03fa66c ffff88280e263180 ffff881403e00000
[  867.146876]  ffff88140b758190 0000000c00000000 ffff8827f5d12000 0000000000000000
[  867.155180]  00000013fd854107 0000000000000002 ffff881403e00000 ffff88280c949bd8
[  867.163483] Call Trace:
[  867.166227]  [<ffffffffa03c39e0>] ? kvm_release_pfn_clean+0x40/0x50 [kvm]
[  867.173821]  [<ffffffffa03fa66c>] ? rr_memory_cow+0x4c/0x190 [kvm]
[  867.180734]  [<ffffffffa03fc83c>] __rr_walk_spt+0x5c/0x3e0 [kvm]
[  867.187457]  [<ffffffffa03e4e15>] ? __direct_map.isra.100+0x295/0x2b0 [kvm]
[  867.195243]  [<ffffffffa03fc83c>] __rr_walk_spt+0x5c/0x3e0 [kvm]
[  867.201962]  [<ffffffffa03fc83c>] __rr_walk_spt+0x5c/0x3e0 [kvm]
[  867.208680]  [<ffffffffa03fcc57>] rr_check_chunk+0x97/0x163a [kvm]
[  867.215583]  [<ffffffff81096755>] ? __vtime_account_system+0x35/0x40
[  867.222672]  [<ffffffff81096c79>] ? vtime_guest_exit+0x49/0x50
[  867.229190]  [<ffffffffa03d9110>] kvm_arch_vcpu_ioctl_run+0x720/0x1590 [kvm]
[  867.237071]  [<ffffffffa03c3f22>] kvm_vcpu_ioctl+0x2c2/0x5b0 [kvm]
[  867.243978]  [<ffffffff81102d4c>] ? acct_account_cputime+0x1c/0x20
[  867.250885]  [<ffffffff811b6e95>] do_vfs_ioctl+0x2e5/0x4d0
[  867.257013]  [<ffffffff81096b69>] ? vtime_account_user+0x69/0x80
[  867.263722]  [<ffffffff811b7101>] SyS_ioctl+0x81/0xa0
[  867.269360]  [<ffffffff816e61eb>] tracesys+0xdd/0xe2
```

#### Problem 2: IOAPIC crash
```
[  503.623279] Kernel BUG at ffffffffa044ebb7 [verbose debug info unavailable]
[  503.631056] invalid opcode: 0000 [#1] SMP
[  503.635650] Modules linked in: kvm_intel(OF) kvm(OF) nfsv3 nfs_acl nfs lockd fscache x86_pkg_temp_thermal coretemp crc32_pclmul ghash_clmulni_intel aesni_intel aes_x86_64 lrw gpio_ich joydev gf128mul glue_helper lpc_ich ablk_helper cryptd shpchp acpi_power_meter acpi_pad mac_hid binfmt_misc ib_iser rdma_cm ib_addr iw_cm ib_cm ib_sa ib_mad ib_core sunrpc iscsi_tcp libiscsi_tcp libiscsi scsi_transport_iscsi ip_tables x_tables autofs4 ixgbe hid_generic igb usbhid dca hid mxm_wmi ptp mpt3sas ahci pps_core raid_class libahci scsi_transport_sas mdio i2c_algo_bit wmi [last unloaded: kvm]
[  503.693778] CPU: 0 PID: 3037 Comm: qemu-system-x86 Tainted: GF          O 3.11.0+ #1
[  503.702424] Hardware name: Cisco Systems Inc UCSC-C220-M4S/UCSC-C220-M4S, BIOS C220M4.4.1.2c.0.0202211901 02/02/2021
[  503.714174] task: ffff881408614650 ti: ffff881405afc000 task.ti: ffff881405afc000
[  503.722531] RIP: 0010:[<ffffffffa044ebb7>]  [<ffffffffa044ebb7>] ioapic_service+0xf7/0x100 [kvm]
[  503.732374] RSP: 0018:ffff881405afdc48  EFLAGS: 00010286
[  503.738296] RAX: 00000000ffffffff RBX: ffff881401e3fc00 RCX: 0000000000000000
[  503.746262] RDX: 0000000000000001 RSI: 0000000000000008 RDI: 0000000000000000
[  503.754230] RBP: ffff881405afdc78 R08: 0000000000000001 R09: 0000000000000000
[  503.762200] R10: 0000000000000000 R11: 0000000000000246 R12: ffff881401e3fc58
[  503.770168] R13: ffff881401e3fc00 R14: ffff881401e3fdb0 R15: 0000000000000100
[  503.778137] FS:  00007f42aab22740(0000) GS:ffff88142f800000(0000) knlGS:0000000000000000
[  503.787172] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  503.793586] CR2: 00007f42aa2381b7 CR3: 00000013f3954000 CR4: 00000000001427f0
[  503.801556] Stack:
[  503.803801]  0000000005afdc70 0000000100000700 0000000000000001 0000000000000000
[  503.812105]  0000000000000008 0000000000000f00 ffff881405afdcb8 ffffffffa044f1d0
[  503.820409]  0000000000000001 0000000000000000 0000000000000001 00000000ffffffff
[  503.828713] Call Trace:
[  503.831457]  [<ffffffffa044f1d0>] kvm_ioapic_set_irq+0xc0/0x1a0 [kvm]
[  503.838658]  [<ffffffffa044fcfe>] kvm_set_ioapic_irq+0x1e/0x20 [kvm]
[  503.845762]  [<ffffffffa04517ac>] kvm_set_irq+0xec/0x150 [kvm]
[  503.852280]  [<ffffffffa044fce0>] ? kvm_vm_ioctl_unregister_coalesced_mmio+0xe0/0xe0 [kvm]
[  503.861519]  [<ffffffffa044fd00>] ? kvm_set_ioapic_irq+0x20/0x20 [kvm]
[  503.868822]  [<ffffffffa045e989>] kvm_vm_ioctl_irq_line+0x29/0x40 [kvm]
[  503.876216]  [<ffffffffa044dbb4>] kvm_vm_ioctl+0x644/0x790 [kvm]
[  503.882929]  [<ffffffff81019d09>] ? read_tsc+0x9/0x20
[  503.888575]  [<ffffffff810b6413>] ? ktime_get+0x43/0xc0
[  503.894412]  [<ffffffff81040204>] ? lapic_next_deadline+0x34/0x40
[  503.901220]  [<ffffffff810bcc3b>] ? clockevents_program_event+0x6b/0xf0
[  503.908609]  [<ffffffff81086fc1>] ? __hrtimer_start_range_ns+0x1b1/0x3e0
[  503.916097]  [<ffffffff811b6e95>] do_vfs_ioctl+0x2e5/0x4d0
[  503.922224]  [<ffffffff81096b69>] ? vtime_account_user+0x69/0x80
[  503.928934]  [<ffffffff811b7101>] SyS_ioctl+0x81/0xa0
[  503.934579]  [<ffffffff816e61eb>] tracesys+0xdd/0xe2
[  503.940121] Code: 00 00 00 00 48 8b bb a0 01 00 00 48 8d 55 d4 31 c9 31 f6 e8 7c 11 00 00 eb c1 66 2e 0f 1f 84 00 00 00 00 00 b8 ff ff ff ff eb c6 <0f> 0b 0f 1f 80 00 00 00 00
 0f 1f 44 00 00 55 31 c0 48 89 e5 48
[  503.961845] RIP  [<ffffffffa044ebb7>] ioapic_service+0xf7/0x100 [kvm]
```
If add -machine pc,kernel_irqchip=off to disable ioapic, this error will be gone.


#### Problem 3: System gets hanging forever on record + multi-core

With 2GB memory and kernel_irqchip=off, raise the core number to 2 or 4, initially the system can run well, but as far as the recording starts, the system starts hanging.

Therefore, so far, only record on one core can work.
