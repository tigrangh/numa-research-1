# Host emulation

## Qemu

```
qemu-system-x86_64 \
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
  -netdev user,id=net2
```

## Verifying the architecture

```
[root@localhost ~]# numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3
node 0 size: 1653 MB
node 0 free: 1332 MB
node 1 cpus: 4 5 6 7
node 1 size: 1963 MB
node 1 free: 1721 MB
node distances:
node     0    1 
   0:   10   20 
   1:   20   10
```

```
[root@localhost ~]# lspci -t
-+-[0000:00]-+-00.0
 |           +-01.0
 |           +-02.0
 |           +-03.0
 |           +-1d.0
 |           +-1d.1
 |           +-1d.2
 |           +-1d.7
 |           +-1f.0
 |           +-1f.2
 |           \-1f.3
 +-[0000:02]-+-00.0-[03]----00.0
 |           \-01.0-[04]----00.0
 \-[0000:80]-+-00.0-[81]----00.0
             \-01.0-[82]----00.0
```

```
[root@localhost ~]# lspci -v
00:00.0 Host bridge: Intel Corporation 82G33/G31/P35/P31 Express DRAM Controller
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Flags: bus master, fast devsel, latency 0

00:01.0 VGA compatible controller: Device 1234:1111 (rev 02) (prog-if 00 [VGA controller])
	Subsystem: Red Hat, Inc. Device 1100
	Flags: bus master, fast devsel, latency 0
	Memory at 80000000 (32-bit, prefetchable) [size=16M]
	Memory at 81012000 (32-bit, non-prefetchable) [size=4K]
	Expansion ROM at 000c0000 [disabled] [size=128K]
	Kernel driver in use: bochs-drm
	Kernel modules: bochs

00:02.0 Host bridge: Red Hat, Inc. QEMU PCIe Expander bridge
	Subsystem: Red Hat, Inc. Device 1100
	Flags: bus master, 66MHz, fast devsel, latency 0

00:03.0 Host bridge: Red Hat, Inc. QEMU PCIe Expander bridge
	Subsystem: Red Hat, Inc. Device 1100
	Flags: bus master, 66MHz, fast devsel, latency 0

00:1d.0 USB controller: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #1 (rev 03) (prog-if 00 [UHCI])
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Flags: bus master, fast devsel, latency 0, IRQ 16
	I/O ports at 60a0 [size=32]
	Kernel driver in use: uhci_hcd

00:1d.1 USB controller: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #2 (rev 03) (prog-if 00 [UHCI])
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Flags: bus master, fast devsel, latency 0, IRQ 17
	I/O ports at 6080 [size=32]
	Kernel driver in use: uhci_hcd

00:1d.2 USB controller: Intel Corporation 82801I (ICH9 Family) USB UHCI Controller #3 (rev 03) (prog-if 00 [UHCI])
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Flags: bus master, fast devsel, latency 0, IRQ 18
	I/O ports at 6060 [size=32]
	Kernel driver in use: uhci_hcd

00:1d.7 USB controller: Intel Corporation 82801I (ICH9 Family) USB2 EHCI Controller #1 (rev 03) (prog-if 20 [EHCI])
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Flags: bus master, fast devsel, latency 0, IRQ 19
	Memory at 81011000 (32-bit, non-prefetchable) [size=4K]
	Kernel driver in use: ehci-pci

00:1f.0 ISA bridge: Intel Corporation 82801IB (ICH9) LPC Interface Controller (rev 02)
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Flags: bus master, fast devsel, latency 0
	Kernel driver in use: lpc_ich
	Kernel modules: lpc_ich

00:1f.2 SATA controller: Intel Corporation 82801IR/IO/IH (ICH9R/DO/DH) 6 port SATA Controller [AHCI mode] (rev 02) (prog-if 01 [AHCI 1.0])
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Flags: bus master, fast devsel, latency 0, IRQ 34
	I/O ports at 6040 [size=32]
	Memory at 81010000 (32-bit, non-prefetchable) [size=4K]
	Capabilities: [80] MSI: Enable+ Count=1/1 Maskable- 64bit+
	Capabilities: [a8] SATA HBA v1.0
	Kernel driver in use: ahci
	Kernel modules: ahci

00:1f.3 SMBus: Intel Corporation 82801I (ICH9 Family) SMBus Controller (rev 02)
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Flags: bus master, fast devsel, latency 0, IRQ 16
	I/O ports at 6000 [size=64]
	Kernel driver in use: i801_smbus
	Kernel modules: i2c_i801

02:00.0 PCI bridge: Red Hat, Inc. QEMU PCIe Root port (prog-if 00 [Normal decode])
	Subsystem: Red Hat, Inc. Device 0000
	Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 0
	Memory at 81641000 (32-bit, non-prefetchable) [size=4K]
	Bus: primary=02, secondary=03, subordinate=03, sec-latency=0
	I/O behind bridge: [disabled] [16-bit]
	Memory behind bridge: 81400000-815fffff [size=2M] [32-bit]
	Prefetchable memory behind bridge: 800000000-8000fffff [size=1M] [32-bit]
	Capabilities: [54] Express Root Port (Slot+), IntMsgNum 0
	Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
	Capabilities: [40] Subsystem: Red Hat, Inc. Device 0000
	Capabilities: [100] Advanced Error Reporting
	Capabilities: [148] Access Control Services
	Kernel driver in use: pcieport

02:01.0 PCI bridge: Red Hat, Inc. QEMU PCIe Root port (prog-if 00 [Normal decode])
	Subsystem: Red Hat, Inc. Device 0000
	Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 0
	Memory at 81640000 (32-bit, non-prefetchable) [size=4K]
	Bus: primary=02, secondary=04, subordinate=04, sec-latency=0
	I/O behind bridge: [disabled] [16-bit]
	Memory behind bridge: 81200000-813fffff [size=2M] [32-bit]
	Prefetchable memory behind bridge: 800100000-8001fffff [size=1M] [32-bit]
	Capabilities: [54] Express Root Port (Slot+), IntMsgNum 0
	Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
	Capabilities: [40] Subsystem: Red Hat, Inc. Device 0000
	Capabilities: [100] Advanced Error Reporting
	Capabilities: [148] Access Control Services
	Kernel driver in use: pcieport

03:00.0 Display controller: Red Hat, Inc. Virtio 1.0 GPU (rev 01)
	Subsystem: Red Hat, Inc. QEMU
	Physical Slot: 0
	Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 0
	Memory at 81400000 (32-bit, non-prefetchable) [size=4K]
	Memory at 800000000 (64-bit, prefetchable) [size=16K]
	Capabilities: [dc] MSI-X: Enable+ Count=3 Masked-
	Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
	Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
	Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
	Capabilities: [94] Vendor Specific Information: VirtIO: ISR
	Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
	Capabilities: [7c] Power Management version 3
	Capabilities: [40] Express Endpoint, IntMsgNum 0
	Kernel driver in use: virtio-pci

04:00.0 Ethernet controller: Red Hat, Inc. Virtio 1.0 network device (rev 01)
	Subsystem: Red Hat, Inc. QEMU
	Physical Slot: 2
	Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 0
	Memory at 81200000 (32-bit, non-prefetchable) [size=4K]
	Memory at 800100000 (64-bit, prefetchable) [size=16K]
	Expansion ROM at 81240000 [disabled] [size=256K]
	Capabilities: [dc] MSI-X: Enable+ Count=4 Masked-
	Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
	Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
	Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
	Capabilities: [94] Vendor Specific Information: VirtIO: ISR
	Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
	Capabilities: [7c] Power Management version 3
	Capabilities: [40] Express Endpoint, IntMsgNum 0
	Kernel driver in use: virtio-pci

80:00.0 PCI bridge: Red Hat, Inc. QEMU PCIe Root port (prog-if 00 [Normal decode])
	Subsystem: Red Hat, Inc. Device 0000
	Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 1
	Memory at 81c41000 (32-bit, non-prefetchable) [size=4K]
	Bus: primary=80, secondary=81, subordinate=81, sec-latency=0
	I/O behind bridge: [disabled] [16-bit]
	Memory behind bridge: 81a00000-81bfffff [size=2M] [32-bit]
	Prefetchable memory behind bridge: 800200000-8002fffff [size=1M] [32-bit]
	Capabilities: [54] Express Root Port (Slot+), IntMsgNum 0
	Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
	Capabilities: [40] Subsystem: Red Hat, Inc. Device 0000
	Capabilities: [100] Advanced Error Reporting
	Capabilities: [148] Access Control Services
	Kernel driver in use: pcieport

80:01.0 PCI bridge: Red Hat, Inc. QEMU PCIe Root port (prog-if 00 [Normal decode])
	Subsystem: Red Hat, Inc. Device 0000
	Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 1
	Memory at 81c40000 (32-bit, non-prefetchable) [size=4K]
	Bus: primary=80, secondary=82, subordinate=82, sec-latency=0
	I/O behind bridge: [disabled] [16-bit]
	Memory behind bridge: 81800000-819fffff [size=2M] [32-bit]
	Prefetchable memory behind bridge: 800300000-8003fffff [size=1M] [32-bit]
	Capabilities: [54] Express Root Port (Slot+), IntMsgNum 0
	Capabilities: [48] MSI-X: Enable+ Count=1 Masked-
	Capabilities: [40] Subsystem: Red Hat, Inc. Device 0000
	Capabilities: [100] Advanced Error Reporting
	Capabilities: [148] Access Control Services
	Kernel driver in use: pcieport

81:00.0 Display controller: Red Hat, Inc. Virtio 1.0 GPU (rev 01)
	Subsystem: Red Hat, Inc. QEMU
	Physical Slot: 1
	Flags: bus master, fast devsel, latency 0, IRQ 11, NUMA node 1
	Memory at 81a00000 (32-bit, non-prefetchable) [size=4K]
	Memory at 800200000 (64-bit, prefetchable) [size=16K]
	Capabilities: [dc] MSI-X: Enable+ Count=3 Masked-
	Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
	Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
	Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
	Capabilities: [94] Vendor Specific Information: VirtIO: ISR
	Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
	Capabilities: [7c] Power Management version 3
	Capabilities: [40] Express Endpoint, IntMsgNum 0
	Kernel driver in use: virtio-pci

82:00.0 Ethernet controller: Red Hat, Inc. Virtio 1.0 network device (rev 01)
	Subsystem: Red Hat, Inc. QEMU
	Physical Slot: 3
	Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 1
	Memory at 81800000 (32-bit, non-prefetchable) [size=4K]
	Memory at 800300000 (64-bit, prefetchable) [size=16K]
	Expansion ROM at 81840000 [disabled] [size=256K]
	Capabilities: [dc] MSI-X: Enable+ Count=4 Masked-
	Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
	Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
	Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
	Capabilities: [94] Vendor Specific Information: VirtIO: ISR
	Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
	Capabilities: [7c] Power Management version 3
	Capabilities: [40] Express Endpoint, IntMsgNum 0
	Kernel driver in use: virtio-pci
```
