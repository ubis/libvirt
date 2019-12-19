# libvirt GPU passthrough

This is quick summary of my libvirt GPU passthrough.

You should read the following links:
* https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF
* https://github.com/vanities/GPU-Passthrough-Arch-Linux-to-Windows10

#

## Hardware

* Intel i7-4790k @ 4.6GHz 1.37V
* Cooler Master Hyper Evo 212
* Gigabyte Z97X-Gaming-3 rev v1.1 (BIOS F8d)
* Crucial 2x8GB DDR3 1866MHz
* Samsung NVME SSD 970 EVO 250GB
* WD BLACK HDD 500GB 7200rpm 64MB
* NVIDIA GeForce GTX 1060 3GB

## Host

* OS: Arch Linux x86_64
* Kernel: 5.4.5-arch1-1 (or latest, i run `pacman -Syu` daily)
* GPU: Integrated Intel Graphics
* Disk: NVME SSD as main disk

## Guest

* OS: Windows 10 Pro
* Build: 1903
* GPU: NVIDIA GeForce GTX 1060
* Disk: WD Black HDD
* CPU: 2C/2T
* RAM: 8GB
* NIC: Bridge & NAT

#

My `/etc/default/grub` file contains:

`intel_iommu=on iommu=pt vfio-pci.ids=10de:1c02,10de:10f1`

in `GRUB_CMDLINE_LINUX_DEFAULT` option.

# Issues

#### Booting:
I haven't used qemu-img disk images for the Guest storage. Guest has VirtIO SCSI device which is at `/dev/sda` and it's my WD Black HDD.

I already had Windows pre-installed on that HDD and there was no VirtIO SCSI driver which lead Windows to BSOD with message `INACCESSIBLE BOOT DEVICE`.

To fix this I had to mount `virtio` disk with all the drivers on the Guest. Then boot into system restore mode (Windows does this automatically once there's 3 failed boots in a row) and open Command Line terminal to load the driver:

`drvload D:\viostor\w10\amd64\viostor.inf`

and install it:

`dism /image:C:\ /add-driver /driver:D:\viostor\w10\amd64\viostor.inf`

Note: after loading driver my HDD came up with `C:\` drive letter.

Read more here about this issue:
https://superuser.com/questions/1057959/windows-10-in-kvm-change-boot-disk-to-virtio

#### NVIDIA's Code 43:
To solve this one, I had to add `<vendor_id state="on" value="1234567890ab"/>` in `hyperv` section and
```
<kvm>
  <hidden state="on"/>
</kvm>
<vmport state="off"/>
<ioapic driver="kvm"/>
```
into `features` section in Guest's libvirt configuration.

#### CPU Pinning:
I passed 2C/2T to the Guest. I left **0 & 1** cores for my Host. My `lscpu -e` looks like this:
```
lscpu -e
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE    MAXMHZ   MINMHZ
  0    0      0    0 0:0:0:0          yes 4600,0000 800,0000
  1    0      0    1 1:1:1:0          yes 4600,0000 800,0000
  2    0      0    2 2:2:2:0          yes 4600,0000 800,0000
  3    0      0    3 3:3:3:0          yes 4600,0000 800,0000
  4    0      0    0 0:0:0:0          yes 4600,0000 800,0000
  5    0      0    1 1:1:1:0          yes 4600,0000 800,0000
  6    0      0    2 2:2:2:0          yes 4600,0000 800,0000
  7    0      0    3 3:3:3:0          yes 4600,0000 800,0000
```

And configuration:
```
<vcpu placement="static">4</vcpu>
<cputune>
  <vcpupin vcpu="0" cpuset="2"/>
  <vcpupin vcpu="1" cpuset="6"/>
  <vcpupin vcpu="2" cpuset="3"/>
  <vcpupin vcpu="3" cpuset="7"/>
</cputune>
```

With this configuration, I don't get any sound glitches/distortion anymore.

#### Sound:
Sound passthrough is pretty straightforward in Arch wiki. You change qemu's user to yours and add

`<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">`

and
```
<qemu:commandline>
  <qemu:env name="QEMU_AUDIO_DRV" value="pa"/>
  <qemu:env name="QEMU_PA_SERVER" value="/run/user/1000/pulse/native"/>
</qemu:commandline>
```

### 2019 12 19 UPDATE
While playing games I haven't noticed any sound glitches or distortions. However, I noticed them when listening to music. Someone in reddit suggested to change soundcard bitrate to match Host/Guest, since PulseAudio defaults it to 44100, so I changed on my Win10 Guest. This change improved things, but not so much. There's QEMU patches for sound fixes which I haven't tried yet. Usually I don't listen music from my Guest so I won't bother for now.

#### Network:
I have two NIC's configured on the Guest, one is bridge and another one is NAT.
As stated by libvirt/qemu: "bridge" won't work Host<->Guest communication, that's why NAT is used. I need Host<->Guest commucation for mouse/keyboard control with Barrier, read about that below.

#### Mouse & keyboard:
I use Barrier which is a fork of Synergy v1: https://github.com/debauchee/barrier
Server on Host, Client on Guest, everything works out of the box. NAT NIC is used for communication between server & client.

A few tips:
* To not lose mouse for a few seconds when UAC pops out in Guest: `change "elevate" to "always" in client settings`.
* While playing on the Guest, bind scrollock key to lock cursor to the screen, otherwise wierd stuff will happen, which depends on games as well.

#### Rest:
I don't know, probably I forgot something. Also look up my `win10.xml` configuration.