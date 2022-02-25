% SNMP Manager and Agent Installation

Seattle Community Networks uses SNMP to monitor network nodes. LibreNMS is used for Network Management, Dashboard generation and Alerting.

# LibreNMS Installation:
[Install LibreNMS](https://docs.librenms.org/Installation/Install-LibreNMS/)
[Install and Configure LibreNMS on Ubuntu with nginx](https://computingforgeeks.com/how-to-install-and-configure-librenms-on-ubuntu-with-nginx/)

# Network-Specific Configuration:
Change active user to librenms:
```sudo su - librenms```

Edit /opt/librenms/config.php:

```
<?php

$config['user'] = 'librenms';
$config['base_url'] = "/";
$config['snmp']['community'] = array('<SNMP COMMUNITY STRING>');
$config['auth_mechanism'] = "mysql"; # default, other options: ldap, http-auth
$config['nets'][] = "10.0.0.0/24";
$config['rrd_purge'] = 0;
$config['enable_billing'] = 1;
$config['show_services'] = 1;
```

As user 'librenms', run /opt/librenms/snmp-scan.php, to scan the configured network for snmp hosts

# Adding Baicells OS configuration to LibreNMS

As user 'librenms' on the librenms server, create the following files and update their contents accordingly:
* For OS detection, ~librenms/includes/definitions/rts.yaml:
```
	os: rts
		text: 'Baicells RTS'
		type: network
		icon: rts
		over:
		- { graph: device_bits, text: 'Device Traffic' }
		- { graph: device_processor, text: 'CPU Usage' }
		- { graph: device_mempool, text: 'Memory Usage' }
		discovery:
		- sysDescr:
			- 'CELL'
```

* For defining custom RTS OS sensors, ~librenms/includes/definitions/discovery/rts.yaml:

```
mib: BAICELLS-MIB
modules:
	os:
    	hardware: BAICELLS-MIB::hardwareVersion.0
    	serial: BAICELLS-MIB::sn.0
    	version: BAICELLS-MIB::softwareVersion.0
	sensors:
    	count:
        	data:
            	-
                	oid: ulThroughput
                	num_oid: '.1.3.6.1.4.1.53058.190.7.{{ $index }}'
                	descr: 'Upload Throughput'
                	group: 'Throughput'
                	index: 'ulthroughput.{{ $index }}'
            	-
                	oid: dlThroughput
                	num_oid: '.1.3.6.1.4.1.53058.190.8.{{ $index }}'
                	descr: 'Download Throughput'
                	group: 'Throughput'
                	index: 'dlThroughput.{{ $index }}'
            	-
                	oid: ulPrbUtilization
                	num_oid: '.1.3.6.1.4.1.53058.190.9.{{ $index }}'
                	descr: 'Upload PRB Utilization'
                	group: 'Utilization'
                	index: 'ulPrbUtilization{{ $index }}'
            	-
                	oid: dlPrbUtilization
                	num_oid: '.1.3.6.1.4.1.53058.190.10.{{ $index }}'
                	descr: 'Download PRB Utilization'
                	group: 'Utilization'
                	index: 'dlPrbUtilization.{{ $index }}'
    	frequency:
        	data:
            	-
                	oid: carrierBwMhz
                	num_oid: '.1.3.6.1.4.1.53058.100.7.{{ $index }}'
                	divisor: 5
                	descr: 'Carrier Bandwidth'
                	index: 'carrierBwMhz.{{ $index }}'
    	percent:
        	data:
            	-
                	oid: eRABEstablishSuccessRate
                	num_oid: '.1.3.6.1.4.1.53058.190.3.{{ $index }}'
                	descr: 'ERAB Establishment Success Rate'
                	group: 'LTE'
                	index: 'eRABEstablishSuccessRate.{{ $index }}'
            	-
                	oid: hoSuccInterEnbS1Rate
                	num_oid: '.1.3.6.1.4.1.53058.190.4.{{ $index }}'
                	descr: 'Inter MME S1 Handover Success Rate'
                	group: 'LTE'
                	index: 'heSuccInterEnbS1Rate.{{ $index }}'
            	-
                	oid: hoSuccInterEnbRate
                	num_oid: '.1.3.6.1.4.1.53058.190.5.{{ $index }}'
                	descr: 'Inter MME Handover Success Rate'
                	group: 'LTE'
                	index: 'hoSuccInterEnbRate.{{ $index }}'
            	-
                	oid: rrcBuildSuccessRate
                	num_oid: '.1.3.6.1.4.1.53058.190.6.{{ $index }}'
                	descr: 'RRC Build Success Rate'
                	group: 'LTE'
                	index: 'rrcBuildSuccessRate.{{ $index }}'
```

* For defining a custom OS class to use Wireless sensors, ~librenms/LibreNMS/OS/Rts.php (note: pay attention to capitalization)

```php
<?php
namespace LibreNMS\OS;

use LibreNMS\Device\WirelessSensor;
use LibreNMS\Interfaces\Discovery\Sensors\WirelessClientsDiscovery;
use LibreNMS\Interfaces\Discovery\Sensors\WirelessUtilizationDiscovery;
use LibreNMS\OS;

class Rts extends OS implements WirelessClientsDiscovery
{
	public function discoverWirelessClients()
	{
    	$oid = '.1.3.6.1.4.1.53058.100.11.0'; //BAICELLS-MIB::ueConnections.0
    	return array(
        	new WirelessSensor('clients', $this->getDeviceId(), $oid, 'rts', 1, 'UE Connections')
    	);
	}
}
```

* A nice looking logo, ~librenms/html/images/os/rts.png
![Baicells Logo](https://imgur.com/9AOohPr.png)		

* Download the baicells mib from [this link](https://na.baicells.com/download/RTS%203.6%20BAICELLS-MIB.mib), and save it to ~librenms/mibs/BAICELLS-MIB (note: no file extension)

* Install snmpd to the ePC node:
``` $ sudo apt install snmpd ```

* Modify /etc/snmp/snmpd.conf:

```
sysLocation 	<SITE NAME STRING>
sysContact  	lcl@seattlecommunitynetwork.org
sysServices 	72
master      	agentx
agentAddress	udp:161
com2sec readonly 10.0.0.1 <SNMP COMMUNITY STRING>
com2sec -Cn ctx_baicells readonly 10.10.0.1 enodeb
group readonlygroup v2c readonly
view all included .1
access readonlygroup "" v2c noauth exact all none none
access readonlygroup ctx_baicells v2c noauth prefix all none none
proxy -Cn ctx_baicells -v 2c -c private 192.168.151.1 .1.3
```

This configuration allows us to access SNMP data on the EPC with the standard community string (refer to internal standards documentation). but will proxy the Baicells SNMP data when we send the community string ‘enodeb’

* Update the snmpd service file to automatically restart snmpd on crash:
   * Edit /lib/systemd/system/snmpd.service, modify the 'ExecStart' line, and add the 'ExecReload', 'Restart', and 'RestartSec' lines:

```
[Unit]
Description=Simple Network Management Protocol (SNMP) Daemon.
After=network.target
ConditionPathExists=/etc/snmp/snmpd.conf

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p /var/run/agentx
ExecStart=/usr/sbin/snmpd -LO2w -u Debian-snmp -g Debian-snmp -I -smux,mteTrigger,mteTriggerConf -f -p /run/snmpd.pid
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

* Enable and restart snmpd:
```
Sudo systemctl daemon-reload
sudo systemctl enable snmpd
sudo systemctl restart snmpd
```

# Baicells SNMP configuration:
* Log into the Baicells configuration console:
https://<Baicells IP Address>

* From the left menu, select System

* Select SNMP
![Example Screenshot: enabling SNMP in the Baicells Console](https://i.imgur.com/YanPtMs.png)
   * Under ‘SNMP Switch,’ select ‘Enable’
   * Configure the following options:
      * Community String: private
      * Contact: lcl@seattlecommunitynetwork.org
      * Location: <SITE NAME STRING> (String should not have any spaces)
      * Source: Any

# Adding New Devices to LibreNMS
* If the ePC is running, librenms should be able to auto-discover it. Run this command from a shell on the jumpbox:
```sudo -u librenms lnms scan```

* LibreNMS should print a status message that it was able to add a new device.

* When first discovered, the ePC will show up generically as it’s ip address. Edit the hostname, but clicking ‘Edit Device’ (gear icon):
   * Click the red pencil icon, and change the ip address to the hostname
   * Fill ‘Overwrite IP’ with the ePC IP address
      * *Note: If the IP is not changed to the hostname, you will not be able to add the eNodeB by it’s IP address*
![Example Screenshot: Updating ePC Hostname in LibreNMS](https://i.imgur.com/LHeL3Zq.png)

* The Baicells eNB needs to be added manually: From LibreNMS, select Devices and click “Add Device”
![Example Screenshot: Manually adding eNodeB to LibreNMS](https://i.imgur.com/Tlqpbh3.png)

* Add a new device, with the following configurations:
   * Hostname: <IP Address of the site’s EPC>
   * Community: ‘enodeb’
   * Force Add: On

* *Note: If you receive an error message stating that a device with the specified IP already exists, make sure that you have* successfully changed the eNodeB’s hostname per the previous step.

* Once the device is added, click the ‘Edit Device’ icon (gear icon) and update the following values:
   * Display name: <eNB Cell Name>
   * Overwrite device contact: lcl@seattlecommunitynetwork.org

# Other helpful notes:

* [Baicells eNB config guide](https://img.baicells.com//Upload/20210810/FILE/195c7e84-47d9-4acb-aa00-cba0e080d885.pdf)

* How to SSH into Baicells eNB:
   * SSH using port 27149 (username same as normal web-based login)
   * Convert the MAC address of this eNB to link local address: http://www.sput.nl/internet/ipv6/ll-mac.html

