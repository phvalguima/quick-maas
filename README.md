## prepare a bare metal instance

Launch an i3.metal EC2 instance in the Frankfurt region by hand. If you
get "Your account is currently being verified", have a cup of coffee.

```
Your account is currently being verified. Verification normally
takes less than 2 hours. Until your account is verified, you may not be
able to launch additional instances or create additional volumes. If you
are still receiving this message after more than 2 hours, please let us
know by writing to aws-verification@amazon.com. We appreciate your
patience.
```

## prepare ssh_config on your local machine

Write `~/.ssh/config`.

```
Host demo-maas
    Hostname 10.0.10.10
    User ubuntu
    ProxyJump demo-i3-metal-frankfurt
    # IdentityFile /PATH/TO/YOUR/KEY

Host demo-i3-metal-frankfurt
    Hostname <GLOBAL_IP_ADDRESS_OF_THE_INSTANCE>
    User ubuntu
    # IdentityFile /PATH/TO/YOUR/KEY
```

Then, SSH to the instance and import keys.

``` bash
[local] $ ssh demo-i3-metal-frankfurt

[baremetal] $ ssh-import-id alitvinov calvinh nobuto yoshikadokawa vlgrevtsev
```

## prepare a LXD container to have a clean MAAS environment

Setup LXD on the bare metal to have a clean MAAS environment to be
confined in a container so you can reset the MAAS env without respawning
the bare metal instance. LXD 3.0 is to be used for easier preseeding.

```bash
[baremetal] $ sudo apt autoremove --purge lxd lxd-client
$ sudo snap install --channel 3.0/stable lxd


$ cat <<EOF | sudo lxd init --preseed
config: {}
networks:
- config:
    ipv4.address: 10.0.10.1/24
    ipv4.nat: "true"
    ipv4.dhcp.ranges: 10.0.10.51-10.0.10.200
    ipv6.address: none
  description: ""
  managed: false
  name: lxdbr0
  type: ""
storage_pools:
- config:
    source: /dev/nvme1n1
  description: ""
  name: default
  driver: lvm
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      nictype: bridged
      parent: lxdbr0
      type: nic
    root:
      path: /
      pool: default
      size: 32GB
      type: disk
  name: default
cluster: null
EOF
```

## clone the repository

Make sure you use the "baremetal" branch.

```bash
[baremetal] $ git clone -b baremetal https://github.com/nobuto-m/quick-maas.git
$ cd quick-maas
```


## run

Run the script to create the "quick-maas" LXD container to run MAAS and
MAAS Pod inside it. OpenStack will be deployed as the part of it.

```bash
[baremetal] $ ./run.sh
```


### how to access

Copy authorized_keys from the bare metal instance to the container.

```bash
[baremetal] $ lxc file push ~/.ssh/authorized_keys quick-maas/home/ubuntu/.ssh/
```

Then, ssh.

```bash
[local] $ ssh demo-maas
```

Open sshuttle for web browser.

```bash
[local] $ sshuttle -r demo-maas -N
```

### how to redeploy OpenStack

```
[container] $ juju destroy-model -y default && juju add-model default
[container] $ juju deploy openstack-base-51 # cloud:xenial-pike
[container] $ juju config neutron-gateway data-port='br-ex:ens7'
```

## TODO

image metadata

juju deploy kubernetes-core-346 # 1.10
