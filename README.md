# opnsense-wazuh
Tools to integrate 2 great security tools OPNsense and Wazuh

### OPNsense-ban.sh

Ban an offensor IP from a host with wazuh agent installed in the OPNsense (perimeter) firewall via API call triggered by wazuh/ossec agent active response feature.

**Pre requisites**

-**Make sure that your Wazuh Server and OPNsense has your Whitelist IPs configured or you can be banned from your own Firewall!**

-OPNsense Firewall in network perimeter

-Wazuh Server and at least one Host with Wazuh Agent (tested on Linux) installed (with active response enabled) - not tested on OSSEC, but it will probabilly work too.


**OPNsense Firewall steps**

-Create a Firewall Alias with the name `wazuh_activeresponse`

-Create a block rule in WAN interface with the Alias created in the last step and put it in the `Source` option

-Create an user and enable the API in it. Instructions here: https://docs.opnsense.org/development/how-tos/api.html

**Wazuh server steps**

Create a custom rule in `/var/ossec/etc/rules/local_rules.xml`

```
  <rule id="100335" level="10" frequency="3" timeframe="10800">
    <if_matched_sid>3357</if_matched_sid>
    <description>Same source IP blocked 3 times in 3 hours - will be banned</description>
    <same_source_ip />
  </rule>
```

_In the example above I've created a rule using the id `3357` from  the Postfix ruleset to block any offender IP that matches 3 times within 3 hours._ 

Edit the ossec.conf - `/var/ossec/etc/ossec.conf`:

Add a new `command` section:
```
  <command>
    <name>opnsense-ban</name>
    <executable>opnsense-ban.sh</executable>
    <expect>srcip</expect>
  </command>
```

Add a new `active-response` section:

```
  <active-response>
    <command>opnsense-ban</command>
    <location>local</location>
    <rules_id>100335</rules_id>
    <level>10</level>
  </active-response>
  ```
  
  Test your new config syntax:
  `/var/ossec/bin/ossec-analysisd -t`
  
  If everything is OK, reload your new server config:
  `/var/ossec/bin/ossec-control reload`
 
  
**Host with wazuh-agent steps**

Download the script:
```
wget https://raw.githubusercontent.com/cloudfence/opnsense-wazuh/master/opnsense-ban.sh -O /var/ossec/active-response/bin/opnsense-ban.sh 

chmod +x /var/ossec/active-response/bin/opnsense-ban.sh
```

Edit the script and put your OPNsense setting in `#Configuration` section, changing the `KEY`, `SECRET` and `URL` vars:

```
# Configuration
KEY="YOURKEY"
SECRET="TELLMEYOURSECRET"
URL="https://<OPNSENSE_IPADDR>/api/firewall/alias_util/add/wazuh_activeresponse"
```

Restart the agent to fetch the new Wazuh server config:
`/var/ossec/bin/ossec-control restart`

**Testing**

Watch the log file to see bad guys been banned:

`tail -f /var/ossec/logs/active-responses.log`

Ban example:

```
Tue May 21 18:55:27 UTC 2019 /var/ossec/active-response/bin/opnsense-ban.sh add - 200.200.200.200 1558464927.10252529 3353
```

