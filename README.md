
# cisco_ios

#### Table of Contents

1. [Module Description - What the module does and why it is useful](#module-description)
2. [Setup - The basics of getting started with cisco_ios](#setup)
    * [What cisco_ios affects](#what-cisco_ios-affects)
    * [Setup requirements](#setup-requirements)
    * [Beginning with cisco_ios](#beginning-with-cisco_ios)
3. [Usage - Configuration options and additional functionality](#usage)
4. [Reference - An under-the-hood peek at what the module is doing and how](#reference)
5. [Limitations - OS compatibility, etc.](#limitations)
6. [Development - Guide for contributing to the module](#development)


## Module Description

The Cisco IOS module allows for the configuration of Cisco Catalyst devices running IOS.

Any changes made by this module affect the current `running-config`. These changes will lost on device reboot unless they are backed up to `startup-config`. This module provides a Puppet task to save `running-config` to `startup-config`.

## Setup

### What cisco_ios affects

This module installs the Net::SSH::Telnet gem; and Puppet Resource API gem, if necessary. To activate the Puppet Resource API gem, a reload of the puppetserver service is necessary. In most cases, this should happen automatically and cause little to no interruption to service.

### Setup Requirements

#### Device access

This module requires a user that can access the device via SSH and that has the `enable mode` privilege.

#### Proxy Puppet agent

Since a Puppet agent is not available for the Catalysts (and, seriously, who would want to run an agent on them?) we need a proxy Puppet agent (either a compile master, or another agent) to run Puppet on behalf of the device.

#### Install dependencies

To install dependencies of the Cisco IOS module:

1. Classify or apply the `cisco_ios` class on each master (master of masters, and if present, compile masters and replica master) that serves catalogs for this module.
1. Classify or apply the `cisco_ios` class on each proxy Puppet agent that proxies for Cisco IOS devices.

Run puppet agent -t on the master(s) before using the module on the agent(s).

### Beginning with cisco_ios

To get started, create or edit `/etc/puppetlabs/puppet/device.conf` on the proxy Puppet agent, add a section for the device (this will become the device's `certname`), specify a type of `cisco_ios`, and specify a `url` to a credentials file.

For example:

```INI
[cisco.example.com]
type cisco_ios
url file:////etc/puppetlabs/puppet/devices/cisco.example.com.conf`
```

The credentials file must contain a hash that matches the schema defined in [lib/puppet/transport/schema/cisco_ios.rb](lib/puppet/transport/schema/cisco_ios.rb) for example:

```puppet
{
  host            => '10.0.0.246',
  port            => 22,
  user            => 'admin',
  password        => 'password',
  enable_password => 'password',
}
```

To automate the creation of these files, use the [device_manager](https://forge.puppet.com/puppetlabs/device_manager) module, the `credentials` section should follow the schema as described above:

```puppet
device_manager { 'cisco.example.com':
  type        => 'cisco_ios',
  credentials => {
    host            => '10.0.0.246',
    port            => 22,
    user            => 'admin',
    password        => 'password',
    enable_password => 'password',
  },
}
```

(Using the `device_manager` module will also automatically classify the proxy Puppet agent with the `cisco_ios` class.)

#### Test your setup

Run `puppet device` on the proxy Puppet agent. For example:

`puppet device --verbose --target cisco.example.com`

#### Signing certificates

The first run of `puppet device` for a device will generate a certificate request:

```bash
Info: Creating a new SSL key for cisco.example.com
Info: Caching certificate for ca
Info: csr_attributes file loading from /opt/puppetlabs/puppet/cache/devices/cisco.example.com/csr_attributes.yaml
Info: Creating a new SSL certificate request for cisco.example.com
Info: Certificate Request fingerprint (SHA256): ...
Info: Caching certificate for ca
```

Unless [autosign](https://puppet.com/docs/puppet/latest/ssl_autosign.html) is enabled, the following (depending upon `waitforcert`) will be output:

```bash
Notice: Did not receive certificate
Notice: Did not receive certificate
Notice: Did not receive certificate
...
```

Or:

```bash
Exiting; no certificate found and waitforcert is disabled
```

On the master, execute the following to sign the certificate for the device:

* Puppet 6 or later

```bash
puppetserver ca sign --certname cisco.example.com
```

This will output that the certificate for the device has been signed:

```bash
Successfully signed certificate request for cisco.example.com
```

* Earlier versions of Puppet

```bash
puppet cert sign cisco.example.com
```

This will output that the certificate for the device has been signed:

```bash
Signing Certificate Request for:
  "cisco.example.com" (SHA256) ...
Notice: Signed certificate request for cisco.example.com
Notice: Removing file Puppet::SSL::CertificateRequest cisco.example.com at '/etc/puppetlabs/puppet/ssl/ca/requests/cisco.example.com.pem'
```

**Note (Security Warning)** The SSH server key, and hence its identity, will not be verified during the first connection attempt. Please follow up by verifying the SSH key for the device is correct. The fingerprint will be added to the known hosts file. By default this is the device cache directory eg. `/opt/puppetlabs/puppet/cache/devices/cisco.example.com/ssl/known_hosts`.
This can be changed by setting the `known_hosts_file` value in the [credentials](#credentials) file.

## Usage

Create a manifest with the changes you want to apply. For example:

```puppet
    ntp_server { '1.2.3.4':
      ensure => 'present',
      key => 94,
      prefer => true,
      minpoll => 4,
      maxpoll => 14,
      source_interface => 'Vlan 42',
    }
```

> Note: The `--apply` and `--resource` options are only available with Puppet agent 5.5.0 and higher.

Run `puppet device --apply` on the proxy Puppet agent to apply the changes:

`puppet device  --target cisco.example.com --apply manifest.pp `

Run `puppet device --resource` on the proxy Puppet agent to obtain the current values:

`puppet device --resource --target cisco.example.com ntp_server`

### Tasks

To save the running config, it is possible to use the `cisco_ios::config_save` task. Before running this task, install the module on your machine, along with [Puppet Bolt](https://puppet.com/docs/bolt/latest/bolt_installing.html). When complete, execute the following command:

```
bolt task run cisco_ios::config_save --nodes ios --modulepath <module_installation_dir> --inventoryfile <inventory_yaml_path>
```

The following [inventory file](https://puppet.com/docs/bolt/latest/inventory_file.html) can be used to connect to your switch.

```yaml
# inventory.yaml
nodes:
  - name: cisco.example.com
    alias: ios
    config:
      transport: remote
      remote:
        remote-transport: cisco_ios
        user: admin
        password: password
        enable_password: password
```

The `--modulepath` param can be retrieved by typing `puppet config print modulepath`.

> NOTE: If you have only bolt installed, `puppet config print` does not exist. See [https://puppet.com/docs/bolt/latest/installing_tasks_from_the_forge.html#task-8928](https://puppet.com/docs/bolt/latest/installing_tasks_from_the_forge.html#task-8928) on how bolt can be used to install modules into your boltdir.

## Reference

Please see the netdev_stdlib docs https://github.com/puppetlabs/netdev_stdlib/blob/master/README.md

### Classes

* [`cisco_ios`](#cisco_ios):
* [`cisco_ios::install`](#cisco_iosinstall): Private class

### Resource types

* [`ios_aaa_accounting`](#ios_aaa_accounting): Configure aaa accounting on device
* [`ios_aaa_authentication`](#ios_aaa_authentication): Configure aaa authentication on device
* [`ios_aaa_authorization`](#ios_aaa_authorization): Configure aaa authorization on device
* [`ios_aaa_new_model`](#ios_aaa_new_model): Enable aaa new model on device
* [`ios_aaa_session_id`](#ios_aaa_session_id): Configure aaa session id on device
* [`ios_config`](#ios_config): Execute an arbitrary configuration against the cisco_ios device with or without a check for idempotency.
* [`ios_radius_global`](#ios_radius_global): Extends the radius_global type.
* [`ios_network_trunk`](#ios_network_trunk): Ethernet logical (switch-port) interface. Configures VLAN trunking.
* [`ios_stp_global`](#ios_stp_global): Manages the Cisco Spanning-tree Global configuration resource.
* [`ios_network_dns`](#ios_network_dns): Configure DNS settings for network devices.
* [`ios_additional_syslog_settings`](#ios_additional_syslog_settings): Configure additional global syslog settings.

#### cisco_ios

The cisco_ios class.

#### cisco_ios::install

Private class.

#### ios_config

Execute an arbitrary configuration against the cisco_ios device with or without a check for idempotency

##### attributes

The following attributes are available in the `ios_config` type.

###### `name`

namevar

The friendly name for this ios command.

###### `command`

The ios command to run.

###### `command_mode`

Valid values: CONF_T.

The command line mode to be in, when executing the command

Default value: CONF_T.

###### `idempotent_regex`

Expected string, when running a regex against the 'show running-config'.

###### `idempotent_regex_options`

Array of one or more options which control how the pattern can match.

Allowed values: ['ignorecase', 'extended', 'multiline', 'fixedencoding', 'noencoding']

###### `negate_idempotent_regex`

Boolean

Negate the regex used with idempotent_regex.

Default value: false.

### ios_aaa_accounting

Configure aaa accounting on device

#### Properties

The following properties are available in the `ios_aaa_accounting` type.

##### `ensure`

Data type: `Enum[present, absent]`

Whether this aaa accounting should be present or absent on the target system.

Default value: present

##### `accounting_service`

Data type: `Enum["auth-proxy","commands","connection","dot1x","exec","identity","network","onep","resource","system","update"]`

AAA Accounting service to use

##### `commands_enable_level`

Data type: `Optional[Integer]`

Enable level - needed for "commands" accounting_service

##### `accounting_list`

Data type: `Optional[String]`

The accounting list - named or default

Default value: default

##### `accounting_status`

Data type: `Optional[Enum["none","start-stop","stop-only"]]`

The status of the accounting

##### `server_groups`

Data type: `Optional[Array[String]]`

Array of the server groups eg. `['tacacs+'], ['test1', 'test2']`

##### `update_newinfo`

Data type: `Optional[Boolean]`

Only send accounting update records when we have new acct info. (For periodic use "update_newinfo_periodic") - use with "update" accounting_service.

##### `update_newinfo_periodic`

Data type: `Optional[Integer[1, 2147483647]]`

Periodic intervals to send accounting update records(in minutes) when we have new acct info. (For non-periodic use "update_newinfo")  - use with "update" accounting_service.

##### `update_periodic`

Data type: `Optional[Integer[1, 2147483647]]`

Periodic intervals to send accounting update records(in minutes) (For new acct info only use "update_newinfo_periodic") - use with "update" accounting_service.

#### Parameters

The following parameters are available in the `ios_aaa_accounting` type.

##### `name`

namevar

Data type: `String`

Name. On resource this is a composite of the authorization_service (and enable level if "commands") and authorization_list name eg. "commands 15 default" or "exec authlist1" - or "update" type eg. "update newinfo"

Default value: default

### ios_aaa_authentication

Configure aaa authentication on device

#### Properties

The following properties are available in the `ios_aaa_authentication` type.

##### `ensure`

Data type: `Enum[present, absent]`

Whether this aaa authentication should be present or absent on the target system.

Default value: present

##### `authentication_list_set`

Data type: `Enum["arap","login","enable","dot1x","eou","ppp","sgbp"]`

Set authentication lists for - Login, Enable or dot1x

##### `authentication_list`

Data type: `String`

The authentication list - named or default

Default value: default

##### `server_groups`

Data type: `Optional[Array[String]]`

Array of the server groups eg. `['tacacs+'], ['test1', 'test2']`

##### `enable_password`

Data type: `Optional[Boolean]`

Use enable password for authentication.

##### `local`

Data type: `Optional[Boolean]`

Use local username authentication.

##### `switch_auth`

Data type: `Optional[Boolean]`

Switch authentication.

#### Parameters

The following parameters are available in the `ios_aaa_authentication` type.

##### `name`

namevar

Data type: `String`

Name. On resource this is a composite of the authentication_list_set and authentication_list name eg. "login_default"

Default value: default

### ios_aaa_authorization

Configure aaa authorization on device

#### Properties

The following properties are available in the `ios_aaa_authorization` type.

##### `ensure`

Data type: `Enum[present, absent]`

Whether this aaa authorization should be present or absent on the target system.

Default value: present

##### `authorization_service`

Data type: `Enum["auth-proxy","commands","configuration","exec","network","reverse_access"]`

AAA Authorization service to use

##### `commands_enable_level`

Data type: `Optional[Integer]`

Enable level - needed for "commands" authorization_service

##### `authorization_list`

Data type: `String`

The authorization list - named or default

Default value: default

##### `server_groups`

Data type: `Optional[Array[String]]`

Array of the server groups eg. `['tacacs+'], ['test1', 'test2']`

##### `local`

Data type: `Optional[Boolean]`

Use local database.

##### `if_authenticated`

Data type: `Optional[Boolean]`

Succeed if user has authenticated.

#### Parameters

The following parameters are available in the `ios_aaa_authorization` type.

##### `name`

namevar

Data type: `String`

Name. On resource this is a composite of the authorization_service (and enable level if "commands") and authorization_list name eg. "commands_15_default" or "exec_authlist1"

Default value: default

### ios_aaa_new_model

Enable aaa new model on device

#### Properties

The following properties are available in the `ios_aaa_new_model` type.

##### `enable`

Data type: `Boolean`

Enable or disable aaa new model

#### Parameters

The following parameters are available in the `ios_aaa_new_model` type.

##### `name`

namevar

Data type: `String`

The name stays as "default"

Default value: default

### ios_aaa_session_id

Configure aaa session id on device

#### Properties

The following properties are available in the `ios_aaa_session_id` type.

##### `session_id_type`

Data type: `Enum["common","unique"]`

Type of aaa session id - common or unique

#### Parameters

The following parameters are available in the `ios_aaa_session_id` type.

##### `name`

namevar

Data type: `String`

The name stays as "default"

Default value: default

### ios_access_list

Configure access list on device

#### Properties

The following properties are available in the `ios_access_list` type.

##### `ensure`

Data type: `Enum[present, absent]`

Whether this aaa accounting should be present or absent on the target system.

Default value: present

##### `access_list_type`

Data type: `Enum["Standard","Extended","none"]`

Type of access list - standard, extended or no type

#### Parameters

The following parameters are available in the `ios_access_list` type.

##### `name`

namevar

Data type: `String`

Access list name or number.

### ios_acl_entry

An entry for ACL

#### Properties

The following properties are available in the `ios_acl_entry` type.

##### `ensure`

Data type: `Enum[present, absent]`

Whether this aaa accounting should be present or absent on the target system.

Default value: present

##### `access_list`

Data type: `String`

Name of parent access list

##### `entry`

Data type: `Integer`

Name. Used as sequence number <1-2147483647>

##### `dynamic`

Data type: `Optional[String]`

Name of a Dynamic list

##### `permission`

Data type: `Enum["permit", "deny", "evaluate"]`

Specify packets to forward/reject

##### `protocol`

Data type: `Optional[Variant[Enum["ahp","eigrp","esp","gre","icmp","igmp","ip","ipinip","nos","ospf","pcp","pim","tcp","udp"],Pattern[/\d+/]]]`

ACL Entry Protocol

##### `source_address`

Data type: `Optional[String]`

Source Address. Either Source Address, address object-group, any or source host are required.

##### `source_address_group`

Data type: `Optional[String]`

Source Address object-group. Either Source Address, address object-group, any or source host are required.

##### `source_address_any`

Data type: `Optional[Boolean]`

Source Address. Either Source Address, address object-group, any or source host are required.

##### `source_address_host`

Data type: `Optional[String]`

Source Address. Either Source Address, address object-group, any or source host are required.

##### `source_address_wildcard_mask`

Data type: `Optional[String]`

Source Address wildcard mask. Must be used with, and only used with, Source Address.

##### `source_eq`

Data type: `Optional[Array[String]]`

Match only packets on a given port number.

##### `source_gt`

Data type: `Optional[String]`

Match only packets with a greater port number.

##### `source_lt`

Data type: `Optional[String]`

Match only packets with a lower port number.

##### `source_neq`

Data type: `Optional[String]`

Match only packets not on a given port number.

##### `source_portgroup`

Data type: `Optional[String]`

Destination port object-group.

##### `source_range`

Data type: `Optional[Array[String]]`

Match only packets in the range of port numbers.

##### `destination_address`

Data type: `Optional[String]`

Destination Address. Either Destination Address, address object-group, any or destination host are required.

##### `destination_address_group`

Data type: `Optional[String]`

Destination Address object-group. Either Destination Address, address object-group, any or destination host are required.

##### `destination_address_any`

Data type: `Optional[Boolean]`

Destination Address. Either Destination Address, address object-group, any or destination host are required.

##### `destination_address_host`

Data type: `Optional[String]`

Destination Address. Either Destination Address, address object-group, any or destination host are required.

##### `destination_address_wildcard_mask`

Data type: `Optional[String]`

Destination Address wildcard mask. Must be used with, and only used with, Destination Address.

##### `destination_eq`

Data type: `Optional[Array[String]]`

Match only packets on a given port number.

##### `destination_gt`

Data type: `Optional[String]`

Match only packets with a greater port number.

##### `destination_lt`

Data type: `Optional[String]`

Match only packets with a lower port number.

##### `destination_neq`

Data type: `Optional[String]`

Match only packets not on a given port number.

##### `destination_portgroup`

Data type: `Optional[String]`

Destination port object-group.

##### `destination_range`

Data type: `Optional[Array[String]]`

Match only packets in the range of port numbers.

##### `ack`

Data type: `Optional[Boolean]`

Match on the ACK bit.

##### `dscp`

Data type: `Optional[String]`

Match packets with given dscp value.

##### `fin`

Data type: `Optional[Boolean]`

Match on the FIN bit.

##### `fragments`

Data type: `Optional[Boolean]`

Check non-initial fragments.

##### `icmp_message_code`

Data type: `Optional[Integer]`

ICMP message code.

##### `icmp_message_type`

Data type: `Optional[Variant[String, Integer]]`

ICMP message type.

##### `igmp_message_type`

Data type: `Optional[Variant[String, Integer]]`

IGMP message type.

##### `log`

Data type: `Optional[Boolean]`

Log matches against this entry. Either log or log_input can be used, but not both.

##### `log_input`

Data type: `Optional[Boolean]`

Log matches against this entry, including input interface. Either log or log_input can be used, but not both.

##### `match_all`

Data type: `Optional[Array[String]]`

Match if all specified flags are present.

##### `match_any`

Data type: `Optional[Array[String]]`

Match if any specified flags are present.

##### `option`

Data type: `Optional[String]`

Match packets with given IP Options value.

##### `precedence`

Data type: `Optional[String]`

Match packets with given precedence value.

##### `psh`

Data type: `Optional[Boolean]`

Match on the PSH bit.

##### `reflect`

Data type: `Optional[String]`

Create reflexive access list entry.

##### `reflect_timeout`

Data type: `Optional[Integer]`

Maximum time to live in seconds. Only to be used with reflect.

##### `rst`

Data type: `Optional[Boolean]`

Match on the RST bit.

##### `syn`

Data type: `Optional[Boolean]`

Match on the SYN bit.

##### `time_range`

Data type: `Optional[String]`

Specify a time-range.

##### `tos`

Data type: `Optional[String]`

Match packets with given TOS value.

##### `urg`

Data type: `Optional[Boolean]`

Match on the URG bit.

#### Parameters

The following parameters are available in the `ios_acl_entry` type.

##### `name`

namevar

Data type: `String`

Name. Made up of access_list and the entry with an underscore seperator. eg. list42_10 is from access_list list42 and entry 10.

### ios_radius_global

Extends the radius_global type.

#### Properties

The following properties are available in the `ios_radius_global` type.

##### `attributes`

Data type: `Optional[Array[Tuple[Integer, String]]]`

An array of [attribute number, attribute options] pairs

> NOTE: There are a huge number of attributes available across devices with varying configuration options. Some of these pose issues for idempotency.
>
> This modules does not attempt to solve these issues and you should take care to review your settings.
>
> Example:
>
> [11, 'default direction inbound'] will set correctly, however the device will return [11, 'default direction in']. You should prefer setting [11, 'default direction in']
>
> Example:
>
> [11, 'default direction outbound'] will set correctly, however the device will remove the setting from the config as this is a default. You should instead prefer not setting this option.


See `radius_global` for other available fields

#### ios_network_trunk

Ethernet logical (switch-port) interface.  Configures VLAN trunking. Extension of network_trunk.


##### Properties

The following properties are available in the `ios_network_trunk` type.

###### `access_vlan`

Data type: `Optional[Variant[Integer[0, 4095], Boolean[false]]`

The VLAN to set when the interface is in access mode. Setting it to false will revert it to the default value.

Examples:

```Puppet
access_vlan => 405
```

```Puppet
access_vlan => false
```

###### `voice_vlan`

Data type: `Optional[Variant[Integer[0, 4095], Enum["dot1p", "none", "untagged"], Boolean[false]]]`

Sets how voice traffic should be treated by the access port. Setting it to false will revert it to the default value.

Examples:

```Puppet
access_vlan => 221
```

```Puppet
access_vlan => 'dot1p'
```

###### `allowed_vlans`

Data type: `Optional[Variant[Enum["all", "none"], Tuple[Enum["add", "remove", "except"], String], String, Boolean[false]]]`

Sets which VLANs the access port will use when trunking is enabled. Setting it to false will revert it to the default value.

Examples:

```Puppet
access_vlan => '101-202'
```

```Puppet
access_vlan => 'none'
```

```Puppet
access_vlan => ['except', '204-301']
```

###### `switchport_nonegotiate`

Data type: `Optional[Boolean]`

When set, prevents the port from sending DTP (Dynamic Trunk Port) messages. Set automatically to true while in 'access mode' and cannot be set in 'dynamic_*' mode.

Examples:

```Puppet
access_vlan => true
```

See `network_trunk` for other availible fields.

### ios_stp_global

Manages the Cisco Spanning-tree Global configuration resource.

#### Properties

The following properties are available in the `ios_stp_global` type.

##### `enable`

Data type: `Optional[Boolean]`

Enable or disable STP functionality [true|false]

##### `bridge_assurance`

Data type: `Optional[Boolean]`

Bridge Assurance on all network ports

##### `extend_system_id`

Data type: `Optional[Boolean]`

Extend system-id into priority portion of the bridge id (PVST & Rapid PVST only)

##### `loopguard`

Data type: `Optional[Boolean]`

Bridge Assurance on all network ports

##### `mode`

Data type: `Optional[Enum["mst","pvst","rapid-pvst"]]`

Operating Mode

##### `mst_forward_time`

Data type: `Optional[Integer]`

Forward delay for the spanning tree

##### `mst_hello_time`

Data type: `Optional[Integer]`

Hello interval for the spanning tree

##### `mst_inst_vlan_map`

Data type: `Optional[Array[Tuple[Integer,String]]]`

An array of [mst_inst, vlan_range] pairs.

##### `mst_max_age`

Data type: `Optional[Integer[6,40]]`

Max age interval for the spanning tree

##### `mst_max_hops`

Data type: `Optional[Integer[1,255]]`

Max hops value for the spanning tree

##### `mst_name`

Data type: `Optional[String]`

Configuration name.

##### `mst_priority`

Data type: `Optional[Array[Tuple[String,Integer]]]`

An array of [mst_inst_list, priority] pairs.

##### `mst_revision`

Data type: `Optional[Integer]`

Configuration revision number.

##### `pathcost`

Data type: `Optional[Enum["long","short"]]`

Method to calculate default port path cost

##### `portfast`

Data type: `Optional[Array[Enum["default","bpduguard_default","bpdufilter_default"]]]`

Spanning tree portfast options

##### `uplinkfast`

Data type: `Optional[Boolean]`

Enable UplinkFast Feature

##### `uplinkfast_max_update_rate`

Data type: `Optional[Integer]`

Maximum number of update packets per second

##### `vlan_forward_time`

Data type: `Optional[Array[Tuple[String,Integer]]]`

An array of [vlan_inst_list, forward_time] pairs.

##### `vlan_hello_time`

Data type: `Optional[Array[Tuple[String,Integer]]]`

An array of [vlan_inst_list, hello_time] pairs.

##### `vlan_max_age`

Data type: `Optional[Array[Tuple[String,Integer]]]`

An array of [vlan_inst_list, max_age] pairs.

##### `vlan_priority`

Data type: `Optional[Array[Tuple[String,Integer]]]`

An array of [vlan_inst_list, priority] pairs.

#### Parameters

The following parameters are available in the `ios_stp_global` type.

##### `name`

namevar

Data type: `String`

ID of the stp global config. Valid values are default.

Default value: default

#### ios_network_dns

Configure DNS settings for network devices.

#### Properties

The following properties are available in the `ios_network_dns` type.

##### ip_domain_lookup

Data type: `Optional[Boolean]`

Sets whether the Domain Name Server (DNS) lookup feature should be enabled.

See `network_dns` for other available fields.

### ios_additional_syslog_settings

Configure additional global syslog settings.

#### Properties

The following properties are available in the `ios_additional_syslog_settings` type.

##### `trap`

Data type: `Optional[Variant[Integer[0,7], Enum["unset"]]]`

Set the syslog server logging level, can be set to a severity level of [0-7] or 'unset'.

Examples:

```Puppet
ios_additional_syslog_settings { "default":
  trap => 3,
}
```

```Puppet
ios_additional_syslog_settings { "default":
  trap => 'unset',
}
```

##### `origin_id`

Data type: `Optional[Variant[Enum['hostname', 'ip', 'ipv6', unset], Tuple[Enum['string'], String]]]`

Sets an origin-id to be added to all syslog messages, can be set to a default value taken from the switch itself or a designated one word string.

Examples:

```Puppet
ios_additional_syslog_settings { "default":
  origin_id => 'ipv6',
}
```

```Puppet
ios_additional_syslog_settings { "default":
  origin_id => ['string', 'Main'],
}
```

```Puppet
ios_additional_syslog_settings { "default":
  origin_id => 'unset',
}
```

## Limitations

We support devices across the Catalyst family — both legacy IOS and IOS-XE. We continually perform testing against a limited set of physical platforms. If you experience errors or missing abstractions, provide the error details (if present) via --debug --trace and a copy of the sanitized configuration (both Puppet and Cisco CLI).

### Devices used in testing
| Device Type | IOS Version                                                                                                                                |
|-------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| 2960        | Cisco IOS Software, C2960S Software (C2960S-UNIVERSALK9-M), Version 12.2(58)SE2, RELEASE SOFTWARE (fc1)                                    |
| 3650        | Cisco IOS Software, IOS-XE Software, Catalyst L3 Switch Software (CAT3K_CAA-UNIVERSALK9-M), Version 03.06.05.E RELEASE SOFTWARE (fc2)      |
| 3750        | Cisco IOS Software, C3750 Software (C3750-IPSERVICESK9-M), Version 12.2(55)SE10, RELEASE SOFTWARE (fc2)                                    |
| 4503        | Cisco IOS Software, IOS-XE Software, Catalyst 4500 L3 Switch  Software (cat4500e-UNIVERSALK9-M), Version 03.07.03.E RELEASE SOFTWARE (fc3) |
| 4507r       | Cisco IOS Software, Catalyst 4000 L3 Switch Software (cat4000-I5K91S-M), Version 12.2(25)EWA9, RELEASE SOFTWARE (fc3)                      |
| 4948        | Cisco IOS Software, Catalyst 4500 L3 Switch Software (cat4500-ENTSERVICESK9-M), Version 12.2(37)SG1, RELEASE SOFTWARE (fc2)                |
| 6503        | Cisco IOS Software, s72033_rp Software (s72033_rp-IPSERVICESK9_WAN-M), Version 12.2(33)SXJ10, RELEASE SOFTWARE (fc3)                       |

### Resources vs Device type
| Resource                   | 2960                 | 3650                 | 3750                 | 4503                 | 4507r                | 4948                 | 6503                 |
|----------------------------|----------------------|----------------------|----------------------|----------------------|----------------------|----------------------|----------------------|
| banner                     | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| domain_name                | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      |
| ios_config                 | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| ios_radius_global          | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| ios_stp_global             | ok*                  | ok*                  | ok*                  | ok*                  | ok*                  | ok*                  | ok                   |
| name_server                | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      |
| network_dns                | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| network_interface          | ok*                  | ok*                  | ok*                  | ok                   | ok                   | ok                   | ok                   |
| network_snmp               | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| network_trunk              | ok*                  | ok*                  | ok                   | ok*                  | ok                   | ok                   | ok                   |
| network_vlan               | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| ntp_auth_key               | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| ntp_config                 | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| ntp_server                 | ok                   | ok                   | ok*                  | ok                   | ok                   | ok*                  | ok                   |
| port_channel               | ok*                  | ok*                  | ok*                  | ok*                  | ok*                  | ok*                  | ok                   |
| radius                     | not supported by IOS | not supported by IOS | not supported by IOS | not supported by IOS | not supported by IOS | not supported by IOS | not supported by IOS |
| radius_global*             | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| radius_server              | ok                   | ok                   | not supported        | ok                   | not supported        | not supported        | not supported        |
| radius_server_group        | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| search_domain              | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      | use network_dns      |
| snmp_community             | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| snmp_notification          | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| snmp_notification_receiver | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| snmp_user                  | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| syslog_server              | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| syslog_settings            | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| tacacs                     | not supported by IOS | not supported by IOS | not supported by IOS | not supported by IOS | not supported by IOS | not supported by IOS | not supported by IOS |
| tacacs_global*             | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| tacacs_server              | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |
| tacacs_server_group        | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   | ok                   |

Cells marked with the * have deviations. See the section below for details.

### Deviations

#### network_interface

##### 2960

The switch does not support the MTU on a per-interface basis. It does not support the following attributes: [link](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960/software/release/15-2_2_e/configuration/guide/b_1522e_2960_2960c_2960s_2960sf_2960p_cg/b_1522e_2960_2960c_2960s_2960sf_2960p_cg_chapter_01001.html)

* mtu

##### 3650

The switch does not support the MTU on a per-interface basis. It does not support the following attributes: [link](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3650/software/release/3se/int_hw_components/configuration_guide/b_int_3se_3650_cg/b_int_3se_3650_cg_chapter_0110.html)

* mtu

##### 3750

The switch does not support the MTU on a per-interface basis. It does not support the following attributes: [link](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/12-2_55_se/configuration/guide/scg3750/swint.html)

* mtu

#### network_trunk

##### 2960

This device does not have native trunking. It does not support the following attributes: [link](https://learningnetwork.cisco.com/thread/75947)

* ensure
* encapsulation

##### XE OS

This device type only supports a single method of encapsulation, `802.1q`, and as such the attribute to set it is not supported.

#### ntp_server

##### 3750

Does not support the following attributes: [link](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750x_3560x/software/release/12-2_55_se/configuration/guide/3750xscg/swadmin.html)

* minpoll
* maxpoll

##### 4948

Does not support the following attributes: [link](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/31sga/configuration/guide/config/swadmin.html)

* minpoll
* maxpoll

##### 4507

Does not support the following attributes: [link](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/31sga/configuration/guide/config/swadmin.html#wp1245750)

* minpoll
* maxpoll

#### port_channel

##### 2960

##### 3560

##### 3650

##### 3750

##### 4503

##### 4507

##### 4948

The above devices do not have native trunking. The following attributes are not supported: [link](https://learningnetwork.cisco.com/thread/75947)

* flowcontrol_send

#### radius_global

The IOS operating system does not support:

* enable

#### radius_server

##### 3750

##### 4507r

##### 4948

##### 6503

The IOS operating system needs to support the new "radius server" command, we do not use "radius-server" [link](https://www.cisco.com/c/en/us/support/docs/security-vpn/remote-authentication-dial-user-service-radius/200403-AAA-Server-Priority-explained-with-new-R.html)

#### ios_stp_global

##### 3650

##### 3750

##### 2960

##### 4503

##### 4507

##### 4948

This device does not support bridge assurance [link](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960/software/release/12-2_53_se/configuration/guide/2960scg/swstp.html)

#### syslog_settings

Does not implement `vrf` or `time_stamp_units` as described in the netdev_stdlib type definition.

#### tacacs_server

##### 2960

##### 3750

The IOS operating system uses the deprecated "tacacs_server" syntax, we cannot use 'unset' functionality for individual fields [link](https://slaptijack.com/networking/new-style-tacacs-configuration/)

#### tacacs_global

The IOS operating system does not support:

* enable
* retransmit_count

### Anomalies in Cisco CLI

#### ntp_server

It has been noted that NTP Server configuration may allow multiple entries of the same NTP Server address with different Source Interfaces

For example:
````
ntp server 1.2.3.4 key 42
ntp server 1.2.3.4 key 94 source Vlan42
ntp server 1.2.3.4 key 50 source Loopback42
````
While Puppet Resource will obtain all entries, Puppet Apply compares against the first entry found with the same name.

##### Workaround

Send an ensure 'absent' manifest to remove all ntp servers of the same name, before rebuilding the ntp server configuration:

````Puppet
    ntp_server { '1.2.3.4':
      ensure => 'absent',
    }
````

followed by:

````Puppet
    ntp_server { '1.2.3.4':
      ensure => 'present',
      key => 94,
      prefer => true,
      minpoll => 4,
      maxpoll => 14,
      source_interface => 'Vlan 42',
    }
````

Any edits can be made by referencing the same ntp_server name and source_interface.

## Development

Contributions are welcome, especially if they can be of use to other users.

Checkout the [repo](https://github.com/puppetlabs/cisco_ios) by forking and creating your feature branch.

Prior to development, copy the types from the [netdev standard library](https://github.com/puppetlabs/netdev_stdlib/tree/master/lib/puppet/type) to the `/lib/puppet/types` directory.

See the [command guide for IOS](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/bsm/command/bsm-cr-book.html).

### Type

Add new types to the type directory.
We use the [Resource API format](https://github.com/puppetlabs/puppet-resource_api/blob/master/README.md)
Use the bundled ios_config example for guidance. Here is a simple example:

```Ruby
  require 'puppet/resource_api'

  Puppet::ResourceApi.register_type(
    name: 'new_thing',
    docs: 'Configure the new thing of the device',
    features: ['remote_resource'],
    attributes: {
      ensure:       {
        type:       'Enum[present, absent]',
        desc:       'Whether the new thing should be present or absent on the target system.',
        default:    'present',
      },
      name:         {
        type:      'String',
        desc:      'The name of the new thing',
        behaviour: :namevar,
      },
      # Other fields in resource API format
    },
  )

```

### Provider

Add a provider — see existing examples. Parsing logic is contained in `ios.rb`. Regular expressions for parsing, getting and setting values, are contained within `command.yaml`.

### Modes

If the new provider requires accessing a CLI "mode", for example, Interface `(config-if)`, add this as a new mode state to [`Puppet::Transport::CiscoIos`](lib/puppet/transport/cisco_ios.rb) and an associated prompt to [`command.yaml`](lib/puppet/transport/command.yaml).

### Testing

There are 2 levels of testing found under `spec`.

#### Unit Testing

Unit tests test the parsing and command generation logic executed locally. Specs typically iterate over `read_tests` and `update_tests`, which contain testing values within `test_data.yaml`.

Execute with `bundle exec rake spec`.

#### Acceptance Testing

Acceptance tests are executed on actual devices.

Use test values and make sure that these are non-destructive.

Typically, the following flow is used:

- Remove any existing entry
- Add test
- Edit test — with as many values as possible
- Remove test

Any other logic or values that can be tested should be added, as appropriate.

##### Executing

Ensure that the IP address/hostname, username, password and enable password are specified as environment variables from your execution environment, for example:

```
export DEVICE_IP=10.0.10.20
export DEVICE_USER=admin
export DEVICE_PASSWORD="devicePa$$w0rd"
export DEVICE_ENABLE_PASSWORD="enablePa$$w0rd"
```

Execute the acceptance test suite with the following command:

`BEAKER_provision=yes PUPPET_INSTALL_TYPE=pe BEAKER_set=vmpooler bundle exec rspec spec/acceptance/`
