# Overview
This implements a rsyslog configuration that logs directly into Influxdb.

I tested the [suggested rsyslog integration with Telegraf](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/syslog) and got a lot of error logs like
```
action 'action 1' resumed (module 'builtin:omfwd')
```
I wasn't able to fix the root cause and decided to use a different approach. My setup uses a template and the [Influx UDP Input](https://docs.influxdata.com/influxdb/v1.7/supported_protocols/udp/) to deliver the data directly into a database. It runs smooth without reducing functionality.

# Setup 
## 1.) Enable UDP Input for Influxdb
Find the **[[udp]]** section to enable the input. Modify the **database** parameter to write it into your db, or it will go into the standard "udp" database.

**/etc/influxdb/influxdb.conf**
```
[[udp]]
  enabled = true
  # bind-address = ":8089"
  # database = "udp"
```
Restart influx and make sure to [adjust your UDP buffer size!](https://docs.influxdata.com/influxdb/v1.7/supported_protocols/udp/)

## 2.) Add a configuration file to rsyslog
Create a new file in rsyslogs config folder.

**/etc/rsyslog.d/50-influx.conf**
```
template(
  name="Influx"
  type="string"
  string="syslog,hostname=%HOSTNAME%,appname=%PROGRAMNAME%,facility=%SYSLOGFACILITY-TEXT%,severity=%SYSLOGSEVERITY-TEXT% severity_code=%SYSLOGSEVERITY%i,facility_code=%SYSLOGFACILITY%i,procid=\"%PROCID%\",message=\"%$!msg%\"\n"
)
set $!msg = replace($msg, "\"", "'");
action(type="omfwd" Protocol="udp" Target="<your influx server>" Port="8089"  Template="Influx")
```
This logging template writes into a measurement called **syslog**. Rename it if you like. 

### rsyslog configuration for the Influx host itself
To avoid a logging feedback loop it is necessary to exclude influxd logs from the forwarding.
```
template(
  SAME AS ABOVE
)
set $!msg = replace($msg, "\"", "'");
if $programname != 'influxd' then {
  action(type="omfwd" Protocol="udp" Target="localhost" Port="8089"  Template="Influx")
}

```

# Using the measurement
The written data filds and keys are based on Telegrafs syslog measurement. Dashboards and queries should work as before. 
```
> show field keys from syslog
name: syslog
fieldKey      fieldType
--------      ---------
facility_code integer
message       string
procid        string
severity_code integer

> show tag keys from syslog
name: syslog
tagKey
------
appname
facility
hostname
severity
```
