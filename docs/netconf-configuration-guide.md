# NETCONF based Network Configuration Guide

## Introduction

This guide walks you through the steps required to configure your Novarq Tactical 1000 switch by using the NETCONF protocol. Here you'll find the configuration steps you need to get your network operational using NETCONF/YANG. A prerequisite required before following these steps is the [Initial Network Configuration Guide](initial-network-configuration-guide.md). Once you can access your device through the network, the NETCONF configuration process can begin.

## Understanding NETCONF/YANG

The Tactical 1000 provides support for NETCONF/YANG through use of the [Netopeer2](https://github.com/cesnet/netopeer2), [sysrepo](https://github.com/sysrepo/sysrepo) and [libyang](https://github.com/cesnet/libyang) projects which are all part of the [sysrepo](https://sysrepo.org) ecosystem.

NETCONF is a network configuration protocol described in [RFC 6241](https://datatracker.ietf.org/doc/html/rfc6241). It provides mechanisms to install, manipulate, and delete the configuration of network devices and also retrieve their operational and telemetry data. It is based on the YANG data modelling language (described in [RFC 7950](https://datatracker.ietf.org/doc/html/rfc7950)), uses XML and JSON for the actual configuration data and is based on a RPC server-client model. It was designed to provide a more robust, reliable, and machine-friendly alternative to legacy methods like SNMP and command-line interface (CLI) scripting and can be used to programmatically configure networks.

Let's see how to configure your system by using NETCONF. The guide will follow previous steps outlined in the [Initial Network Configuration Guide](initial-network-configuration-guide.md)

## Preparation and setup
To start using NETCONF on your system, there are some steps that need to be taken in order to start up all of the NETCONF related software.

```sh
root@noble-lan969x:/# netopeer2-server -d -v 3
```

_Expected output:_
```
[INF]: SR: Connection 3 created.
[INF]: SR: Triggering "ietf-netconf-server" "done" event on enabled data.
[INF]: LN: Listening on 0.0.0.0:830 for SSH connections.
[INF]: SR: Triggering "ietf-keystore" "done" event on enabled data.
[INF]: SR: Triggering "ietf-truststore" "done" event on enabled data.
[INF]: SR: Triggering "libnetconf2-netconf-server" "done" event on enabled data.
[INF]: SR: Triggering "ietf-netconf-acm" "done" event on enabled data.
[INF]: SR: Triggering "ietf-netconf-acm" "done" event on enabled data.
[INF]: SR: Triggering "ietf-netconf-acm" "done" event on enabled data.
[INF]: SR: Triggering "ietf-netconf-acm" "done" event on enabled data.
```

It is recommended to use the debug (`-d`) and verbosity flags (`-v`) the first time you start the Netopeer2 NETCONF server.

Next, start the sysrepo plugins that match what you want to manage through NETCONF. In this case we will be using 
sysrepo-plugin-interfaces from the [sysrepo-plugins](https://github.com/telekom/sysrepo-plugins) plugin collection.

```sh
root@noble-lan969x:~/src/sysrepo-plugins# ./build/plugins/ietf-interfaces-plugin/ietf-interfaces-plugin
```

_Expected output:_
```
[INF] Connection 4 created.
[INF] ietf-interfaces-plugin: Creating plugin subscriptions
[INF] ietf-interfaces-plugin: Registering operational callbacks for module Interfaces
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/admin-status
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/oper-status
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/if-index
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/phys-address
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/higher-layer-if
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/lower-layer-if
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/speed
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/statistics
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/address
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/neighbor
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/address
[INF] ietf-interfaces-plugin: Creating operational subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/neighbor
[INF] ietf-interfaces-plugin: Registering module change callbacks for module Interfaces
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/name
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/enabled
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/description
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/type
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/link-up-down-trap-enable
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/enabled
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/forwarding
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/mtu
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/address/ip
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/address/prefix-length
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/neighbor/ip
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv4/neighbor/link-layer-address
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/enabled
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/forwarding
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/mtu
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/address/ip
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/address/prefix-length
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/neighbor/ip
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/neighbor/link-layer-address
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/dup-addr-detect-transmits
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/ipv6/autoconf/create-global-addresses
[INF] ietf-interfaces-plugin: Creating module change subscription for xpath /ietf-interfaces:interfaces/interface/bridge-port/component-name
[INF] ietf-interfaces-plugin: Registering RPC callbacks for module Interfaces
[INF] ietf-interfaces-plugin: Registered module Interfaces
[INF] ietf-interfaces-plugin: Created plugin subscriptions
```

Lastly, on another system, start `netopeer2-cli` and connect to the `netopeer2-server` running on the Tactical1000 switch.
```sh
root@fedfc68a4ca1:/# netopeer2-cli
load_config: No saved configuration.
> connect 192.168.7.2
```

_Expected output:_:
```
Keyboard-Interactive Authentication
Please enter your authentication token
root's password:
```

## Basic Network Configuration through NETCONF

First off, we can issue a `get-config` command to retrieve the current state of the interfaces on the system:
```sh
> get-config --source running --filter-xpath /interfaces
```

_Expected output:_
```
DATA
<data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
    <interface>
      <name>eth0</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth1</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>true</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
        <address>
          <ip>192.168.7.2</ip>
          <prefix-length>24</prefix-length>
        </address>
        <neighbor>
          <ip>192.168.7.1</ip>
          <link-layer-address>98:fa:9b:78:03:6f</link-layer-address>
        </neighbor>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
        <address>
          <ip>fe80::e080:a3ff:febc:a302</ip>
          <prefix-length>64</prefix-length>
        </address>
        <neighbor>
          <ip>ff02::2</ip>
          <link-layer-address>33:33:00:00:00:02</link-layer-address>
        </neighbor>
        <neighbor>
          <ip>ff02::16</ip>
          <link-layer-address>33:33:00:00:00:16</link-layer-address>
        </neighbor>
        <neighbor>
          <ip>ff02::1:ffbc:a302</ip>
          <link-layer-address>33:33:ff:bc:a3:02</link-layer-address>
        </neighbor>
      </ipv6>
    </interface>
    <interface>
      <name>eth10</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth11</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth12</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth13</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth14</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth15</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth16</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth17</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth18</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth19</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth2</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth20</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth21</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth22</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth23</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth24</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth25</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth26</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth27</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth28</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth3</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth4</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth5</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth6</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth7</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth8</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth9</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>lo</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:softwareLoopback</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <address>
          <ip>127.0.0.1</ip>
          <prefix-length>8</prefix-length>
        </address>
        <neighbor>
          <ip>0.0.0.0</ip>
          <link-layer-address>00:00:00:00:00:00</link-layer-address>
        </neighbor>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>65536</mtu>
        <address>
          <ip>::1</ip>
          <prefix-length>128</prefix-length>
        </address>
      </ipv6>
    </interface>
  </interfaces>
</data>
```

### Step 1: Edit an interface's MTU through NETCONF
To change the configuration data through NETCONF, we will issue an `edit-config` command through the `netopeer2-cli` session that is running from the previous steps.

First create a `mtu.xml` file in the same directory where you started `netopeer2-cli`.
The contents of the file should look like this:
```
<interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
  <interface>
    <name>lo</name>
    <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:softwareLoopback</type>
    <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
      <mtu>12345</mtu>
    </ipv4>
  </interface>
</interfaces>
```

Next, run the following command in the `netopeer2-cli` session:
```sh
> edit-config --target running --config=mtu.xml
```

_Expected output:_
```
OK
>
```

This command should change the loopback (lo) interface MTU.

### Step 2: Verify the interface MTU

To confirm that the MTU has really changed, we can verify it in two ways.
First retrieve the configuration data from NETCONF again.

```sh
> get-config --source running --filter-xpath /interfaces
```

_Expected Output_:
```
...
    <interface>
      <name>lo</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:softwareLoopback</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>12345</mtu>
        <address>
          <ip>127.0.0.1</ip>
          <prefix-length>8</prefix-length>
        </address>
        <neighbor>
          <ip>0.0.0.0</ip>
          <link-layer-address>00:00:00:00:00:00</link-layer-address>
        </neighbor>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>65536</mtu>
        <address>
          <ip>::1</ip>
          <prefix-length>128</prefix-length>
        </address>
      </ipv6>
    </interface>
...
```

This confirms your that the Tactical1000 interfaces can be configured using NETCONF, enabling automated network configuration of the device.

## Configuration Persistence

The commands in this guide configure your switch for the current runtime session. To save these changes between reboots, they need to be copied from the running to the startup datastore.
To do that, run the following command:
```sh
> copy-config --target startup --source running
```

_Expected output:_
```
OK
```

Then issue a `get-config` command for the startup datastore:
```sh
> get-config --source startup --filter-xpath /interfaces
```

_Expected output:_
```
DATA
<data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
    <interface>
      <name>eth0</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth1</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>true</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
        <address>
          <ip>192.168.7.2</ip>
          <prefix-length>24</prefix-length>
        </address>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
        <address>
          <ip>fe80::3cd2:d5ff:fe43:f802</ip>
          <prefix-length>64</prefix-length>
        </address>
        <neighbor>
          <ip>ff02::2</ip>
          <link-layer-address>33:33:00:00:00:02</link-layer-address>
        </neighbor>
        <neighbor>
          <ip>ff02::16</ip>
          <link-layer-address>33:33:00:00:00:16</link-layer-address>
        </neighbor>
        <neighbor>
          <ip>ff02::1:ff43:f802</ip>
          <link-layer-address>33:33:ff:43:f8:02</link-layer-address>
        </neighbor>
      </ipv6>
    </interface>
    <interface>
      <name>eth10</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth11</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth12</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth13</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth14</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth15</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth16</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth17</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth18</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth19</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth2</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth20</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth21</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth22</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth23</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth24</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth25</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth26</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth27</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth28</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth3</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth4</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth5</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth6</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth7</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth8</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>eth9</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>false</enabled>
        <forwarding>false</forwarding>
        <mtu>1500</mtu>
      </ipv6>
    </interface>
    <interface>
      <name>lo</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:softwareLoopback</type>
      <enabled>false</enabled>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>12345</mtu>
        <address>
          <ip>127.0.0.1</ip>
          <prefix-length>8</prefix-length>
        </address>
        <neighbor>
          <ip>0.0.0.0</ip>
          <link-layer-address>00:00:00:00:00:00</link-layer-address>
        </neighbor>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <enabled>true</enabled>
        <forwarding>false</forwarding>
        <mtu>65536</mtu>
        <address>
          <ip>::1</ip>
          <prefix-length>128</prefix-length>
        </address>
      </ipv6>
    </interface>
  </interfaces>
</data>
```

## Compliance

Since the NETCONF subsystem on the Tactical 1000 is based on existing open source projects like Netopeer2 and sysrepo,
it offers support for all of the RFCs and features that the open-source dependencies have.
Most importantly this includes:

### YANG
* Support for YANG Metadata ([RFC 7952](https://tools.ietf.org/html/rfc7952)).
* Support for YANG Schema Mount ([RFC 8528](https://tools.ietf.org/html/rfc8528)).
* Support for YANG 1.1 ([RFC 7950](https://datatracker.ietf.org/doc/html/rfc7950)).

### NETCONF
* Support for [RFC 5277](https://www.rfc-editor.org/rfc/rfc5277.html) NETCONF Event Notifications
* Support for [RFC 6022](https://datatracker.ietf.org/doc/html/rfc6022) YANG Module for NETCONF Monitoring
* Support for [RFC 6242](https://datatracker.ietf.org/doc/html/rfc6242) Using the NETCONF Protocol over Secure Shell (SSH)
* Support for [RFC 7589](https://datatracker.ietf.org/doc/html/rfc7589) Using the NETCONF Protocol over Transport Layer Security (TLS) with Mutual X.509 Authentication
* Support for [RFC 8071](https://www.rfc-editor.org/rfc/rfc8071.html) NETCONF Call Home and RESTCONF Call Home
* Support for [RFC 8341](https://datatracker.ietf.org/doc/html/rfc8341) Network Configuration Access Control Model
* Support for [RFC 8525](https://datatracker.ietf.org/doc/html/rfc8525) YANG Library
* Support for [RFC 8526](https://datatracker.ietf.org/doc/html/rfc8526) NETCONF Extensions to Support the Network Management Datastore Architecture
* Support for [RFC 8639](https://www.rfc-editor.org/rfc/rfc8639.html) Subscription to YANG Notifications
* Support for [RFC 8640](https://www.rfc-editor.org/rfc/rfc8640.html) Dynamic Subscription to YANG Events and Datastores over NETCONF
* Support for [RFC 8641](https://www.rfc-editor.org/rfc/rfc8641.html) Subscription to YANG Notifications for Datastore Updates

### YANG implementation coverage

Due to leveraging the [sysrepo-plugins](https://github.com/telekom/sysrepo-plugins) plugin collection, the Tactical 1000
has support for configuration and management of different subsystems out of the box.

This includes support for the following YANG models:
* ietf-hardware
* ietf-system
* ietf-interfaces
* ietf-if-extensions
* ietf-if-vlan-encapsulation
* ietf-ip
* ietf-routing
* ieee802-dot1q-bridge
* ieee802-dot1q-tpmr
* ieee802-dot1q-vlan-bridge
* ieee802-got1q-pb

## What's Next?

This showcases basic network configuration through NETCONF, leveraging existing open-source projects to achieve greater flexibility and ease of use. The integration of open-source NETCONF software gives Tactical 1000 the foundation to build a simple and reliable network that is ready to scale with complete transparency and control and which can be easily automated using state of the art solutions.
