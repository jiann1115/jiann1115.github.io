---
layout: post
title:  "Openstack lbaasv2 agent"
date:   2018-10-22 18:00:40 -0700
categories: networking
---



# OPENSTACK LBAASv2 Agent

# 1. Architecture

Reference:

[https://wiki.openstack.org/wiki/Neutron/LBaaS/Agent](https://wiki.openstack.org/wiki/Neutron/LBaaS/Agent)

[https://wiki.openstack.org/wiki/Neutron/LBaaS/Architecture](https://wiki.openstack.org/wiki/Neutron/LBaaS/Architecture)

## 1.1 Plugin part

### 1.1.1 Calling agent

Plugin packs all required information in the json message and sends it over AMQP to durable message queue. The message is consumed by one of the running agents and corresponding call is made via one of
the drivers in synchronous manner. After driver call completes agent sends message to the plugin with status information for modified object.

### 1.1.2 Receiving response 

Plugin should wait on another queue where responses are posted by the agent. Information in response should allow plugin to uniquely identify object for which status of operation is returned. Plugin consumes response messages in synchronous manner one by one since their processing is not a time consuming operation

## 1.2 Agent part

Agent should be able to consume messages in multithreaded manner. E.g. agent should allow execution of several operations at once. One technical difficulty of this is device sharing. Consider the case when several operations go for the same device. Such operations should be executed sequentially rather than concurrently.

## 1.3 Database (DB)

Save all object information. Save relation between agent and loadbalancer. MySQL is applied in the project.

## 1.4 Message Queue (MQ)

Message queue is for request and response message using oslo messaging library. It provides rpc client-server model. Abstract of AMQP/non-AMQP implementation, ex: rabbitMQ





## 1.5 Modules Decomposition

- LBaaS Quantum Extension - is responsible for handling REST API calls
- LBaaS Quantum AdvSvc Plugin - is a core of service, it is responsible for:
  - DB storage
  - Request validation
  - Scheduling of load balancing services (deployment to LB devices)
- LBaaS Agent - is stand-alone service that manages drivers
- Driver - is a module that transforms LBaaS object model into vendor-specific model and deploys configuration onto load-balancing device 

```
[Configuration]
/etc/neutron/neutron.conf
service_plugins = neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2

```

![1540200124452](/images/openstack/1540200124452.png)

```
# PATH = /opt/stack/neutron-lbaas/neutron-lbaas/
# Call plugin driver in the file:
$PATH/neutron_lbaas/services/loadbalancer/plugin.py
# DB's source code:
$PATH/services/loadbalancer/plugin.py:79:      self.db = ldbv2.LoadBalancerPluginDbv2()
$PATH/db/loadbalancer/loadbalancer_dbv2.py:50:class LoadBalancerPluginDbv2(bas...

```



## 1.6    Workflow

 LBaaS management is performed via REST API. If the call involves DB only (read, name update) then it is done synchronous way, all other calls (like status retrieval, write operations) are done asynchronously.

There are two types of locks:

- Object-level lock is done on an instance (vip, pool, member) and restricts concurrent changes. The lock is achieved by moving object  into one of PENDING_* states.
- Device-level lock is done to lock the whole configuration and disallow concurrent changes. Since this is a restrictive policy, requests are actually put into queue and then processed by driver one-by-one. The      queue may be implemented completely as Python code or be an external MQ.

![1540200558346](/images/openstack/1540200558346.png)

Ordinary update workflow is:

1.	REST API request is accepted by Quantum and routed to the corresponding Extension and Plugin.
2.	Plugin performs validation of request (schema conformance, values and references check, etc). If validation fails one of 40x codes is returned (depending on reason).
3.	DB object is updated and object is moved to PENDING_UPDATE state.
4.	Request is transformed into task and pushed into queue.
5.	Plugin responses user with HTTP 202 reply. Steps 1-5 are done synchronously.
6.	Agent picks message from the queue and forwards it to Driver. Driver changes configuration of load balancing device. Once completed the response message is pushed into Plugin's queue.
7.	Plugin retrieves message and updates DB with either "ACTIVE" or "ERROR" status

Data model should be resistant to different failures and crash of modules:

- If crash in 1), 2) or 3) all changes are lost, HTTP 500 error is returned to user.

