# Ansible Molecule tests

Tests run on VMs using Ansible Molecule, the Vagrant plugin and Vagrant's libvirt provider.

> **Note**
> Initial setup instructions are provided for Arch Linux to have a first working version, but a devcontainer should be
> defined in order to have a consistent environment for all contributors.

## Initial setup

### Arch Linux

Run all commands from the root of the repository.

Install the required Python packages:

```shell
uv pip install -r requirements.txt -r molecule/requirements.txt
```

Install libvirt and some vagrant dependencies.

```shell
pacman -S libvirt dnsmasq net-tools nfs-utils 
```

Install Vagrant and the libvirt provider:

```shell
yay -S vagrant
vagrant plugin install vagrant-libvirt
```

Read the
[troubleshooting section of the Vagrant's article on Arch wiki](https://wiki.archlinux.org/title/Vagrant#Troubleshooting).

Install `community.vagrant` Ansible collection.

```shell
ansible-galaxy collection install community.vagrant
```

### GitHub Actions

Running the following commands on a GitHub Actions runner leaves it ready to execute the tests. A GitHub Actions
workflow based on these instructions is defined in
[`.github/workflows/molecule.yml`](../.github/workflows/molecule.yml). 

Install the required Python packages:

```shell
uv pip install -r requirements.txt -r molecule/requirements.txt
```

Install Vagrant and libvirt.

```shell
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(. /etc/os-release && echo "$VERSION_CODENAME") main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
apt-get update
apt-get install --yes libvirt-daemon-system libvirt-dev qemu-system-x86 vagrant
```

Enable and start the libvirt service, grant the runner access to KVM and the libvirt socket.

```shell
systemctl enable --now libvirtd
chmod a+rw /dev/kvm /var/run/libvirt/libvirt-sock
```

Install the libvirt Vagrant provider:

```shell
vagrant plugin install vagrant-libvirt
```

Install `community.vagrant` Ansible collection.

```shell
ansible-galaxy collection install community.vagrant
```

## Running tests

Use the base configuration file when running the Ansible Molecule tests (do it from the root of the repository), like
this. 

```shell
molecule --base-config molecule/base.yml test
```
