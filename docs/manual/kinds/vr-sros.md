---
search:
  boost: 4
---
# Nokia SR OS

[Nokia SR OS](https://www.nokia.com/networks/products/service-router-operating-system/) virtualized router is identified with `vr-sros` or `vr-nokia_sros` kind in the [topology file](../topo-def-file.md). It is built using [vrnetlab](../vrnetlab.md) project and essentially is a Qemu VM packaged in a docker container format.

vr-sros nodes launched with containerlab come up pre-provisioned with SSH, SNMP, NETCONF and gNMI services enabled.

## Managing vr-sros nodes

!!!note
    Containers with SR OS inside will take ~3min to fully boot.  
    You can monitor the progress with `watch docker ps` waiting till the status will change to `healthy`.

Nokia SR OS node launched with containerlab can be managed via the following interfaces:

=== "bash"
    to connect to a `bash` shell of a running vr-sros container:
    ```bash
    docker exec -it <container-name/id> bash
    ```
=== "CLI"
    to connect to the SR OS CLI
    ```bash
    ssh admin@<container-name/id>
    ```
=== "NETCONF"
    NETCONF server is running over port 830
    ```bash
    ssh root@<container-name> -p 830 -s netconf
    ```
=== "gNMI"
    using the best in class [gnmic](https://gnmic.kmrd.dev) gNMI client as an example:
    ```bash
    gnmic -a <container-name/node-mgmt-address> --insecure \
    -u admin -p admin \
    capabilities
    ```
=== "Telnet"
    serial port (console) is exposed over TCP port 5000:
    ```bash
    # from container host
    telnet <node-name> 5000
    ```  
    You can also connect to the container and use `telnet localhost 5000` if telnet is not available on your container host.

!!!info
    Default user credentials: `admin:admin`

## Interfaces mapping

vr-sros container uses the following mapping for its interfaces:

* `eth0` - management interface connected to the containerlab management network
* `eth1` - first data interface, mapped to the first data port of SR OS line card
* `eth2+` - second and subsequent data interface

Interfaces can be defined in a non-sequential way, for example:

```yaml
  links:
    # sr1 port 3 is connected to sr2 port 5
    - endpoints: ["sr1:eth3", "sr2:eth5"]
```

When containerlab launches vr-sros node, it will assign IPv4/6 address to the `eth0` interface. These addresses can be used to reach management plane of the router.

Data interfaces `eth1+` need to be configured with IP addressing manually using CLI/management protocols.

## Features and options

### Variants

Virtual SR OS simulator can be run in multiple HW variants as explained in [the vSIM installation guide](https://documentation.nokia.com/cgi-bin/dbaccessfilename.cgi/3HE15836AAADTQZZA01_V1_vSIM%20Installation%20and%20Setup%20Guide%2020.10.R1.pdf).

`vr-sros` container images come with [pre-packaged SR OS variants](https://github.com/hellt/vrnetlab/tree/master/sros#variants) as defined in the upstream repo as well as support [custom variant definition](https://github.com/hellt/vrnetlab/tree/master/sros#custom-variant). The pre-packaged variants are identified by the variant name and come up with cards and mda already configured. On the other hand, custom variants give users total flexibility in emulated hardware configuration, but cards and MDAs must be configured manually.

To make vr-sros to boot in one of the packaged variants, set the type to one of the predefined variant values:

```yaml
topology:
  nodes:
    sros:
      kind: vr-sros
      image: vrnetlab/vr-sros:20.10.R1
      type: sr-1s # if omitted, the default sr-1 variant will be used
      license: license-sros20.txt
```

#### Custom variants

A custom variant can be defined by specifying the *TIMOS line* for the control plane and line card components:

```yaml
type: >- # (1)!
  cp: cpu=2 ram=4 chassis=ixr-e slot=A card=cpm-ixr-e ___
  lc: cpu=2 ram=4 max_nics=34 chassis=ixr-e slot=1 card=imm24-sfp++8-sfp28+2-qsfp28 mda/1=m24-sfp++8-sfp28+2-qsfp28
```

1. for distributed chassis CPM and IOM are indicated with markers `cp:` and `lc:`.

    notice the delimiter string `___` that **must** be present between CPM and IOM portions of a custom variant string.

    `max_nics` value **must** be set in the `lc` part and specifies a maximum number of network interfaces this card will be equipped with.

    Memory `mem` is provided in GB.

It is possible to define a custom variant with multiple line cards; just repeat the `lc` portion of a type. Note that each line card is a separate VM, increasing pressure on the host running such a node. You may see some issues starting multi line card nodes due to the VMs being moved between CPU cores unless a [cpu-set](../nodes.md#cpu-set) is used.

```yaml title="distributed chassis with multiple line cards"
type: >-
  cp: cpu=2 min_ram=4 chassis=sr-7 slot=A card=cpm5 ___
  lc: cpu=4 min_ram=4 max_nics=6 chassis=sr-7 slot=1 card=iom4-e mda/1=me6-10gb-sfp+ ___
  lc: cpu=4 min_ram=4 max_nics=6 chassis=sr-7 slot=2 card=iom4-e mda/1=me6-10gb-sfp+
```

???tip "How to define links in a multi line card setup?"
    When a node uses multiple line cards users should pay special attention to the way links are defined in the topology file. As explained in the [interface mapping](#interfaces-mapping) section, SR OS nodes use `ethX` notation for their interfaces, where `X` denotes a port number on a line card/MDA.

    Things get a little more tricky when multiple line cards are provided. First, every line card must be defined with a `max_nics` property that serves a simple purpose - identify how many ports at maximum this line card can bear. In the example above both line cards are equipped with the same IOM/MDA and can bear 6 ports at max. Thus, `max_nics` is set to 6.

    Another significant value of a line card definition is the `slot` position. Line cards are inserted into slots, and slot 1 comes before slot 2, and so on.

    Knowing the slot number and the maximum number of ports a line card has, users can identify which indexes they need to use in the `link` portion of a topology to address the right port of a chassis. Let's use the following example topology to explain how this all maps together:

    ```yaml
    topology:
      nodes:
        R1:
          kind: vr-sros
          image: vr-sros:22.7.R2
          type: >-
            cp: cpu=2 min_ram=4 chassis=sr-7 slot=A card=cpm5 ___
            lc: cpu=4 min_ram=4 max_nics=6 chassis=sr-7 slot=1 card=iom4-e mda/1=me6-10gb-sfp+ ___
            lc: cpu=4 min_ram=4 max_nics=6 chassis=sr-7 slot=2 card=iom4-e mda/1=me6-10gb-sfp+
        R2:
          kind: vr-sros
          image: sros:22.7.R2
          type: >-
            cp: cpu=2 min_ram=4 chassis=sr-7 slot=A card=cpm5 ___
            lc: cpu=4 min_ram=4 max_nics=6 chassis=sr-7 slot=1 card=iom4-e mda/1=me6-10gb-sfp+ ___
            lc: cpu=4 min_ram=4 max_nics=6 chassis=sr-7 slot=2 card=iom4-e mda/1=me6-10gb-sfp+

      links:
      - endpoints: ["R1:eth1", "R2:eth3"]
      - endpoints: ["R1:eth7", "R2:eth8"]
    ```

    Starting with the first pair of endpoints `R1:eth1 <--> eth3:R2`; we see that port1 of R1 is connected with port3 of R2. Looking at the slot information and `max_nics` value of 6 we see that the linecard in slot 1 can host maximum 6 ports. This means that ports from 1 till 6 belong to the line card equipped in slot=1. Consequently, links ranging from `eth1` to `eth6` will address the ports of that line card.

    The second pair of endpoints `R1:eth7 <--> eth8:R2` addresses the ports on a line card equipped in the slot 2. This is driven by the fact that the first six interfaces belong to line card in slot 1 as we just found out. This means that our second line card that sits in slot 2 and has as well six ports, will be addressed by the interfaces `eth7` till `eth12`, where `eth7` is port1 and `eth12` is port6.

An integrated variant is provided with a simple TIMOS line:

```yaml
type: "cpu=2 ram=4 slot=A chassis=ixr-r6 card=cpiom-ixr-r6 mda/1=m6-10g-sfp++4-25g-sfp28" # (1)!
```

1. No `cp` nor `lc` markers are needed to define an integrated variant.

### Node configuration

vr-sros nodes come up with a basic "blank" configuration where only the card/mda are provisioned, as well as the management interfaces such as Netconf, SNMP, gNMI.

#### User-defined config

It is possible to make SR OS nodes to boot up with a user-defined startup config instead of a built-in one. With a [`startup-config`](../nodes.md#startup-config) property of the node/kind a user sets the path to the config file that will be mounted to a container and used as a startup config:

```yaml
name: sros_lab
topology:
  nodes:
    sros:
      kind: vr-sros
      startup-config: myconfig.txt
```

With such topology file containerlab is instructed to take a file `myconfig.txt` from the current working directory, copy it to the lab directory for that specific node under the `/tftpboot/config.txt` name and mount that dir to the container. This will result in this config to act as a startup config for the node.

#### Configuration save

Containerlab's [`save`](../../cmd/save.md) command will perform a configuration save for `vr-sros` nodes via Netconf. The configuration will be saved under `config.txt` file and can be found at the node's directory inside the lab parent directory:

```bash
# assuming the lab name is "cert01"
# and node name is "sr"
cat clab-cert01/sr/tftpboot/config.txt
```

### License

Path to a valid license must be provided for all vr-sros nodes with a [`license`](../nodes.md#license) directive.

If your SR OS license file is issued for a specific UUID, you can define it with custom type definition:

```yaml
# note, typically only the cp needs the UUID defined.
type: "cp: uuid=00001234-5678-9abc-def1-000012345678 cpu=4 ram=6 slot=A chassis=SR-12 card=cpm5 ___ lc: cpu=4 ram=6 max_nics=36 slot=1 chassis=SR-12 card=iom3-xp-c mda/1=m10-1gb+1-10gb"
```

### File mounts

When a user starts a lab, containerlab creates a node directory for storing [configuration artifacts](../conf-artifacts.md). For `vr-sros` kind containerlab creates `tftpboot` directory where the license file will be copied.

## Lab examples

The following labs feature vr-sros node:

* [SR Linux and vr-sros](../../lab-examples/vr-sros.md)
