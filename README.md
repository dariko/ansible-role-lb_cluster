# lb_cluster ansible role

This role manages a group of hosts as an active-active cluster of load
balancers using keepalived/ipvs.

## Configuration

These variables are used by the role

|variable|required|default|description|
|--------|--------|-------|-----------|
|lb_cluster_nodes|yes||list of cluster members names|
|lb_cluster_sysctl_vars|no|see [lb_cluster_sysctl_vars](/defaults/main.yml)|sysctl variables to configure|
|lb_cluster_instances|no|[]|list of instances (see [cluster instances](#cluster-instances))
|lb_cluster_realservers|no|[]|list of realservers (see [realservers](#realservers))

### Cluster Instances

Each cluster instance must respect this schema:

|attribute|required|default|description|
|---------|--------|-------|-----------|
|name     |yes     |       |Name of the keepalived instance(`systemctl status keepalived@<name>`|
|interface|yes     |       |Interface where the vip will be placed|
|state    |no      |BACKUP |Startup state of the instance|
|virtual_router_id|yes|    |ID of the keepalived instance (must be unique in the cluster)|
|master_node|no    |       |The node with this name will be given priority higher than the computed ones|
|nopreempt|no      |       |If set will configure the keepalived instance as `nopreempt`|
|garp_master_refresh|no|   |If set, configure the corresponding keepalived parameter|
|advert_int|no     |       |If set, configure the corresponding keepalived parameter|
|authentication|no |       |If set keepalived will set the authentication password to `authentication.auth_pass`|
|virtual_ipaddress|yes|    |List of the VIPs managed by keepalived|
|virtual_ipaddress_excluded|no||List of the VIPs managed by keepalived without them being present in the vrrp packets|
|ip_variable|no    |ansible_host|Variable extracted from hostvars for each node to determine the peer address|
|virtual_servers|no|       |List of the virtual servers(see [Instance virtual servers](#instance-virtual-servers)|

#### Instance virtual servers

Each instance virtual server must respect this schema:

|attribute|required|default|description|
|---------|--------|-------|-----------|
|address  |yes     |       |Listen address|
|port     |yes     |       |Listen port|
|delay_loop|no     |       |If set, configure the corresponding keepalived parameter|
|lvs_sched|yes     |       |LVS balancing scheduler (`rr`, `mh`, ...)|
|lvs_method|yes     |       |LVS balancing method (`NAT`, `DR`, ...)|
|protocol |yes     |       |protocol   |
|sorry_server|no   |       |If set, configure the corresponding keepalived parameter|
|real_servers|yes  |       |List of the realservers (names) balanced by this instance|

### Realservers

Each realserver must respect this schema:

|attribute|required|default|description|
|---------|--------|-------|-----------|
|address  |yes     |       |address    |
|port     |yes     |       |port       |
|weight   |no      |       |If set, configure the corresponding keepalived parameter|
|inhibit_on_failure|no|    |If set and true, configure the corresponding keepalived parameter|
|checks   |no      |[]     |List of the checks to apply|

## Configuration example

```
lb_cluster_nodes: "{{groups.loadbalancers}}"

lb_cluster_instances:
-   name: 127.0.0.1
    interface: eth0
    virtual_router_id: 1
    authentication:
        auth_pass: password
    virtual_ipaddress:
    -   127.0.0.2/32
    peers: "{{groups.loadbalancer}}"
    virtual_servers:
    -   port: 80
        address: 127.0.0.2
        delay_loop: 6
        lvs_sched: mh
        lvs_method: DR
        protocol: TCP
        real_servers:
        -   myfe4-80
    -   port: 443
        address: 127.0.0.2
        #address: 10.1.32.10
        lvs_sched: mh
        lvs_method: DR
        protocol: TCP
        real_servers:
        -   myfe4-443

-   name: 127.0.0.3
    interface: eth0
    virtual_router_id: 2
    authentication:
        auth_pass: password
    virtual_ipaddress:
    -   127.0.0.3
    peers: "{{groups.loadbalancer}}"
    virtual_servers:
    -   port: 3306
        address: 127.0.0.3
        delay_loop: 6
        lvs_sched: mh
        lvs_method: DR
        protocol: TCP
        real_servers:
        -   mydb1-3306


lb_cluster_realservers
-   address: 127.0.0.1
    name: myfe4-80
    port: 80
    weight: 3
    checks:
    -   |
        TCP_CHECK {
            connect_timeout 3
            retry 3
            delay_before_retry 1
            connect_port 80
        }

-   address: 127.0.0.1
    name: myfe4-443
    port: 443
    weight: 3
    checks:
    -   |
        TCP_CHECK {
            connect_timeout 3
            retry 3
            delay_before_retry 1
            connect_port 443
        }

# this requires the `/root/bin/mysql-check.sh` to be present and usable
-   address: 127.0.0.1
    name: mydb1-3306
    port: 3306
    inhibit_on_failure: yes
    checks:
    -   |
        MISC_CHECK {
            misc_path "/root/bin/mysql-check.sh 10.1.32.14"
            misc_timeout 5
        }

```
