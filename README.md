# rsyslog-influx
rsyslog configuration to log directly into Influxdb

The suggested rsyslog integration with Telegraf produced a lot of problems and logs like
```
action 'action 1' resumed (module 'builtin:omfwd')
```
This setup uses a template and the [Influx UDP Input](https://docs.influxdata.com/influxdb/v1.7/supported_protocols/udp/) to deliver the data directly into a database.

## Enable UDP Input for Influxdb
Find the **[[udp]]** section to enable the input. Modify **database** parameter to write it into your db, or it will go into "udp".

### /etc/influxdb/influxdb.conf
```
[[udp]]
  enabled = true
  # bind-address = ":8089"
  # database = "udp"
```

[Check your UDP buffer size](https://docs.influxdata.com/influxdb/v1.7/supported_protocols/udp/)

## Add a configuration to rsyslog
The snippet creates a logging template that writes into a measurement called **syslog**. Rename it if you like. 
### /etc/rsyslog.d/50-influx.conf
```
template(
  name="Influx"
  type="string"
  string="syslog,hostname=%HOSTNAME%,appname=%PROGRAMNAME%,facility=%SYSLOGFACILITY-TEXT%,severity=%SYSLOGSEVERITY-TEXT% severity_code=%SYSLOGSEVERITY%i,facility_code=%SYSLOGFACILITY%i,procid=\"%PROCID%\",message=\"%$!msg%\"\n"
)
set $!msg = replace($msg, "\"", "'");
action(type="omfwd" Protocol="udp" Target="<your influx server>" Port="8089"  Template="Influx")
```

### rsyslog config for the Influx host itself
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
The written data is compatible to Telegrafs tags and fields for syslog. Dashboards and queries should work as before. 
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
