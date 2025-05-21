# Agent Based - Single NIC - Static IP

This is an example a common six node OpenShift deployment on baremetal. This uses the agent based installer to create an OpenShift cluster operating on a single NIC using a static IP.

## Scenario
The organization is installing an OpenShift cluster with one NIC per node. The node interface eno1 of each node has been patched into the top of rack switch and the switchports have been configured as access ports on VLAN 10. The organization has supplied VLAN 10 (192.168.10.0/24) as the OpenShift machine network and designated 192.168.10.10-15 as the node IP addresses. The gateway for the 192.168.10.0/24 subnet is 192.168.10.1 and the DNS provider is 192.168.0.2. DHCP is not available in this organization due to security requirements. A load balancer will be required to distribute API, API-INT, and application traffic to the OpenShift nodes. 192.168.0.3 has been designated for the HAProxy load balancer IP address.

![ocp-single-nic](https://github.com/dlystra/openshift-networking-examples/blob/main/diagrams/ocp-single-nic.png)

## Variables

| Variable Name      | Type    | Example            | Description                             |
|--------------------|---------|--------------------|-----------------------------------------|
| `cluster-name`     | string  | `ocp-cluster`      | Desired name of OCP cluster             |
| `dns-ip`           | string  | `192.168.0.2`      | DNS server IP address                   |
| `domain`           | string  | `example.com`      | Domain name of OCP cluster              |
| `gwy-ip`           | string  | `192.168.10.1`     | Physical subnet gateway IP address      |
| `machine-subnet`   | string  | `192.168.10.0/24`  | Physical subnet for OCP nodes           |
| `node-int`         | string  | `eno1`             | Node interface name                     |
| `master-0-ip`      | string  | `192.168.10.10`    | Desired IP of master-0 node             |
| `master-0-mac`     | string  | `00:25:64:fd:1f:ac`| Node interface MAC address              |
| `master-0-ip`      | string  | `192.168.10.11`    | Desired IP of master-0 node             |
| `master-1-mac`     | string  | `00:25:64:fd:1f:ad`| Node interface MAC address              |
| `master-2-ip`      | string  | `192.168.10.12`    | Desired IP of master-0 node             |
| `master-2-mac`     | string  | `00:25:64:fd:1f:ae`| Node interface MAC address              |
| `public-ssh-key`   | string  |                    | Public SSH key for SSH access to node   |
| `pull-secret`      | string  |                    | OCP pull secret                         |
| `worker-0-ip`      | string  | `192.168.10.13`    | Desired IP of master-0 node             |
| `worker-0-mac`     | string  | `00:25:64:fd:1f:af`| Node interface MAC address              |
| `worker-1-ip`      | string  | `192.168.10.14`    | Desired IP of master-0 node             |
| `worker-1-mac`     | string  | `00:25:64:fd:1f:b0`| Node interface MAC address              |
| `worker-2-ip`      | string  | `192.168.10.15`    | Desired IP of master-0 node             |
| `worker-2-mac`     | string  | `00:25:64:fd:1f:b1`| Node interface MAC address              |

## Networking Requirements

- Switchport
  - The switch's ports is configured as an access port

- DHCP
  - None

- Routing
  - 192.168.10.0/24 can route to the internet (or local registry) via 192.168.10.1
  - 192.168.10.0/24 can route to the DNS provider (192.168.0.2)
  - The load balancer can route to 192.168.10.0/24

- DNS
  - api.ocp-cluster.example.com resolves to the load balancer IP (192.168.0.3)
  - api-int.ocp-cluster.example.com resolves to the load balancer IP (192.168.0.3)
  - *.apps.ocp-cluster.example.com resolves to the load balancer IP (192.168.0.3)
  - master-0.ocp-cluster.example.com resolves to 192.168.10.10
  - master-1.ocp-cluster.example.com resolves to 192.168.10.11
  - master-2.ocp-cluster.example.com resolves to 192.168.10.12
  - worker-0.ocp-cluster.example.com resolves to 192.168.10.13
  - worker-1.ocp-cluster.example.com resolves to 192.168.10.14
  - worker-2.ocp-cluster.example.com resolves to 192.168.10.15

- Loadbalancing

| Service       | Port | Mode | nodes                        |
|---------------|------|------|------------------------------|
| API           | 6443 | TCP  | master-0, master-1, master-2 |
| API (internal)| 22623| TCP  | master-0, master-1, master-2 |
| HTTPS Ingress | 443  | TCP  | worker-0, worker-1, worker-2 |
| HTTP Ingress  | 80   | TCP  | worker-0, worker-1, worker-2 |

## Populated Examples

**'install-config.yaml'**
```yaml
apiVersion: v1
baseDomain: example.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp-cluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.10.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{{ pull-secret }}'
sshKey: '{{ public-ssh-key }}'
```

**'agent-config.yaml'**
```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: ocp-cluster
rendezvousIP: 192.168.10.10
hosts:
  - hostname: master-0
    role: master
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: eno1
        macAddress: 00:25:64:fd:1f:ac
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:ac
          ipv4:
            enabled: true
            address:
              - ip: 192.168.10.10
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - 192.168.0.2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.10.1
            next-hop-interface: eno1
            table-id: 254
  - hostname: master-1
    role: master
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: eno1
        macAddress: 00:25:64:fd:1f:ad
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:ad
          ipv4:
            enabled: true
            address:
              - ip: 192.168.10.11
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - 192.168.0.2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.10.1
            next-hop-interface: eno1
            table-id: 254
  - hostname: master-2
    role: master
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: eno1
        macAddress: 00:25:64:fd:1f:ae
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:ae
          ipv4:
            enabled: true
            address:
              - ip: 192.168.10.12
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - 192.168.0.2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.10.1
            next-hop-interface: eno1
            table-id: 254
  - hostname: worker-0
    role: worker
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: eno1
        macAddress: 00:25:64:fd:1f:af
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:af
          ipv4:
            enabled: true
            address:
              - ip: 192.168.10.13
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - 192.168.0.2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.10.1
            next-hop-interface: eno1
            table-id: 254
  - hostname: worker-1
    role: worker
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: eno1
        macAddress: 00:25:64:fd:1f:b0
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:b0
          ipv4:
            enabled: true
            address:
              - ip: 192.168.10.14
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - 192.168.0.2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.10.1
            next-hop-interface: eno1
            table-id: 254
  - hostname: worker-2
    role: worker
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: eno1
        macAddress: 00:25:64:fd:1f:b1
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:b1
          ipv4:
            enabled: true
            address:
              - ip: 192.168.10.15
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - 192.168.0.2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.10.1
            next-hop-interface: eno1
            table-id: 254
```
**'haproxy.cfg'**
```yaml
global
  log         127.0.0.1 local2
  pidfile     /var/run/haproxy.pid
  maxconn     4000
  daemon
defaults
  mode                    http
  log                     global
  option                  dontlognull
  option http-server-close
  option                  redispatch
  retries                 3
  timeout http-request    10s
  timeout queue           1m
  timeout connect         10s
  timeout client          1m
  timeout server          1m
  timeout http-keep-alive 10s
  timeout check           10s
  maxconn                 3000
listen api-server-6443 
  bind *:6443
  mode tcp
  server master-0 master-0.ocp-cluster.example.com:6443 check inter 1s
  server master-1 master-1.ocp-cluster.example.com:6443 check inter 1s
  server master-2 master-2.ocp-cluster.example.com:6443 check inter 1s
listen machine-config-server-22623 
  bind *:22623
  mode tcp
  server master-0 master-0.ocp-cluster.example.com:22623 check inter 1s
  server master-1 master-1.ocp-cluster.example.com:22623 check inter 1s
  server master-2 master-2.ocp-cluster.example.com:22623 check inter 1s
listen ingress-router-443 
  bind *:443
  mode tcp
  balance source
  server worker-0 worker-0.ocp-cluster.example.com:443 check inter 1s
  server worker-1 worker-1.ocp-cluster.example.com:443 check inter 1s
  server worker-2 worker-2.ocp-cluster.example.com:443 check inter 1s
listen ingress-router-80 
  bind *:80
  mode tcp
  balance source
  server worker-0 worker-0.ocp-cluster.example.com:80 check inter 1s
  server worker-1 worker-1.ocp-cluster.example.com:80 check inter 1s
  server worker-2 worker-2.ocp-cluster.example.com:80 check inter 1s
```