- If crash in 4) or 5) then Plugin handles it and moves object into ERROR state, HTTP 500 error is returned.

- If crash in 6) or 7) the object remains in "PENDING_" state. The Plugin will move the object into ERROR state after some preconfigured time or after restart.


## 2. OPENSTACK LBAASv2 Agent Setup

### 2.1 Install Linux

#### 2.1.1 Environment

The DevStack is running in this environment.

```
Distributor ID:    Ubuntu
Description:    Ubuntu 16.04 LTS
Release:    16.04
Codename:    xenial
=====================================================================
Disk: 2T
Memory: 128G
```

The DevStack is running in this environment.

Please download the ubuntu minimal iso. [http://archive.ubuntu.com/ubuntu/dists/xenial/main/installer-amd64/current/images/netboot/mini.iso]

#### 2.1.2. DevStack

Follow the instruction under "add stack user" and "install Devstack"

1.	https://docs.openstack.org/devstack/latest/#add-stack-user
2.	https://docs.openstack.org/devstack/latest/#download-devstack

#### 2.1.2.1 Prepare local.conf

Place local.conf to /opt/stack/devstack/
$ cp local.conf  /opt/stack/devstack/

Local.conf is shown below.
Note: please adjust VOLUME_BACKING_FILE_SIZE based on deployed scenario and physical source on your host. Recommended minimal value is 150G. VOLUME_BACKING_FILE_SIZE will require double space on disk. one for default, one for backup.

```
[[local|localrc]]
enable_plugin neutron-lbaas https://git.openstack.org/openstack/neutron-lbaas stable/pike
enable_plugin neutron-lbaas-dashboard https://git.openstack.org/openstack/neutron-lbaas-dashboard stable/pike

ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# Logging
LOGFILE=$DEST/logs/stack.sh.log
LOGDAYS=2

# Nova
enable_service n-novnc n-cauth
ENABLED_SERVICES+=,n-api,n-cpu,n-cond,n-sch
ENABLED_SERVICES+=,n-cell

# Neutron
disable_service n-net
ENABLED_SERVICES+=,q-svc,q-agt,q-dhcp,q-l3,q-meta
ENABLED_SERVICES+=,q-lbaasv2,q-vpn,q-fwaas


# Tempest
ENABLED_SERVICES+=,tempest

VIF_PLUGGING_IS_FATAL=False
VIF_PLUGGING_TIMEOUT=0

# Cinder
VOLUME_BACKING_FILE_SIZE=768G
```

#### 2.1.3. Change git branch to stable/pike, ex:

```
$ cd devstack
$ git checkout stable/pike
```

Change local.conf branch, ex:

```
enable_plugin neutron-lbaas https://git.openstack.org/openstack/neutron-lbaas stable/pike
enable_plugin neutron-lbaas-dashboard https://git.openstack.org/openstack/neutron-lbaas-dashboard stable/pike
```




#### 2.1.4. Install DevStack

```
$./stack.sh
```

If something goes wrong, after fixed it, please unstack before redo stack.sh

```
$ ./unstack.sh
```

If you want to clean all data, includes data in MySQL.

```
$ ./clean.sh
```

If everything is ok, it shows

```
=========================
DevStack Component Timing
=========================
Total runtime         896

run_process            94
test_with_retry         3
apt-get-update          3
pip_install            96
restart_apache_server   9
wait_for_service       12
git_timed              96
apt-get                23
=========================

This is your host IP address: 192.168.133.99
This is your host IPv6 address: fe80::20c:29ff:fe2f:d769
Horizon is now available at http://192.168.133.99/dashboard
Keystone is serving at http://192.168.133.99/identity/
The default users are: admin and demo
The password: secret
```

By far, nova, neutron and all services specified in local.conf are running now.	
To check service(unit) running log, use journalctl.
https://docs.openstack.org/devstack/latest/systemd.html#journalctl-examples
To restart/stop/start service(unit), use systemctl
https://docs.openstack.org/devstack/latest/systemd.html#manipulating-units

**service (unit)**

| service (unit)                   |                               |
| -------------------------------- | ----------------------------- |
| n-novnc                          | vnc console for nova instance |
| n-api                            | Nova apis                     |
| q-svc                            | Neutron controller            |
| q-dhcp                           | Neutron dhcp server           |
| q-lbaasv2(option, not necessary) | Neutron lbaas v2 service      |
| q-l3                             | Neutron l3 network service    |

**All service list using restart:**

sudo systemctl restart devstack@c-api.service

sudo systemctl restart devstack@c-sch.service

sudo systemctl restart devstack@c-vol.service

sudo systemctl restart devstack@dstat.service

sudo systemctl restart devstack@etcd.service

sudo systemctl restart devstack@g-api.service

sudo systemctl restart devstack@g-reg.service

sudo systemctl restart devstack@keystone.service

sudo systemctl restart devstack@n-api-meta.service

sudo systemctl restart devstack@n-api.service

sudo systemctl restart devstack@n-cauth.service

sudo systemctl restart devstack@n-cell-child.service

sudo systemctl restart devstack@n-cell-region.service

sudo systemctl restart devstack@n-cond.service

sudo systemctl restart devstack@n-cpu.service

sudo systemctl restart devstack@n-novnc.service

sudo systemctl restart devstack@n-sch.service

sudo systemctl restart devstack@placement-api.service

sudo systemctl restart devstack@q-agt.service

sudo systemctl restart devstack@q-dhcp.service

sudo systemctl restart devstack@q-l3.service

sudo systemctl restart devstack@q-lbaasv2.service  (option, not necessary)

sudo systemctl restart devstack@q-meta.service

sudo systemctl restart devstack@q-svc.service



#### 2.1.5. Check Cinder Volume

check if cinder volume is successfully created

```
$ ls /opt/stack/data/stack-volumes-lvmdriver-1-backing-file
```

```
$sudo vgs
  VG                        #PV #LV #SN Attr   VSize   VFree 
  openstack-11-vg             1   2   0 wz--n-   1.82t     0 
  stack-volumes-default       1   0   0 wz--n-  10.01g 10.01g
  stack-volumes-lvmdriver-1   1  16   0 wz--n- 768.00g 38.21g
$sudo pvs
  PV         VG                        Fmt  Attr PSize   PFree 
  /dev/loop0 stack-volumes-default     lvm2 a--   10.01g 10.01g
  /dev/loop1 stack-volumes-lvmdriver-1 lvm2 a--  768.00g 38.21g
  /dev/sda5  openstack-11-vg           lvm2 a--    1.82t     0
```

if stack-volumes-lvmdriver-1 is not initialized, follow below instructions. change bs to VOLUME_BACKING_FILE_SIZE

```
cd /opt/stack/data/
dd if=/dev/zero of=stack-volumes-lvmdriver-1-backing-file bs=768 count=1G
```

if volume group is not there, create it.

```
sudo /sbin/pvcreate /dev/loop1
sudo /sbin/vgcreate "stack-volumes-lvmdriver-1" /dev/loop1
```

check volume group

```
$sudo losetup 
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE
/dev/loop0         0      0         0  0 /opt/stack/data/stack-volumes-default-backing-file
/dev/loop1         0      0         0  0 /opt/stack/data/stack-volumes-lvmdriver-1-backing-file
```

if there is no vg, do the follow commands

```
sudo losetup /dev/loop1 /opt/stack/data/stack-volumes-lvmdriver-1-backing-file
sudo losetup /dev/loop0 /opt/stack/data/stack-volumes-default-backing-file
```

disable image cache

```
vi /etc/cinder/cinder.conf
image_volume_cache_enabled = False
```

restart cinder service

```
sudo systemctl restart devstack@c-*
```

.....................................

.....................................

.....................................

.....................................

.....................................

## 5. Code Description

### 5.1 agent.py

Create and launch agent service

### 5.2 agent_manage.py

RPC method from openstack plugin:  Openstack CMD to device though this

| **Method**            | **Description**                                              |
| --------------------- | ------------------------------------------------------------ |
| create_loadbalancer   | Create a loadbalancer<br />    Input:<br />    1. loadbalancer dictionary |
| update_loadbalancer   | Update a loadbalancer    Input:<br />    1. old loadbalancer dictionary<br />    2. new loadbalancer dictionary |
| delete_loadbalancer   | Delete a loadbalancer    Input:<br />    1. listener dictionary |
| create_listener       | Create a listener Input:<br />    1. listener dictionary     |
| update_listener       | Update a listener Input:<br />    1. old listener dictionary<br />    2. new listener dictionary |
| delete_listener       | Delete a listener Input:<br />    1. listener dictionary     |
| create_pool           | Create a pool Input:<br />    1. pool dictionary             |
| update_pool           | Update a pool Input:<br />    1. old pool dictionary<br />    2. new pool dictionary |
| delete_pool           | Delete a pool Input:<br />    1. pool dictionary             |
| create_member         | Create a member Input:<br />    1. member dictionary         |
| update_member         | Update a member Input:<br />    1. old member dictionary<br />    2. new member dictionary |
| delete_member         | Delete a member Input:<br />    1. member dictionary         |
| create_health_monitor | Create a health monitor Input:<br />    1. health_monitor dictionary |
| update_health_monitor | Update a health monitor Input:<br />    1. old health_monitor dictionary<br />    2. new health_monitor dictionary |
| delete_health_monitor | Deletea health monitor Input:<br />    1. health_monitor dictionary |

> Every update of a LoadBalancer, including updates to and adding/removing children entities, will put the LoadBalancer into a PENDING_UPDATE provisioning_status. No other updates to that LoadBalancer or its children will be allowed until the operation has completed.

The method of maintain:  sync openstack with device

| **Method**                | **Description**                                              |
| :------------------------ | :----------------------------------------------------------- |
| _setup_state_rpc()        | Setup   the state of rpc                                     |
| _report_state()           | Report   the state in period                                 |
| initialize_service_hook() | When service   initializing, it call back initialize_service_hook for:   1. build consumer   topic.host   2. sync_state() |
| sync_state()              | Sync state when the agent start or periodic_resync().   Compare   openstack loadbalancer with device   driver<br />   1. Delete opnestack redundant loadbalancer<br />   2. Reload device   driver loadbalancer<br />   3. Delete device   driver redundant loadbalancer |
| periodic_resync()         | Check to do sync_state() in period. If needed, needs_resync is set True. |
| collect_stats()           | Collect statistics from device driver in period.             |
| remove_orphans ()         | Remove any orphaned services in the device                   |

### 5.3 plugin_rpc.py

**RPC with openstack plugin**:

| **Method**                | **Description**                                              |
| :------------------------ | ------------------------------------------------------------ |
| get_ready_devices         | Get the ready loadbalancer id list from   DB                 |
| get_loadbalancer          | Get the current loadbalancer data from DB                    |
| update_status             | Update the provisioning_status and operating_status   for loadbalancer, listener, pool, member, and health_monitor |
| update_loadbalancer_stats | Update the statistics of load balancer:   bytes_in, bytes_out, active_connections, total_connections |
| loadbalancer_destroyed    | Delete loadbalancer                                          |
| listener_destroyed        | Delete listener                                              |
| pool_destroyed            | Delete pool                                                  |
| member_destroyed          | Delete member                                                |
| health_monitor_destroyed  | Delete health monitor                                        |

### 5.4.	device_driver

| **Method**             | **Description**                                              |
| ---------------------- | ------------------------------------------------------------ |
| loadbalancer.create    | Prototype   Input:    1. loadbalancer data model             |
| loadbalancer.update    | Update a loadbalancer   Input:    1. old loadbalancer data model    2. new loadbalancer data model |
| loadbalancer.delete    | Delete a loadbalancer    Input:    1. loadbalancer data model |
| loadbalancer.get_stats | Get the statistics from device   Input:    1. loadbalancer data model   Output:    1. stat dictionary |
| listener.create        | Prototype   Input:    1. listener data model                 |
| listener.update        | Update a listener   Input:    1. old listener data model    2. new listener data model |
| listener.delete        | Delete a listener   Input:    1. listener data model         |
| pool.create            | Create a pool and its loadbalancer and   listener   Input:    1. pool data model |
| pool.update            | Update a pool   Input:    1. old pool data model    2. new pool data model |
| pool.delete            | Delete a pool   Input:    1. pool data model                 |
| member.create          | Create a member   Input:    1. member data model             |
| member.updae           | Update a member   Input:    1. old member data model    2. new member data model |
| member.delete          | Delete a member   Input:    1. loadbalancer data model       |
| health_monitor.create  | Create a health monitor   Input:    1. health_monitor data model |
| health_monitor.update  | Update a health monitor   Input:    1. old health_monitor data model    2. new health_monitor data model |
| health_monitor.delete  | Deletea health monitor   Input:    1. health_monitor data model |

.....................................

.....................................

.....................................

.....................................

.....................................

## 6. The LBaaS V2 Data Model

https://developer.openstack.org/api-ref/load-balancer/v2/
https://specs.openstack.org/openstack/heat-specs/specs/mitaka/lbaasv2-support.html

### 6.1 LoadBalancer

 LBaaS V2 LoadBalancer, which creates a LoadBalancer along with the VIP.

- id - unique identifier
- tenant_id
- name
- description
- vip_port_id - the neutron port tied to the vip (this can be      used to get the vip_subnet, and vip_address). This will be stored in the      database but not exposed through the API.
- vip_subnet_id - the subnet a neutron port should be created on
- vip_address - the address of the subnet
- provisioning_status
- operating_status
- admin_state_up
- listeners - a list of back-references child listener ids
- provider - provider in which this load balancer should be      provisioned

**Operating Status Codes**

| **Code**    | **Description**                                              |
| ----------- | ------------------------------------------------------------ |
| **ONLINE**  | **Entity is operating normally**   **All pool members are healthy** |
| DRAINING    | The member is   not accepting new connections                |
| **OFFLINE** | **Entity is administratively disabled**                      |
| DEGRADED    | One or more of   the entity’s components are in ERROR        |
| ERROR       | The entity has   failed   The member is   failing it’s health monitoring checks   All of the   pool members are in ERROR |
| NO_MONITOR  | No health   monitor is configured for this entity and it’s status is unknown |

**Provisioning Status Codes**

| **Code**           | **Description**                             |
| ------------------ | ------------------------------------------- |
| **ACTIVE**         | **The entity was provisioned successfully** |
| DELETED            | The entity has   been successfully deleted  |
| **ERROR**          | **Provisioning failed**                     |
| **PENDING_CREATE** | **The entity is being created**             |
| **PENDING_UPDATE** | **The entity is being updated**             |
| **PENDING_DELETE** | **The entity is being deleted**             |

> Entities in a PENDING_\* state are immutable and cannot be modified until the requested operation completes. The entity will return to the ACTIVE provisioning status once the asynchronus operation completes.*
>
> An entity in ERROR has failed provisioning. The entity may be deleted and recreated.*

### 6.2.	Listener

 LBaaS V2 Listener, which creates a Listener associated with the LoadBalancer for a particular port and protocol.

* protocol - Protocol to load balancer: HTTP, HTTPS, TCP, UDP
* protocol_port - port number to listen

### 6.3.	Pool

 LBaaS V2 Pool, which creates a Pool associated with the Listener.

* lb_algorithm - Load balancing algorithm to be used. String Value. Allowed Values - ROUND_ROBIN, LEAST_CONNECTIONS, SOURCE_IP

* protocol - Protocol of the Pool. String Value. Allowed Values - TCP, HTTP, HTTPS

### 6.4.	PoolMember

 Backend servers to be added to the Load balancing pool.

* subnet - Subnet ID or name for the member. String Value. Must be of type neutron.subnet

### 6.5.	HealthMonitor

 LBaaS V2 HealthMonitor, creates a health monitor for the Pool.

* delay - The minimum time in milliseconds between regular connections of member. Integer Value. Update allowed.
* type - One of predefined health monitor types.. String Value. Allowed values - PING, TCP, HTTP, HTTPS
* max_retries - Number of permissible connection failures before changing the member status to INACTIVE. Integer Value. Update allowed. Must be in the range of 1 to 10.
* timeout - Maximum number of milliseconds for a monitor to wait for a connection to be established before it times out.

.....................................

.....................................

.....................................

.....................................

.....................................

## 8. Network of openstack

### 8.1 Network Topology

![1540265416278](/images/openstack/1540265416278.png)

### 8.2 Network connectivity Architecture

![1540265507765](/images/openstack/1540265507765.png)

### 8.3 iptables NAT

```
$ ip netns exec qrouter-12e8831d-b459-42fc-8f35-9b17aef1335c iptables -t nat –S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N neutron-l3-agent-OUTPUT
-N neutron-l3-agent-POSTROUTING
-N neutron-l3-agent-PREROUTING
-N neutron-l3-agent-float-snat
-N neutron-l3-agent-snat
-N neutron-postrouting-bottom
-A PREROUTING -j neutron-l3-agent-PREROUTING
-A OUTPUT -j neutron-l3-agent-OUTPUT
-A POSTROUTING -j neutron-l3-agent-POSTROUTING
-A POSTROUTING -j neutron-postrouting-bottom
-A neutron-l3-agent-OUTPUT -d 172.24.4.3/32 -j DNAT --to-destination 10.0.0.10
-A neutron-l3-agent-OUTPUT -d 172.24.4.7/32 -j DNAT --to-destination 10.0.0.7
-A neutron-l3-agent-POSTROUTING ! -i qg-efc4d091-96 ! -o qg-efc4d091-96 -m conntrack ! --ctstate DNAT -j ACCEPT
-A neutron-l3-agent-PREROUTING -d 169.254.169.254/32 -i qr-+ -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 9697
-A neutron-l3-agent-PREROUTING -d 172.24.4.3/32 -j DNAT --to-destination 10.0.0.10
-A neutron-l3-agent-PREROUTING -d 172.24.4.7/32 -j DNAT --to-destination 10.0.0.7
-A neutron-l3-agent-float-snat -s 10.0.0.10/32 -j SNAT --to-source 172.24.4.3
-A neutron-l3-agent-float-snat -s 10.0.0.7/32 -j SNAT --to-source 172.24.4.7
-A neutron-l3-agent-snat -j neutron-l3-agent-float-snat
-A neutron-l3-agent-snat -o qg-efc4d091-96 -j SNAT --to-source 172.24.4.11
-A neutron-l3-agent-snat -m mark ! --mark 0x2/0xffff -m conntrack --ctstate DNAT -j SNAT --to-source 172.24.4.11
-A neutron-postrouting-bottom -m comment --comment "Perform source NAT on outgoing traffic." -j neutron-l3-agent-snat 
```

### 8.4 Reference

* [https://docs.openstack.org/liberty/networking-guide/scenario-classic-ovs.html](https://www.rdoproject.org/networking/networking-in-too-much-detail/)

* <https://www.rdoproject.org/networking/networking-in-too-much-detail/> 

* <http://www.yet.org/2014/09/openvswitch-troubleshooting/> 