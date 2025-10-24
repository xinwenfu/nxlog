# Send Windows Event Logs (via NXLog) → rsyslog on Kali Linux

Tools:
- NXLog Community Edition (CE) on Windows
- rsyslog on Kali Linux

## 1. Install rsyslog on Kali

On Kali Linux, syslogd is usually provided by rsyslog, which is the modern replacement for the old syslogd daemon.
Here’s how you can enable and verify it:

### Step 1: Check if rsyslog is installed
```
dpkg -l | grep rsyslog
```

If it’s not installed, run:
```
sudo apt update
sudo apt install rsyslog -y
```
### Step 2: Enable and start the syslog (rsyslog) service
```
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

This ensures it starts automatically on boot and runs immediately.

### Step 3: Verify that it’s running
```
sudo systemctl status rsyslog
```

You should see “active (running)” in green.

### Step 4: Check the default log directory

By default, logs go under:
```
/var/log/
```

For example:
```
/var/log/syslog — general system messages
/var/log/auth.log — authentication messages
/var/log/kern.log — kernel messages
```


### Restart the service:
```
sudo systemctl restart rsyslog
```

## 2. Configure rsyslog on Kali

By default, rsyslog listens only locally (not over the network), so we need to enable remote logging.

Edit the rsyslog config:
```
sudo nano /etc/rsyslog.conf
```

Find and uncomment (remove #) the following lines:

For UDP (port 514):
```
module(load="imudp")
input(type="imudp" port="514")
```

For TCP (port 514):
```
module(load="imtcp")
input(type="imtcp" port="514")
```

Save and restart rsyslog:
```
sudo systemctl restart rsyslog
```

Confirm it’s listening:
```
sudo netstat -tulnp | grep 514
```

You should see something like:
```
udp   0   0 0.0.0.0:514   0.0.0.0:*   1234/rsyslogd
tcp   0   0 0.0.0.0:514   0.0.0.0:*   1234/rsyslogd
```

## 3. Install NXLog Community Edition (CE)
Install NXLog Community Edition. Only the agent (not platform) is needed.
Download **nxlog-6.10.10368_windows_x64.msi** or similar from: https://nxlog.co/downloads

## 4. Configure NXLog CE
Default config file path:
```
C:\Program Files\nxlog\conf\nxlog.conf
```

Edit the config file (nxlog.conf) with **Administrator privileges**.

Below is an example config for sending Windows logs to rsyslog. **Note**: nxlog is installed at C:\Tools\nxlog.

```
Panic Soft

define INSTALLDIR C:\Tools\nxlog

#ModuleDir %INSTALLDIR%\modules
#CacheDir  %INSTALLDIR%\data
#SpoolDir  %INSTALLDIR%\data

define CERTDIR %INSTALLDIR%\cert
define CONFDIR %INSTALLDIR%\conf\nxlog.d

# Note that these two lines define constants only; the log file location
# is ultimately set by the `LogFile` directive (see below). The
# `MYLOGFILE` define is also used to rotate the log file automatically
# (see the `_fileop` block).
define LOGDIR %INSTALLDIR%\data
define MYLOGFILE %LOGDIR%\nxlog.log

# If you are not using NXLog Manager, disable the `include` line
# and enable LogLevel and LogFile.
include %CONFDIR%\*.conf

#LogLevel    INFO
#LogFile     %MYLOGFILE%

<Extension _syslog>
    Module  xm_syslog
</Extension>

# This block rotates `%MYLOGFILE%` on a schedule. Note that if `LogFile`
# is changed in managed.conf via NXLog Manager, rotation of the new
# file should also be configured there.
<Extension _fileop>
    Module  xm_fileop

    # Check the size of our log file hourly, rotate if larger than 5MB
    <Schedule>
        Every   1 hour
        <Exec>
            if ( file_exists('%MYLOGFILE%') and
                 (file_size('%MYLOGFILE%') >= 5M) )
            {
                 file_cycle('%MYLOGFILE%', 8);
            }
        </Exec>
    </Schedule>

    # Rotate our log file every week on Sunday at midnight
    <Schedule>
        When    @weekly
        Exec    if file_exists('%MYLOGFILE%') file_cycle('%MYLOGFILE%', 8);
    </Schedule>
</Extension>


<Input in>
    Module      im_msvistalog
</Input>

# No extra spaces, quotes, or comments on the same line as Host
<Output out>
    Module      om_udp
    Host        10.0.2.25:514
    Exec        to_syslog_bsd();
</Output>

<Route r1>
    Path        in => out
</Route>
```

Note: If you prefer TCP, change om_udp to om_tcp.

Save and restart NXLog service:
```
net stop nxlog
net start nxlog
```

### 5. Verify on Kali that logs are arriving

Run:
```
sudo tail -f /var/log/syslog
```

Then trigger an event on Windows (e.g., open Event Viewer or restart a service).
You should see lines like:
```
Oct 23 09:45:02 WINHOSTNAME Application: Information ... 
```

### If nothing appears:

- Check the firewall on Kali (sudo ufw allow 514/udp)
- Check Windows firewall (allow outbound UDP 514)
- Check NXLog logs at C:\Program Files\nxlog\data\nxlog.log to see anything wrong.
