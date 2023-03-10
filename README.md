# Windows VMs imaging

An LXD-oriented Windows VMs imaging toolset.


## Features

- WinRM access (Ansible-ready).
- High-performance power configuration.
- Most/all drivers installed.
- Serial console access (disabled by default).


## Supported versions

The following Windows versions are supported.

- Windows 10 Enterprise
- Windows Server 2008 R2 SP1 (using libvirtd)
- Windows Server 2012
- Windows Server 2016
- Windows Server 2019
- Windows Server 2022


## Requirements

- curl
- lxd
- make
- python
- xorriso


## Usage

```
# Build a disk image (disk.qcow2) and metadata archive (lxd.tar.xz)
# in ./output/win2022/
make 2022

# Import into LXD using helper script
sh ./tools/import.sh ./output/win2022/

# Create and launch the virtual machine
lxc launch win2022 w22 -c security.secureboot=false
```

All systems have an administrator-level account named `admin` with
password `changeme`.


## Considerations

### Missing updates

Configurations are meant to support offline installs. This supports
ensuring that no updates are part of the images. In other words, we're
able to control when, if any, updates/patches will be installed.

### Storage space

Make sure the project's partition has enough storage space.

- The Windows ISOs and virtio drivers' ISO are automatically downloaded
  into the `./isos/` directory.
- The `./tmp/` directory is used to repack the virtio drivers' ISO with
  additional installation files.
- The `./output/` directory will contain the VM's disk image, metadata
  tarball and unattended install ISO.

### Windows Server 2008 R2 SP1

WWindows Server 2008 R2 SP1 is EOL since 2020. However, it still used by
some organizations. For this reason, being able to deploy it in a lab
environment is desired. LXD, however, does not support it due to its
use of modern OS and virtualization features, and lack of flexibility in
regards to QEMU parameters. libvirtd can be used in parallel to LXD to
deploy that Windows Server edition. When targeting 2k8R2, LXD is not used.
This project only provides a way to build an unattended install ISO and
a libvirtd domain XML file. Additionally, automatic configuration is
only minimally done/supported. If you'd like to built a relatively
similar image:

- let Windows install itself using the provided `Autounattend.xml`;
- on the desktop, open `powershell` and run `E:\local\install-ps3.ps1`;
- on reboot, open `powershell` and run `E:\local\ConfigureRemotingForAnsible.ps1`;
- run `E:\local\power.ps1`;
- run `E:\local\qemu-ga.ps1`;
- finally, run `E:\local\sysprep.bat`.

### Serial console access

It is possible to access Windows's serial console interface by
installing the EMS/SAC service. An installation script is provided by
disabled. To install EMS/SAC simply uncomment the stanza in the
`Autounattend.xml` file. Note that installing EMS/SAC on Windows 10
requires network connectivity and that it appears to be broken on
Server 2012.

```
lxc console w22
SAC> cmd
SAC> ch -si 1
Username: admin
Domain:
Password: changeme
C:\Windows\System32>
```


## Funny stuff

- LXD supports setting the UEFI's boot priority through a
  _device_ entry with the `boot.priority` parameter. This means
  that we can auto-"boot" to Window's ISO and launch the installer
  instead of spamming the Escape key to enter the boot menu. It is
  also possible to add ISOs using the `raw.qemu` parameter with a
  `-drive` option. The implication of using the former is
  that Windows won't mount the drive as a `X:\`-type drive.

  Windows Server 2019 and 2022 will install without issues with the
  ISOs attached to the VM in a split setup. However, versions 10,
  2012 and 2016 will only install if both ISOs are attached using
  `raw.qemu`. Without that configuration, the Windows installer will
  simply error out with an `invalid <ProductKey>` message.

  In other words, with the `device` parameter for booting and the
  double `-drive` parameter in `raw.qemu`, Windows 2012, 2016 and 10
  will see two (2) drives, but Windows 2019 and 2022 will see three (3).
  `Autounattend.xml` files have to be adjusted for this configuration.
  haha no, not exactly! Windows 10 sees two (2) devices during
  installation and three (3) afterwards. :p
- EMS/SAC is installed on the Server 2012 image, but doesn't seem to
  work.


## References

- https://github.com/ruzickap/packer-templates
- https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-server-eos-faq/end-of-support-windows-server-2008-2008r2
- https://github.com/lxc/lxd/commit/f14c88de78bf9f2bbe91dd661004ab772ccf179e
- https://bugs.launchpad.net/qemu/+bug/1593605
