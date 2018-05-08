Install RedHat OpenShift Origin in your development box.

## Installation

1. Create a VM as explained in https://youtu.be/aqXSbDZggK4 (this video) by Grant Shipley

2. Define mandatory variables for the installation process

```
# Domain name to access the cluster
$ export DOMAIN=<public ip addres>.nip.io 

# User created after installation
$ export USERNAME=<current user name>

# Password for the user
$ export PASSWORD=password
```

3. Define optional variables for the installation process

```
# Instead of using loopback, setup DeviceMapper on this disk.
# !! All data on the disk will be wiped out !!
$ export DISK="/dev/sda"
```

3. Run the automagic installation script as root:

```
curl https://raw.githubusercontent.com/gshipley/installcentos/master/install-openshift.sh | /bin/bash
```

## Development

For development it's possible to switch the script repo

```
# Change location of source repository
$ export SCRIPT_REPO="https://raw.githubusercontent.com/gshipley/installcentos/master"
$ curl $SCRIPT_REPO/install-openshift.sh | /bin/bash
```
## DNS

This gist is mostly based on
[the dnsmasq appendix](https://github.com/openshift/training/blob/master/deprecated/beta-4-setup.md#appendix---dnsmasq-setup) from the
[openshift-training](https://github.com/openshift/training)
repo. However, I updated it and included fixes for the many gotchas I found along the way.

This is useful for folks who want to set up a DNS service as
part of the cluster itself, either because they cannot
easily change their DNS setup outside of the cluster, or
just because they want to keep the cluster setup
self-contained.

This is meant to be done *before* you run the
`openshift-ansible` playbook.

If you already have docker running, you will have to restart it after
doing these steps.

Eventually, I hope to convert this to an ansible playbook.

---

### A. Install dnsmasq

1. Choose and ssh into the node on which you want to install
dnsmasq. This node will be the one that all the other nodes
will contact for DNS resolution.  *Do not* install it on any
of the master nodes, since it will conflict with the
Kubernetes DNS service.

2. Install dnsmasq

    ```
    yum install -y dnsmasq
    ```

### B. Configure dnsmasq

1. Create the file `/etc/dnsmasq.d/openshift.conf` with the
following contents:

    ```
    strict-order
    domain-needed
    local=/example.com/
    bind-dynamic
    log-queries

    # If you want to map all *.cloudapps.example.com to one address (probably
    # something you do want if you're planning on exposing services to the
    # outside world using routes), then add the following line, replacing the
    # IP address with the IP of the node that will run the router. This
    # probably should not be one of your master nodes since they will most
    # likely be made unscheduleable (the default behaviour of the playbook).

    address=/.cloudapps.example.com/192.168.122.152

    # When you eventually create your router, make sure to use a selector
    # so that it is deployed on the node you chose, e.g. in my case:
    #   oadm router --create=true \
    #     --service-account=router \
    #     --credentials=/etc/origin/master/openshift-router.kubeconfig \
    #     --images='rcm-img-docker01.build.eng.bos.redhat.com:5001/openshift3/ose-${component}:${version}' \
    #     --selector=kubernetes.io/hostname=192.168.122.152
    ```

2. Add in the `/etc/hosts` file all the public IPs of the nodes and
their hostnames, e.g.


    ```
    192.168.122.26 ose3-master.example.com ose3-master
    192.168.122.90 ose3-node1.example.com ose3-node1
    192.168.122.152 ose3-node2.example.com ose3-node2
    ```

3. Copy the original `/etc/resolv.conf` to `/etc/resolv.conf.upstream`
and in `/etc/dnsmasq.conf`, set `resolve-file` to
`/etc/resolv.conf.upstream`.

### C. Configure `resolv.conf`

1. On *all the nodes*, make sure that `/etc/resolv.conf` has the IP of
the node running dnsmasq only. **Important:** on the node running
dnsmasq, do not use 127.0.0.1, use the actual cluster IP, just like the
other nodes. This is because docker by default
[ignores](http://docs.docker.com/engine/userguide/networking/default_network/configure-dns/)
local addresses
when copying `/etc/resolv.conf` into containers. Here's a sample
`/etc/resolv.conf`:

    ```
    search example.com
    # change this IP to the IP of the node running dnsmasq
    nameserver 192.168.122.90
    ```

2. Make sure that in `/etc/sysconfig/network-scripts/ifcfg-eth0`, the
variable `PEERDNS` is set to `no` so that `/etc/resolv.conf` doesn't get
overwritten on each reboot. Reboot the machines and check that
`/etc/resolv.conf` hasn't been overwritten.

### D. Open DNS port and enable dnsmasq

1. Finally, make sure port 53/udp is open on the dnsmasq node. We have
to use iptables rules for this, even if you have `firewalld` installed.
Otherwise, the `openshift-ansible` playbook will disable and mask it and
we will lose those rules. If you do have `firewalld`, let's mask it and
replace it with `iptables-services`:

    ```
    # systemctl stop firewalld
    # systemctl disable firewalld
    # systemctl mask firewalld
    # yum install -y iptables-services
    # systemctl enable iptables
    # systemctl start iptables
    ```

2. Install the DNS iptables rules

    ```
    # iptables -I INPUT 1 -p TCP --dport 53 -j ACCEPT
    # iptables -I INPUT 1 -p UDP --dport 53 -j ACCEPT
    # iptables-save > /etc/sysconfig/iptables
    ```

3. Restart the `iptables` service and make sure that the rules are still
there afterwards.

4. Enable and start dnsmasq:

    ```
    # systemctl enable dnsmasq
    # systemctl start dnsmasq
    ```

### E. Check that everything is working

1. To verify that everything is good, run the following on each node,
and check that the answered IP is correct (`dig` is in the `bind-utils`
package if you need to install it):

    ```
    # Check that nodes can look each other up (replace names as needed)
    [root@ose3-node1]# dig ose3-node2.example.com
    # Check that nodes can look up the wildcard domain. This should
    # return the address of the node that will run your router.
    [root@ose3-node1]# dig foo.cloudapps.example.com
    ```
