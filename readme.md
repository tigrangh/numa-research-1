# Index

## [Host emulation](#host-emulation-1)
### [Qemu](#qemu-1)
### [Enable the serial console access](#enable-the-serial-console-access-1)
### [Add iommu related kernel cmdline](#add-iommu-related-kernel-cmdline-1)
### [Prepare VFIO related modules](#prepare-vfio-related-modules-1)
### [Verify the architecture](#verify-the-architecture-1)
### [Verify the IOMMU groups](#verify-the-iommu-groups-1)

## Host emulation

### Qemu

```
qemu-system-x86_64 \
  -nodefaults \
  -serial stdio \
  -nographic \
  -bios /usr/share/edk2-ovmf/OVMF_CODE.fd \
  -enable-kvm \
  -machine type=q35,accel=kvm \
  -m 4G \
  -cpu Skylake-Client-v1 \
  -drive file=disk.qcow2,format=qcow2 \
  -usb -device usb-tablet \
  \
  -smp 8,sockets=2,cores=4,threads=1 \
  \
  -object memory-backend-ram,id=mem0,size=2G \
  -object memory-backend-ram,id=mem1,size=2G \
  \
  -numa node,nodeid=0,cpus=0-3,memdev=mem0 \
  -numa node,nodeid=1,cpus=4-7,memdev=mem1 \
  \
  -device pxb-pcie,id=pxbpcie.1,bus_nr=2,bus=pcie.0,numa_node=0 \
  -device pxb-pcie,id=pxbpcie.2,bus_nr=128,bus=pcie.0,numa_node=1 \
  -device pcie-root-port,id=pcierootport.gpu.1,bus=pxbpcie.1,slot=0 \
  -device pcie-root-port,id=pcierootport.gpu.2,bus=pxbpcie.2,slot=1 \
  -device virtio-gpu-pci,bus=pcierootport.gpu.1 \
  -device virtio-gpu-pci,bus=pcierootport.gpu.2 \
  -device pcie-root-port,id=pcierootport.net.1,bus=pxbpcie.1,slot=2 \
  -device virtio-net-pci,netdev=net1,bus=pcierootport.net.1 \
  -netdev user,id=net1 \
  -device pcie-root-port,id=pcierootport.net.2,bus=pxbpcie.2,slot=3 \
  -device virtio-net-pci,netdev=net2,bus=pcierootport.net.2 \
  -netdev user,id=net2 \
  -device pcie-root-port,id=pcierootport.nvme.1,bus=pxbpcie.1,slot=4 \
  -device pcie-root-port,id=pcierootport.nvme.2,bus=pxbpcie.2,slot=5 \
  -drive file=nvme1.qcow2,if=none,id=nvme1,format=qcow2 \
  -device nvme,drive=nvme1,serial=nvme1,bus=pcierootport.nvme.1 \
  -drive file=nvme2.qcow2,if=none,id=nvme2,format=qcow2 \
  -device nvme,drive=nvme2,serial=nvme2,bus=pcierootport.nvme.2 \
  -device intel-iommu,caching-mode=on
```

### Enable the serial console access

Set Persistent Text Mode: `systemctl set-default multi-user.target.`
Set Persistent GUI Mode: `systemctl set-default graphical.target.`

```
systemctl enable serial-getty@ttyS0.service
```

### Add iommu related kernel cmdline

```
grubby --update-kernel=ALL --args="intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction"
```
### Prepare VFIO related modules

```
printf "options vfio-pci ids=1af4:1050,1af4:1041,1b36:0010\n" > /etc/modprobe.d/vfio.conf
printf "vfio\nvfio_iommu_type1\nvfio_pci\n#vfio_virqfd\n" > /etc/modules-load.d/vfio.conf
```

_`comment: looks like virtio-gpu-pci can be used for vfio passthrough`_

### Verify the architecture

```
[root@localhost ~]# numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3
node 0 size: 1607 MB
node 0 free: 1240 MB
node 1 cpus: 4 5 6 7
node 1 size: 2011 MB
node 1 free: 1726 MB
node distances:
node     0    1 
   0:   10   20 
   1:   20   10
```

