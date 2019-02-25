# Overview
This implements a rsyslog configuration that logs directly into Influxdb.

I tested the [suggested rsyslog integration with Telegraf](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/syslog) and got a lot of error logs like
```
action 'action 1' resumed (module 'builtin:omfwd')
```
I wasn't able to fix the root cause and decided to use a different approach. My setup uses a log template and the [Influx UDP Input](https://docs.influxdata.com/influxdb/v1.7/supported_protocols/udp/) to deliver the data directly into a database. It runs smooth without reducing functionality.

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
Restart rsyslog if you are done.

### rsyslog configuration for the Influx host itself
If you want to collect data on the host running the InfluxDB you need a slightly different configuration to avoid a logging feedback loop. Here it is necessary to exclude influxd logs from the forwarding.
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
A simple test to check if your setup works is to ask the database directly. 
```
root@collector ~ # influx
Connected to http://localhost:8086 version 1.7.1
InfluxDB shell version: 1.7.1
Enter an InfluxQL query
> use udp   (or your DB name)
Using database udp
> select * from syslog limit 3
name: syslog
time                appname facility facility_code hostname message                                                          procid severity severity_code
----                ------- -------- ------------- -------- -------                                                          ------ -------- -------------
1550679421543669928 CRON    authpriv 10            colector  pam_unix(cron:session): session opened for user root by (uid=0) 3317   info     6
1550679421545719847 CRON    cron     9             colector  (root) CMD (   cd / && run-parts --report /etc/cron.hourly)     3318   info     6
1550679421546549914 CRON    authpriv 10            colector  pam_unix(cron:session): session closed for user root            3317   info     6
```

The written data fields and tags are based on Telegrafs syslog measurement. Dashboards and queries should work as before. 
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
