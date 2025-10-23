# Send Windows Event Logs (via NXLog) → rsyslog on Kali Linux

## Install rsyslog on Kali

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

## Send Windows Event Logs to Kali Linux

Tools:
- NXLog Community Edition (Windows)
- rsyslog (Kali Linux)

🧩 Step 1: Ensure rsyslog on Kali is ready to receive logs

By default, rsyslog listens only locally (not over the network), so we need to enable remote logging.

Edit the rsyslog config:

sudo nano /etc/rsyslog.conf


Find and uncomment (remove #) the following lines:

For UDP (port 514):

module(load="imudp")
input(type="imudp" port="514")


For TCP (port 514):

module(load="imtcp")
input(type="imtcp" port="514")


Save and restart rsyslog:

sudo systemctl restart rsyslog


Confirm it’s listening:

sudo netstat -tulnp | grep 514


You should see something like:

udp   0   0 0.0.0.0:514   0.0.0.0:*   1234/rsyslogd
tcp   0   0 0.0.0.0:514   0.0.0.0:*   1234/rsyslogd

🧩 Step 2: Install and configure NXLog on Windows

Install NXLog CE
Download from: https://nxlog.co/downloads

Default config file path:

C:\Program Files\nxlog\conf\nxlog.conf


Edit the config file (nxlog.conf) with Administrator privileges.

Replace its contents with the following example config:

## This is a minimal example configuration for sending Windows logs to rsyslog

define ROOT C:\Program Files\nxlog
Moduledir %ROOT%\modules
CacheDir %ROOT%\data
Pidfile %ROOT%\data\nxlog.pid
SpoolDir %ROOT%\data
LogFile %ROOT%\data\nxlog.log

<Extension _syslog>
    Module      xm_syslog
</Extension>

<Input in>
    Module      im_msvistalog
</Input>

<Output out>
    Module      om_udp
    Host        192.168.1.10        # <-- Replace with your Kali IP
    Port        514
    Exec        to_syslog_bsd();
</Output>

<Route 1>
    Path        in => out
</Route>


🔹 If you prefer TCP, change om_udp to om_tcp.

Save and restart NXLog service:

net stop nxlog
net start nxlog


or from the Services GUI → Restart nxlog.

🧩 Step 3: Verify on Kali that logs are arriving

Run:

sudo tail -f /var/log/syslog


Then trigger an event on Windows (e.g., open Event Viewer or restart a service).
You should see lines like:

Oct 23 09:45:02 WINHOSTNAME Application: Information ... 


If nothing appears:

Check the firewall on Kali (sudo ufw allow 514/udp)

Check Windows firewall (allow outbound UDP 514)

Check NXLog logs at C:\Program Files\nxlog\data\nxlog.log