```
[root@localhost ~]# lspci -t
-+-[0000:80]-+-00.0-[81]----00.0
 |           +-01.0-[82]----00.0
 |           \-02.0-[83]----00.0
 +-[0000:02]-+-00.0-[03]----00.0
 |           +-01.0-[04]----00.0
 |           \-02.0-[05]----00.0
 \-[0000:00]-+-00.0
             +-01.0
             +-02.0
             +-1d.0
             +-1d.1
             +-1d.2
             +-1d.7
             +-1f.0
             +-1f.2
             \-1f.3
```

```
[root@localhost ~]# lspci -v -nn -k
00:00.0 Host bridge [0600]: Intel Corporation 82G33/G31/P35/P31 Express DRAM Controller [8086:29c0]
        Subsystem: Red Hat, Inc. QEMU Virtual Machine [1af4:1100]
        Flags: bus master, fast devsel, latency 0, IOMMU group 0

00:01.0 Host bridge [0600]: Red Hat, Inc. QEMU PCIe Expander bridge [1b36:000b]
        Subsystem: Red Hat, Inc. Device [1af4:1100]
        Flags: bus master, 66MHz, fast devsel, latency 0, IOMMU group 1

00:02.0 Host bridge [0600]: Red Hat, Inc. QEMU PCIe Expander bridge [1b36:000b]
        Subsystem: Red Hat, Inc. Device [1af4:1100]
        Flags: bus master, 66MHz, fast devsel, latency 0, IOMMU group 2

00:1d.0 USB controller [0c03]: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #1 [8086:2934] (rev 03) (prog-if 00 [UHCI])
        Subsystem: Red Hat, Inc. QEMU Virtual Machine [1af4:1100]
        Flags: bus master, fast devsel, latency 0, IRQ 16, IOMMU group 3
        I/O ports at 60a0 [size=32]
        Kernel driver in use: uhci_hcd

00:1d.1 USB controller [0c03]: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #2 [8086:2935] (rev 03) (prog-if 00 [UHCI])
        Subsystem: Red Hat, Inc. QEMU Virtual Machine [1af4:1100]
        Flags: bus master, fast devsel, latency 0, IRQ 17, IOMMU group 3
        I/O ports at 6080 [size=32]
        Kernel driver in use: uhci_hcd

00:1d.2 USB controller [0c03]: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #3 [8086:2936] (rev 03) (prog-if 00 [UHCI])
        Subsystem: Red Hat, Inc. QEMU Virtual Machine [1af4:1100]
        Flags: bus master, fast devsel, latency 0, IRQ 18, IOMMU group 3
        I/O ports at 6060 [size=32]
        Kernel driver in use: uhci_hcd

00:1d.7 USB controller [0c03]: Intel Corporation 82801I (ICH9 Family) USB2 EHCI Controller #1 [8086:293a] (rev 03) (prog-if 20 [EHCI])
        Subsystem: Red Hat, Inc. QEMU Virtual Machine [1af4:1100]
        Flags: bus master, fast devsel, latency 0, IRQ 19, IOMMU group 3
        Memory at 80001000 (32-bit, non-prefetchable) [size=4K]
        Kernel driver in use: ehci-pci

00:1f.0 ISA bridge [0601]: Intel Corporation 82801IB (ICH9) LPC Interface Controller [8086:2918] (rev 02)
        Subsystem: Red Hat, Inc. QEMU Virtual Machine [1af4:1100]
        Flags: bus master, fast devsel, latency 0, IOMMU group 4
        Kernel driver in use: lpc_ich
        Kernel modules: lpc_ich

00:1f.2 SATA controller [0106]: Intel Corporation 82801IR/IO/IH (ICH9R/DO/DH) 6 port SATA Controller [AHCI mode] [8086:2922] (rev 02) (prog-if 01 [AHCI 1.0])
        Subsystem: Red Hat, Inc. QEMU Virtual Machine [1af4:1100]
        Flags: bus master, fast devsel, latency 0, IRQ 37, IOMMU group 4
        I/O ports at 6040 [size=32]
        Memory at 80000000 (32-bit, non-prefetchable) [size=4K]
        Capabilities: [80] MSI: Enable+ Count=1/1 Maskable- 64bit+
        Capabilities: [a8] SATA HBA v1.0
        Kernel driver in use: ahci
        Kernel modules: ahci

00:1f.3 SMBus [0c05]: Intel Corporation 82801I (ICH9 Family) SMBus Controller [8086:2930] (rev 02)
        Subsystem: Red Hat, Inc. QEMU Virtual Machine [1af4:1100]
        Flags: bus master, fast devsel, latency 0, IRQ 16, IOMMU group 4
        I/O ports at 6000 [size=64]
        Kernel driver in use: i801_smbus
        Kernel modules: i2c_i801

02:00.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c] (prog-if 00 [Normal decode])
        Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 0, IOMMU group 11
        Memory at 80842000 (32-bit, non-prefetchable) [size=4K]
        Bus: primary=02, secondary=03, subordinate=03, sec-latency=0
        I/O behind bridge: [disabled]
        Memory behind bridge: 80600000-807fffff [size=2M]
        Prefetchable memory behind bridge: 0000000800000000-00000008000fffff [size=1M]
        Capabilities: [54] Express Root Port (Slot+), MSI 00
        Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
        Capabilities: [40] Subsystem: Red Hat, Inc. Device [1b36:0000]
        Capabilities: [100] Advanced Error Reporting
        Capabilities: [148] Access Control Services
        Kernel driver in use: pcieport

02:01.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c] (prog-if 00 [Normal decode])
        Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 0, IOMMU group 12
        Memory at 80841000 (32-bit, non-prefetchable) [size=4K]
        Bus: primary=02, secondary=04, subordinate=04, sec-latency=0
        I/O behind bridge: [disabled]
        Memory behind bridge: 80400000-805fffff [size=2M]
        Prefetchable memory behind bridge: 0000000800100000-00000008001fffff [size=1M]
        Capabilities: [54] Express Root Port (Slot+), MSI 00
        Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
        Capabilities: [40] Subsystem: Red Hat, Inc. Device [1b36:0000]
        Capabilities: [100] Advanced Error Reporting
        Capabilities: [148] Access Control Services
        Kernel driver in use: pcieport

02:02.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c] (prog-if 00 [Normal decode])
        Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 0, IOMMU group 13
        Memory at 80840000 (32-bit, non-prefetchable) [size=4K]
        Bus: primary=02, secondary=05, subordinate=05, sec-latency=0
        I/O behind bridge: [disabled]
        Memory behind bridge: 80200000-803fffff [size=2M]
        Prefetchable memory behind bridge: [disabled]
        Capabilities: [54] Express Root Port (Slot+), MSI 00
        Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
        Capabilities: [40] Subsystem: Red Hat, Inc. Device [1b36:0000]
        Capabilities: [100] Advanced Error Reporting
        Capabilities: [148] Access Control Services
        Kernel driver in use: pcieport

03:00.0 Display controller [0380]: Red Hat, Inc. Virtio 1.0 GPU [1af4:1050] (rev 01)
        Subsystem: Red Hat, Inc. QEMU [1af4:1100]
        Physical Slot: 0
        Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 0, IOMMU group 14
        Memory at 80600000 (32-bit, non-prefetchable) [size=4K]
        Memory at 800000000 (64-bit, prefetchable) [size=16K]
        Capabilities: [dc] MSI-X: Enable+ Count=3 Masked-
        Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
        Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
        Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
        Capabilities: [94] Vendor Specific Information: VirtIO: ISR
        Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
        Capabilities: [7c] Power Management version 3
        Capabilities: [40] Express Endpoint, MSI 00
        Kernel driver in use: virtio-pci

04:00.0 Ethernet controller [0200]: Red Hat, Inc. Virtio 1.0 network device [1af4:1041] (rev 01)
        Subsystem: Red Hat, Inc. QEMU [1af4:1100]
        Physical Slot: 2
        Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 0, IOMMU group 15
        Memory at 80400000 (32-bit, non-prefetchable) [size=4K]
        Memory at 800100000 (64-bit, prefetchable) [size=16K]
        Expansion ROM at 80440000 [disabled] [size=256K]
        Capabilities: [dc] MSI-X: Enable+ Count=4 Masked-
        Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
        Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
        Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
        Capabilities: [94] Vendor Specific Information: VirtIO: ISR
        Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
        Capabilities: [7c] Power Management version 3
        Capabilities: [40] Express Endpoint, MSI 00
        Kernel driver in use: virtio-pci

05:00.0 Non-Volatile memory controller [0108]: Red Hat, Inc. QEMU NVM Express Controller [1b36:0010] (rev 02) (prog-if 02 [NVM Express])
        Subsystem: Red Hat, Inc. Device [1af4:1100]
        Physical Slot: 4
        Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 0, IOMMU group 16
        Memory at 80200000 (64-bit, non-prefetchable) [size=16K]
        Capabilities: [40] MSI-X: Enable- Count=65 Masked-
        Capabilities: [80] Express Endpoint, MSI 00
        Capabilities: [60] Power Management version 3
        Kernel driver in use: vfio-pci
        Kernel modules: nvme

80:00.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c] (prog-if 00 [Normal decode])
        Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 1, IOMMU group 5
        Memory at 81042000 (32-bit, non-prefetchable) [size=4K]
        Bus: primary=80, secondary=81, subordinate=81, sec-latency=0
        I/O behind bridge: [disabled]
        Memory behind bridge: 80e00000-80ffffff [size=2M]
        Prefetchable memory behind bridge: 0000000800200000-00000008002fffff [size=1M]
        Capabilities: [54] Express Root Port (Slot+), MSI 00
        Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
        Capabilities: [40] Subsystem: Red Hat, Inc. Device [1b36:0000]
        Kernel driver in use: pcieport

80:01.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c] (prog-if 00 [Normal decode])
        Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 1, IOMMU group 6
        Memory at 81041000 (32-bit, non-prefetchable) [size=4K]
        Bus: primary=80, secondary=82, subordinate=82, sec-latency=0
        I/O behind bridge: [disabled]
        Memory behind bridge: 80c00000-80dfffff [size=2M]
        Prefetchable memory behind bridge: 0000000800300000-00000008003fffff [size=1M]
        Capabilities: [54] Express Root Port (Slot+), MSI 00
        Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
        Capabilities: [40] Subsystem: Red Hat, Inc. Device [1b36:0000]
        Capabilities: [100] Advanced Error Reporting
        Capabilities: [148] Access Control Services
        Kernel driver in use: pcieport

80:02.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c] (prog-if 00 [Normal decode])
        Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 1, IOMMU group 7
        Memory at 81040000 (32-bit, non-prefetchable) [size=4K]
        Bus: primary=80, secondary=83, subordinate=83, sec-latency=0
        I/O behind bridge: [disabled]
        Memory behind bridge: 80a00000-80bfffff [size=2M]
        Prefetchable memory behind bridge: [disabled]
        Capabilities: [54] Express Root Port (Slot+), MSI 00
        Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
        Capabilities: [40] Subsystem: Red Hat, Inc. Device [1b36:0000]
        Capabilities: [100] Advanced Error Reporting
        Capabilities: [148] Access Control Services
        Kernel driver in use: pcieport

81:00.0 Display controller [0380]: Red Hat, Inc. Virtio 1.0 GPU [1af4:1050] (rev 01)
        Subsystem: Red Hat, Inc. QEMU [1af4:1100]
        Physical Slot: 1
        Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 1, IOMMU group 8
        Memory at 80e00000 (32-bit, non-prefetchable) [size=4K]
        Memory at 800200000 (64-bit, prefetchable) [size=16K]
        Capabilities: [dc] MSI-X: Enable+ Count=3 Masked-
        Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
        Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
        Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
        Capabilities: [94] Vendor Specific Information: VirtIO: ISR
        Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
        Capabilities: [7c] Power Management version 3
        Capabilities: [40] Express Endpoint, MSI 00
        Kernel driver in use: virtio-pci

82:00.0 Ethernet controller [0200]: Red Hat, Inc. Virtio 1.0 network device [1af4:1041] (rev 01)
        Subsystem: Red Hat, Inc. QEMU [1af4:1100]
        Physical Slot: 3
        Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 1, IOMMU group 9
        Memory at 80c00000 (32-bit, non-prefetchable) [size=4K]
        Memory at 800300000 (64-bit, prefetchable) [size=16K]
        Expansion ROM at 80c40000 [disabled] [size=256K]
        Capabilities: [dc] MSI-X: Enable+ Count=4 Masked-
        Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
        Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
        Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
        Capabilities: [94] Vendor Specific Information: VirtIO: ISR
        Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
        Capabilities: [7c] Power Management version 3
        Capabilities: [40] Express Endpoint, MSI 00
        Kernel driver in use: virtio-pci

83:00.0 Non-Volatile memory controller [0108]: Red Hat, Inc. QEMU NVM Express Controller [1b36:0010] (rev 02) (prog-if 02 [NVM Express])
        Subsystem: Red Hat, Inc. Device [1af4:1100]
        Physical Slot: 5
        Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 1, IOMMU group 10
        Memory at 80a00000 (64-bit, non-prefetchable) [size=16K]
        Capabilities: [40] MSI-X: Enable- Count=65 Masked-
        Capabilities: [80] Express Endpoint, MSI 00
        Capabilities: [60] Power Management version 3
        Kernel driver in use: vfio-pci
        Kernel modules: nvme
```


```
[root@localhost ~]# lscpu
Architecture:                x86_64
  CPU op-mode(s):            32-bit, 64-bit
  Address sizes:             40 bits physical, 48 bits virtual
  Byte Order:                Little Endian
CPU(s):                      8
  On-line CPU(s) list:       0-7
Vendor ID:                   GenuineIntel
  BIOS Vendor ID:            QEMU
  Model name:                Intel Core Processor (Skylake)
    BIOS Model name:         pc-q35-10.0
    CPU family:              6
    Model:                   94
    Thread(s) per core:      1
    Core(s) per socket:      4
    Socket(s):               2
    Stepping:                3
    BogoMIPS:                6815.96
    Flags:                   fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pg
                             e mca cmov pat pse36 clflush mmx fxsr sse sse2 ht s
                             yscall nx rdtscp lm constant_tsc rep_good nopl xtop
                             ology cpuid tsc_known_freq pni pclmulqdq ssse3 fma 
                             cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_dea
                             dline_timer aes xsave avx f16c rdrand hypervisor la
                             hf_lm abm 3dnowprefetch cpuid_fault pti fsgsbase bm
                             i1 hle avx2 smep bmi2 erms invpcid rtm rdseed adx s
                             map xsaveopt xsavec xgetbv1 arat
Virtualization features:     
  Hypervisor vendor:         KVM
  Virtualization type:       full
Caches (sum of all):         
  L1d:                       256 KiB (8 instances)
  L1i:                       256 KiB (8 instances)
  L2:                        32 MiB (8 instances)
  L3:                        32 MiB (2 instances)
NUMA:                        
  NUMA node(s):              2
  NUMA node0 CPU(s):         0-3
  NUMA node1 CPU(s):         4-7
Vulnerabilities:             
  Gather data sampling:      Unknown: Dependent on hypervisor status
  Indirect target selection: Mitigation; Aligned branch/return thunks
  Itlb multihit:             KVM: Mitigation: VMX unsupported
  L1tf:                      Mitigation; PTE Inversion
  Mds:                       Vulnerable: Clear CPU buffers attempted, no microco
                             de; SMT Host state unknown
  Meltdown:                  Mitigation; PTI
  Mmio stale data:           Vulnerable: Clear CPU buffers attempted, no microco
                             de; SMT Host state unknown
  Reg file data sampling:    Not affected
  Retbleed:                  Vulnerable
  Spec rstack overflow:      Not affected
  Spec store bypass:         Vulnerable
  Spectre v1:                Mitigation; usercopy/swapgs barriers and __user poi
                             nter sanitization
  Spectre v2:                Mitigation; Retpolines; STIBP disabled; RSB filling
                             ; PBRSB-eIBRS Not affected; BHI Retpoline
  Srbds:                     Unknown: Dependent on hypervisor status
  Tsa:                       Not affected
  Tsx async abort:           Vulnerable: Clear CPU buffers attempted, no microco
                             de; SMT Host state unknown
```

### Verify the IOMMU groups

```
[root@localhost ~]# for d in /sys/kernel/iommu_groups/*/devices/*; do n=${d#*/iommu_groups/*}; n=${n%%/*}; printf 'IOMMU group %s ' "$n"; lspci -nns "${d##*/}"; done
IOMMU group 0 00:00.0 Host bridge [0600]: Intel Corporation 82G33/G31/P35/P31 Express DRAM Controller [8086:29c0]
IOMMU group 10 83:00.0 Non-Volatile memory controller [0108]: Red Hat, Inc. QEMU NVM Express Controller [1b36:0010] (rev 02)
IOMMU group 11 02:00.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c]
IOMMU group 12 02:01.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c]
IOMMU group 13 02:02.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c]
IOMMU group 14 03:00.0 Display controller [0380]: Red Hat, Inc. Virtio 1.0 GPU [1af4:1050] (rev 01)
IOMMU group 15 04:00.0 Ethernet controller [0200]: Red Hat, Inc. Virtio 1.0 network device [1af4:1041] (rev 01)
IOMMU group 16 05:00.0 Non-Volatile memory controller [0108]: Red Hat, Inc. QEMU NVM Express Controller [1b36:0010] (rev 02)
IOMMU group 1 00:01.0 Host bridge [0600]: Red Hat, Inc. QEMU PCIe Expander bridge [1b36:000b]
IOMMU group 2 00:02.0 Host bridge [0600]: Red Hat, Inc. QEMU PCIe Expander bridge [1b36:000b]
IOMMU group 3 00:1d.0 USB controller [0c03]: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #1 [8086:2934] (rev 03)
IOMMU group 3 00:1d.1 USB controller [0c03]: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #2 [8086:2935] (rev 03)
IOMMU group 3 00:1d.2 USB controller [0c03]: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #3 [8086:2936] (rev 03)
IOMMU group 3 00:1d.7 USB controller [0c03]: Intel Corporation 82801I (ICH9 Family) USB2 EHCI Controller #1 [8086:293a] (rev 03)
IOMMU group 4 00:1f.0 ISA bridge [0601]: Intel Corporation 82801IB (ICH9) LPC Interface Controller [8086:2918] (rev 02)
IOMMU group 4 00:1f.2 SATA controller [0106]: Intel Corporation 82801IR/IO/IH (ICH9R/DO/DH) 6 port SATA Controller [AHCI mode] [8086:2922] (rev 02)
IOMMU group 4 00:1f.3 SMBus [0c05]: Intel Corporation 82801I (ICH9 Family) SMBus Controller [8086:2930] (rev 02)
IOMMU group 5 80:00.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c]
IOMMU group 6 80:01.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c]
IOMMU group 7 80:02.0 PCI bridge [0604]: Red Hat, Inc. QEMU PCIe Root port [1b36:000c]
IOMMU group 8 81:00.0 Display controller [0380]: Red Hat, Inc. Virtio 1.0 GPU [1af4:1050] (rev 01)
IOMMU group 9 82:00.0 Ethernet controller [0200]: Red Hat, Inc. Virtio 1.0 network device [1af4:1041] (rev 01)
```
