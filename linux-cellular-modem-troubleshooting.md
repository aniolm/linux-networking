# Linux Cellular Modem Troubleshooting & Diagnostics

A practical command reference for diagnosing cellular modems on Linux, covering AT commands, QMI, MBIM, ModemManager, WWAN interfaces, SIM/network registration, APN/data sessions, signal quality and common failure modes.

---

## 1. Quick Architecture Overview

Typical Linux cellular stack:

```text
Cellular modem
    │
    ├── AT command ports
    │      └── /dev/ttyUSB* / /dev/ttyACM*
    │
    ├── QMI
    │      ├── /dev/cdc-wdm*
    │      └── wwan0
    │
    ├── MBIM
    │      ├── /dev/cdc-wdm*
    │      └── wwan0
    │
    └── USB / PCIe
           │
           └── kernel driver
                  ├── qmi_wwan
                  ├── cdc_mbim
                  └── usbserial / option

Userspace:
    ModemManager / mmcli
    NetworkManager / nmcli
    libqmi / qmicli
    libmbim / mbimcli
```

Useful rule:

```text
AT       -> modem control / diagnostics
QMI      -> Qualcomm modem management protocol
MBIM     -> generic mobile broadband management protocol
mmcli    -> high-level ModemManager interface
nmcli    -> network connection management
ip       -> Linux network interface/IP/routing
tcpdump  -> packet-level troubleshooting
```

---

# 2. Discover the Modem

## USB devices

```bash
lsusb
lsusb -t
lsusb -v
```

Useful:

```bash
lsusb | grep -i -E 'qualcomm|sierra|quectel|fibocom|telit|simcom|u-blox|wwan'
```

## PCIe modems

```bash
lspci
lspci -nn
lspci -k
```

## Kernel messages

```bash
dmesg | grep -i -E 'usb|wwan|qmi|mbim|modem|ttyUSB|ttyACM'
journalctl -k | grep -i -E 'usb|wwan|qmi|mbim|modem'
```

Follow kernel messages live:

```bash
dmesg -w
```

or:

```bash
journalctl -kf
```

## Device nodes

```bash
ls -l /dev/ttyUSB*
ls -l /dev/ttyACM*
ls -l /dev/cdc-wdm*
ls -l /dev/wwan*
```

Check udev information:

```bash
udevadm info -q all -n /dev/ttyUSB0
udevadm info -q all -n /dev/cdc-wdm0
```

---

# 3. Identify Which Port Is Which

Many modems expose several serial ports:

```text
ttyUSB0
ttyUSB1
ttyUSB2
ttyUSB3
...
```

Do not assume that `ttyUSB0` is the AT port.

Check:

```bash
udevadm info -q property -n /dev/ttyUSB0
```

Useful symlinks:

```bash
ls -l /dev/serial/by-id/
ls -l /dev/serial/by-path/
```

These are preferable to hard-coding `/dev/ttyUSB0`.

---

# 4. Serial Port Basics

Check whether a process owns the port:

```bash
sudo lsof /dev/ttyUSB0
```

or:

```bash
sudo fuser -v /dev/ttyUSB0
```

Typical serial configuration:

```bash
stty -F /dev/ttyUSB0 115200
```

Inspect:

```bash
stty -F /dev/ttyUSB0 -a
```

Common baud rates:

```text
9600
115200
230400
460800
921600
```

---

# 5. Sending AT Commands

## picocom

```bash
sudo picocom -b 115200 /dev/ttyUSB0
```

Exit:

```text
Ctrl-A Ctrl-X
```

## minicom

```bash
sudo minicom -D /dev/ttyUSB0 -b 115200
```

## screen

```bash
sudo screen /dev/ttyUSB0 115200
```

## echo

For simple commands:

```bash
printf 'AT\r' > /dev/ttyUSB0
```

Read:

```bash
cat /dev/ttyUSB0
```

Better for scripts:

```bash
printf 'ATI\r' > /dev/ttyUSB0
timeout 2 cat /dev/ttyUSB0
```

---

# 6. Basic AT Commands

## Modem responsiveness

```text
AT
```

Expected:

```text
OK
```

## Manufacturer

```text
AT+CGMI
```

## Model

```text
AT+CGMM
```

## Revision / firmware

```text
AT+CGMR
```

## Serial number

```text
AT+CGSN
```

or:

```text
AT+GSN
```

## Module information

```text
ATI
```

Typical output contains:

```text
Manufacturer
Model
Revision
IMEI
```

---

# 7. AT Command Help

Generic:

```text
AT+CLAC
```

Some modems support:

```text
AT+<COMMAND>=?
```

Example:

```text
AT+CSQ=?
AT+COPS=?
AT+CGDCONT=?
```

Read current value:

```text
AT+CSQ?
AT+COPS?
AT+CGDCONT?
```

Set value:

```text
AT+CGDCONT=1,"IP","internet"
```

---

# 8. SIM Diagnostics

## SIM presence

```text
AT+CPIN?
```

Typical:

```text
+CPIN: READY
```

Possible states:

```text
READY
SIM PIN
SIM PUK
NOT INSERTED
```

## SIM identifiers

```text
AT+CCID
```

or:

```text
AT+QCCID
```

Quectel-specific examples may differ.

## IMSI

```text
AT+CIMI
```

## SIM status

```text
AT+CPIN?
```

## SIM toolkit / application information

Vendor-specific commands may be available.

---

# 9. Operator / Network Registration

## Registration status

```text
AT+CREG?
```

Circuit-switched registration.

For packet-domain registration:

```text
AT+CGREG?
```

LTE/EPS registration:

```text
AT+CEREG?
```

5G-capable modems may expose additional vendor-specific registration commands.

Typical values:

```text
0 = not registered
1 = registered
2 = searching
3 = registration denied
5 = registered, roaming
```

## Operator

```text
AT+COPS?
```

## Available operators

```text
AT+COPS=?
```

Warning: this can take a long time.

## Packet domain

```text
AT+CGATT?
```

Typical:

```text
+CGATT: 1
```

---

# 10. Signal Quality

Generic:

```text
AT+CSQ
```

Example:

```text
+CSQ: 20,99
```

`99` generally means unknown/not detectable.

For LTE/5G, vendor-specific commands often provide much more useful information.

Typical useful metrics:

```text
RSSI
RSRP
RSRQ
SINR
CQI
PCI
EARFCN
NR-ARFCN
Band
Cell ID
```

---

# 11. Generic PDP Context / APN

List contexts:

```text
AT+CGDCONT?
```

Example:

```text
+CGDCONT: 1,"IP","internet",...
```

Configure:

```text
AT+CGDCONT=1,"IP","internet"
```

IPv4/IPv6 variants may be:

```text
AT+CGDCONT=1,"IPV6","internet"
AT+CGDCONT=1,"IPV4V6","internet"
```

---

# 12. QMI Overview

QMI is commonly used with Qualcomm-based modems.

Typical Linux devices:

```text
/dev/cdc-wdm0
wwan0
```

Check:

```bash
ls -l /dev/cdc-wdm*
ip link show
```

Check kernel modules:

```bash
lsmod | grep -E 'qmi|wwan'
```

Load:

```bash
sudo modprobe qmi_wwan
```

---

# 13. Install QMI Tools

Debian/Ubuntu:

```bash
sudo apt install libqmi-utils
```

Check:

```bash
qmicli --version
```

---

# 14. Basic qmicli Commands

Device capabilities:

```bash
sudo qmicli -d /dev/cdc-wdm0 --dms-get-capabilities
```

Manufacturer:

```bash
sudo qmicli -d /dev/cdc-wdm0 --dms-swi-get-current-firmware
```

Device operating mode:

```bash
sudo qmicli -d /dev/cdc-wdm0 --dms-get-operating-mode
```

IMEI:

```bash
sudo qmicli -d /dev/cdc-wdm0 --dms-uim-get-iccid
```

Depending on modem firmware, exact DMS commands can vary.

---

# 15. QMI Device Identification

```bash
sudo qmicli -d /dev/cdc-wdm0 --dms-get-manufacturer
sudo qmicli -d /dev/cdc-wdm0 --dms-get-model
sudo qmicli -d /dev/cdc-wdm0 --dms-get-revision
sudo qmicli -d /dev/cdc-wdm0 --dms-get-ids
```

---

# 16. QMI SIM / UIM

SIM status:

```bash
sudo qmicli -d /dev/cdc-wdm0 --uim-get-card-status
```

ICCID:

```bash
sudo qmicli -d /dev/cdc-wdm0 --uim-read-transparent=...
```

PIN operations are also exposed through the UIM service, but exact commands depend on the libqmi version and card/application.

---

# 17. QMI Network Registration

Serving system:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system
```

Signal:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-get-signal-info
```

Signal strength:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-get-signal-strength
```

System information:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-get-system-info
```

Cell information:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-get-cell-location-info
```

Operator scan:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-network-scan
```

Warning: network scans can take significant time.

---

# 18. QMI Network Technology

Useful command:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-get-system-info
```

Look for:

```text
LTE
WCDMA
GSM
NR5G
```

Depending on modem/libqmi version, additional NAS commands may expose:

```text
LTE band
NR band
PCI
EARFCN
NR-ARFCN
RSRP
RSRQ
SINR
```

---

# 19. QMI Data Session

QMI raw data sessions are normally handled through WDS.

Start a connection:

```bash
sudo qmicli -d /dev/cdc-wdm0 \
  --wds-start-network="apn=internet,ip-type=4" \
  --client-no-release-cid
```

The command returns a CID and packet-data handle.

Stop:

```bash
sudo qmicli -d /dev/cdc-wdm0 \
  --wds-stop-network=<CID>
```

Exact syntax and options vary by libqmi version.

---

# 20. qmi-network

For simple QMI connection management:

```bash
sudo apt install libqmi-utils
```

Configuration:

```bash
sudoedit /etc/qmi-network.conf
```

Typical:

```text
APN=internet
APN_USER=
APN_PASSWORD=
```

Start:

```bash
sudo qmi-network /dev/cdc-wdm0 start
```

Stop:

```bash
sudo qmi-network /dev/cdc-wdm0 stop
```

Status:

```bash
sudo qmi-network /dev/cdc-wdm0 status
```

---

# 21. QMI WDS Status

```bash
sudo qmicli -d /dev/cdc-wdm0 --wds-get-packet-service-status
```

Useful information:

```text
connected
disconnected
call end reason
verbose call end reason
IP family
```

---

# 22. QMI IP Configuration

After a raw QMI session, query settings:

```bash
sudo qmicli -d /dev/cdc-wdm0 --wds-get-current-settings
```

This can expose:

```text
IPv4 address
IPv4 gateway
IPv4 DNS
IPv4 MTU
IPv6 address
IPv6 gateway
IPv6 DNS
```

Then configure the Linux interface appropriately.

---

# 23. QMI and wwan0

Inspect:

```bash
ip link show wwan0
ip addr show dev wwan0
ip route show dev wwan0
```

Bring interface up:

```bash
sudo ip link set wwan0 up
```

Check:

```bash
ip -s link show wwan0
```

---

# 24. MBIM Overview

MBIM is another common modem protocol.

Kernel driver:

```text
cdc_mbim
```

Typical devices:

```text
/dev/cdc-wdm0
wwan0
```

Check:

```bash
lsmod | grep cdc_mbim
```

Load:

```bash
sudo modprobe cdc_mbim
```

---

# 25. MBIM Tools

Install:

```bash
sudo apt install libmbim-utils
```

Check:

```bash
mbimcli --version
```

Device capabilities:

```bash
sudo mbimcli -d /dev/cdc-wdm0 --query-device-caps
```

Device services:

```bash
sudo mbimcli -d /dev/cdc-wdm0 --query-device-services
```

Subscriber ready state:

```bash
sudo mbimcli -d /dev/cdc-wdm0 --query-subscriber-ready-status
```

Registration:

```bash
sudo mbimcli -d /dev/cdc-wdm0 --query-registration-state
```

Signal:

```bash
sudo mbimcli -d /dev/cdc-wdm0 --query-signal-state
```

---

# 26. MBIM Connect

Example:

```bash
sudo mbimcli -d /dev/cdc-wdm0 \
  --connect="apn=internet,ip-type=ipv4"
```

Disconnect:

```bash
sudo mbimcli -d /dev/cdc-wdm0 --disconnect
```

Check connection state:

```bash
sudo mbimcli -d /dev/cdc-wdm0 --query-connection-state
```

Exact options depend on libmbim version and modem capabilities.

---

# 27. ModemManager

Check service:

```bash
systemctl status ModemManager
```

Start:

```bash
sudo systemctl start ModemManager
```

Enable:

```bash
sudo systemctl enable ModemManager
```

Logs:

```bash
journalctl -u ModemManager
```

Follow:

```bash
journalctl -u ModemManager -f
```

---

# 28. mmcli — List Modems

```bash
mmcli -L
```

Example:

```text
/org/freedesktop/ModemManager1/Modem/0
```

Detailed information:

```bash
mmcli -m 0
```

---

# 29. mmcli — SIM

```bash
mmcli -m 0 --output-keyvalue
```

Look for:

```text
modem.generic.sim
modem.generic.sim-path
modem.3gpp.operator-name
modem.3gpp.operator-code
```

SIM object:

```bash
mmcli -i 0
```

---

# 30. mmcli — Registration

```bash
mmcli -m 0
```

Useful fields:

```text
registration
operator
access technologies
signal quality
```

More detailed:

```bash
mmcli -m 0 --output-json
```

---

# 31. mmcli — Signal

```bash
mmcli -m 0 --signal-get
```

Depending on ModemManager version:

```bash
mmcli -m 0 --signal-setup=10
```

Then:

```bash
mmcli -m 0 --signal-get
```

This can provide technology-specific metrics such as:

```text
LTE RSRP
LTE RSRQ
LTE SINR
NR5G metrics
```

---

# 32. mmcli — Enable Modem

```bash
sudo mmcli -m 0 --enable
```

Disable:

```bash
sudo mmcli -m 0 --disable
```

Reset:

```bash
sudo mmcli -m 0 --reset
```

Power state:

```bash
sudo mmcli -m 0 --set-power-state=on
```

---

# 33. mmcli — Create Data Connection

Example:

```bash
sudo mmcli -m 0 \
  --simple-connect="apn=internet,ip-type=ipv4"
```

Disconnect:

```bash
sudo mmcli -m 0 --simple-disconnect
```

Check:

```bash
mmcli -m 0
```

---

# 34. ModemManager Debug Logging

Temporarily increase logging:

```bash
sudo mmcli -G DEBUG
```

Follow:

```bash
journalctl -u ModemManager -f
```

Restore:

```bash
sudo mmcli -G ERR
```

Depending on ModemManager version, logging options may differ.

---

# 35. NetworkManager

List devices:

```bash
nmcli device
```

Detailed:

```bash
nmcli device show
```

List connections:

```bash
nmcli connection show
```

Device status:

```bash
nmcli device status
```

---

# 36. NetworkManager Cellular

List cellular devices:

```bash
nmcli device | grep -i gsm
```

Create connection:

```bash
sudo nmcli connection add \
  type gsm \
  ifname '*' \
  con-name cellular \
  apn internet
```

Bring up:

```bash
sudo nmcli connection up cellular
```

Bring down:

```bash
sudo nmcli connection down cellular
```

Show:

```bash
nmcli connection show cellular
```

---

# 37. APN Troubleshooting

Verify APN at each layer:

```text
AT+CGDCONT?
```

QMI:

```bash
qmicli ... --wds-get-current-settings
```

ModemManager:

```bash
mmcli -m 0
```

NetworkManager:

```bash
nmcli connection show cellular
```

Common problem:

```text
SIM registered
        ↓
packet service registered
        ↓
APN accepted
        ↓
PDP context created
        ↓
IP address assigned
```

If registration works but APN fails, investigate the data-session layer rather than RF registration.

---

# 38. IP Configuration

```bash
ip addr show wwan0
```

Routes:

```bash
ip route
ip route show dev wwan0
```

IPv6:

```bash
ip -6 addr show dev wwan0
ip -6 route show dev wwan0
```

Neighbor table:

```bash
ip neigh show dev wwan0
ip -6 neigh show dev wwan0
```

Statistics:

```bash
ip -s link show wwan0
```

---

# 39. Connectivity Testing

Ping gateway:

```bash
ping -I wwan0 <gateway>
```

Ping public IP:

```bash
ping -I wwan0 1.1.1.1
```

IPv6:

```bash
ping6 -I wwan0 2606:4700:4700::1111
```

DNS:

```bash
dig example.com
```

Force interface:

```bash
curl --interface wwan0 https://example.com
```

Route lookup:

```bash
ip route get 1.1.1.1
```

---

# 40. Packet Capture

Capture all traffic:

```bash
sudo tcpdump -ni wwan0
```

DNS:

```bash
sudo tcpdump -ni wwan0 port 53
```

ICMP:

```bash
sudo tcpdump -ni wwan0 icmp
```

TCP:

```bash
sudo tcpdump -ni wwan0 tcp
```

HTTP/HTTPS:

```bash
sudo tcpdump -ni wwan0 'tcp port 80 or tcp port 443'
```

Save:

```bash
sudo tcpdump -ni wwan0 -w modem.pcap
```

Read:

```bash
tcpdump -r modem.pcap
```

---

# 41. Kernel WWAN Diagnostics

Check modules:

```bash
lsmod | grep -E 'wwan|qmi|mbim|usbserial|option'
```

Module information:

```bash
modinfo qmi_wwan
modinfo cdc_mbim
modinfo option
```

Loaded drivers:

```bash
lspci -k
lsusb -t
```

---

# 42. USB Problems

Watch USB events:

```bash
udevadm monitor --kernel --udev
```

Watch kernel:

```bash
dmesg -w
```

USB topology:

```bash
lsusb -t
```

Power management:

```bash
cat /sys/bus/usb/devices/*/power/control
```

Check USB errors:

```bash
dmesg | grep -i -E 'usb.*error|reset|disconnect|timeout'
```

---

# 43. Modem Disappeared

Check:

```bash
lsusb
ls -l /dev/cdc-wdm*
ls -l /dev/ttyUSB*
ip link
```

Kernel:

```bash
dmesg | tail -100
```

Check ModemManager:

```bash
mmcli -L
systemctl status ModemManager
```

If USB device vanished completely, investigate:

```text
USB power
USB reset
USB autosuspend
PCIe/USB link
kernel driver
modem firmware crash
```

---

# 44. Port Busy

Find owner:

```bash
sudo lsof /dev/cdc-wdm0
sudo lsof /dev/ttyUSB0
```

or:

```bash
sudo fuser -v /dev/cdc-wdm0
```

Common processes:

```text
ModemManager
NetworkManager
qmi-proxy
mbim-proxy
serial terminal
custom modem daemon
```

Avoid simultaneously controlling the modem through multiple independent tools.

---

# 45. qmi-proxy

Check:

```bash
ps aux | grep qmi
```

Typical:

```text
qmi-proxy
```

Depending on distribution:

```bash
systemctl status libqmi
```

or inspect processes/services directly.

If qmicli reports resource/busy errors, check whether ModemManager or another QMI client is already using the device.

---

# 46. MBIM Proxy

Check:

```bash
ps aux | grep mbim
```

Typical process:

```text
mbim-proxy
```

Again, avoid competing control paths.

---

# 47. Modem Reset Strategy

Soft reset via ModemManager:

```bash
sudo mmcli -m 0 --reset
```

Disable/enable:

```bash
sudo mmcli -m 0 --disable
sudo mmcli -m 0 --enable
```

Power cycle where supported:

```bash
sudo mmcli -m 0 --set-power-state=off
sudo mmcli -m 0 --set-power-state=on
```

Hardware-specific reset may involve:

```text
USB re-enumeration
GPIO reset
power-cycle
PCIe reset
vendor-specific AT command
```

---

# 48. Common Failure: SIM Not Detected

Check:

```text
AT+CPIN?
```

QMI:

```bash
qmicli -d /dev/cdc-wdm0 --uim-get-card-status
```

ModemManager:

```bash
mmcli -m 0
```

Then check:

```text
SIM physically present
SIM voltage
SIM contacts
PIN state
SIM application
UIM service
```

---

# 49. Common Failure: SIM Detected, No Registration

Check:

```text
AT+CREG?
AT+CGREG?
AT+CEREG?
AT+COPS?
```

QMI:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system
sudo qmicli -d /dev/cdc-wdm0 --nas-get-system-info
```

Investigate:

```text
signal
band support
operator availability
roaming
registration mode
antenna
SIM subscription
network restrictions
```

---

# 50. Common Failure: Registered but No Data

This is one of the most important diagnostic distinctions.

Check:

```text
Registration      -> OK
Packet service    -> ?
APN               -> ?
PDP context       -> ?
Data session      -> ?
IP address        -> ?
Route             -> ?
DNS               -> ?
```

AT:

```text
AT+CGATT?
AT+CGDCONT?
```

QMI:

```bash
--wds-get-packet-service-status
--wds-get-current-settings
```

ModemManager:

```bash
mmcli -m 0
```

Linux:

```bash
ip addr show wwan0
ip route
```

---

# 51. Common Failure: Data Session Starts but No IP

Check:

```bash
ip addr show wwan0
```

QMI:

```bash
sudo qmicli -d /dev/cdc-wdm0 --wds-get-current-settings
```

Look for:

```text
IPv4 address
gateway
DNS
MTU
```

Then:

```bash
ip route get 1.1.1.1
```

---

# 52. Common Failure: IP Exists but Internet Does Not Work

Test layers independently:

```bash
ping -I wwan0 <gateway>
ping -I wwan0 1.1.1.1
dig @<dns-server> example.com
curl --interface wwan0 https://example.com
```

Interpretation:

```text
Gateway fails
    -> modem/data-session/L2-L3 issue

Gateway works, public IP fails
    -> routing/carrier/NAT issue

Public IP works, DNS fails
    -> DNS issue

DNS works, HTTPS fails
    -> routing/firewall/MTU/TLS/application issue
```

---

# 53. MTU Diagnostics

Check:

```bash
ip link show wwan0
```

Test:

```bash
ping -M do -s 1400 -I wwan0 1.1.1.1
```

Reduce:

```bash
ping -M do -s 1300 -I wwan0 1.1.1.1
```

Find usable payload:

```bash
for s in 1500 1450 1400 1350 1300 1200; do
    echo "=== $s ==="
    ping -M do -c 1 -s "$s" -I wwan0 1.1.1.1
done
```

---

# 54. Routing Problems

Show all routes:

```bash
ip route
```

Detailed:

```bash
ip -4 route
ip -6 route
```

Which route is selected?

```bash
ip route get 1.1.1.1
```

With source:

```bash
ip route get 1.1.1.1 from <source-ip>
```

Policy routing:

```bash
ip rule
ip route show table all
```

---

# 55. DNS Problems

```bash
resolvectl status
```

Test:

```bash
resolvectl query example.com
```

Direct:

```bash
dig example.com
dig @1.1.1.1 example.com
```

Check:

```bash
cat /etc/resolv.conf
```

---

# 56. IPv6 Cellular Diagnostics

```bash
ip -6 addr show dev wwan0
ip -6 route show
ip -6 neigh show
```

Test:

```bash
ping6 -I wwan0 2606:4700:4700::1111
```

Check modem PDP type:

```text
AT+CGDCONT?
```

Potential context:

```text
IP
IPV6
IPV4V6
```

---

# 57. 5G Diagnostics

Useful concepts:

```text
LTE anchor
NSA
SA
NR5G
5G NSA
5G SA
NR-ARFCN
PCI
RSRP
RSRQ
SINR
band
cell ID
```

QMI:

```bash
sudo qmicli -d /dev/cdc-wdm0 --nas-get-system-info
sudo qmicli -d /dev/cdc-wdm0 --nas-get-signal-info
sudo qmicli -d /dev/cdc-wdm0 --nas-get-cell-location-info
```

AT commands are highly vendor-specific for 5G diagnostics.

---

# 58. Vendor-Specific AT Commands

Do not assume commands are portable.

Examples of vendor families:

```text
Quectel
  AT+Q...
  AT+QCSQ
  AT+QNWINFO
  AT+COPS?
  AT+CGREG?

Sierra Wireless
  AT!...
  AT+CSQ
  AT+CEREG?

Fibocom
  AT+...
  vendor-specific LTE/NR commands

Telit
  AT#...
  vendor-specific commands

SIMCom
  AT+...
  vendor-specific commands
```

Always verify the command against the modem's AT command manual.

---

# 59. Useful Generic AT Diagnostic Sequence

A good first-pass sequence:

```text
AT
ATI
AT+CGMI
AT+CGMM
AT+CGMR
AT+CGSN
AT+CPIN?
AT+CIMI
AT+CCID
AT+COPS?
AT+CREG?
AT+CGREG?
AT+CEREG?
AT+CGATT?
AT+CSQ
AT+CGDCONT?
```

Then vendor-specific RF commands.

---

# 60. Useful QMI Diagnostic Sequence

```bash
sudo qmicli -d /dev/cdc-wdm0 --dms-get-manufacturer
sudo qmicli -d /dev/cdc-wdm0 --dms-get-model
sudo qmicli -d /dev/cdc-wdm0 --dms-get-revision

sudo qmicli -d /dev/cdc-wdm0 --uim-get-card-status

sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system
sudo qmicli -d /dev/cdc-wdm0 --nas-get-signal-info
sudo qmicli -d /dev/cdc-wdm0 --nas-get-system-info

sudo qmicli -d /dev/cdc-wdm0 --wds-get-packet-service-status
sudo qmicli -d /dev/cdc-wdm0 --wds-get-current-settings

ip link show wwan0
ip addr show wwan0
ip route show
```

---

# 61. Useful ModemManager Diagnostic Sequence

```bash
mmcli -L
mmcli -m 0
mmcli -m 0 --output-json
mmcli -m 0 --signal-get
nmcli device status
nmcli device show wwan0
nmcli connection show
journalctl -u ModemManager --since "10 min ago"
journalctl -u NetworkManager --since "10 min ago"
```

---

# 62. Logs to Collect for a Bug Report

Collect:

```bash
uname -a
lsusb
lsusb -t
lspci -k
ip link
ip addr
ip route
ls -l /dev/cdc-wdm*
ls -l /dev/ttyUSB*
mmcli -L
mmcli -m 0
nmcli device status
nmcli connection show
```

Kernel:

```bash
dmesg > dmesg.txt
```

ModemManager:

```bash
journalctl -u ModemManager > modemmanager.log
```

NetworkManager:

```bash
journalctl -u NetworkManager > networkmanager.log
```

---

# 63. Troubleshooting Decision Tree

```text
                    MODEM PROBLEM
                         │
                         ▼
                  Is modem visible?
                    /          \
                  NO            YES
                  │              │
              USB/PCIe       Is SIM visible?
              power/driver     /        \
                              NO        YES
                              │          │
                           SIM/UIM    Registered?
                                      /       \
                                    NO         YES
                                    │           │
                              RF/operator    Data session?
                                             /       \
                                           NO         YES
                                           │           │
                                        APN/PDP      IP?
                                                     /  \
                                                   NO    YES
                                                   │      │
                                                QMI/MBIM  Route/DNS?
                                                         /       \
                                                       NO         YES
                                                       │           │
                                                   network      Application
                                                   config        / MTU /
                                                                 firewall
```

---

# 64. Layered Troubleshooting Model

When debugging, identify the failing layer first:

```text
Layer 1:
USB / PCIe / power / modem firmware

Layer 2:
QMI / MBIM / serial / kernel driver

Layer 3:
SIM / registration / radio / operator

Layer 4:
PDP context / APN / data session

Layer 5:
IP address / gateway / routes

Layer 6:
DNS / firewall / MTU

Layer 7:
Application / HTTPS / service
```

Avoid jumping directly to APN changes when the modem is not even registered.

---

# 65. Useful One-Liners

Find modem devices:

```bash
ls -l /dev/{ttyUSB*,ttyACM*,cdc-wdm*} 2>/dev/null
```

Find modem-related kernel messages:

```bash
dmesg | grep -iE 'modem|wwan|qmi|mbim|cdc-wdm|ttyUSB|ttyACM'
```

Find processes using modem ports:

```bash
sudo lsof /dev/cdc-wdm0 /dev/ttyUSB0 2>/dev/null
```

Check WWAN:

```bash
ip -br link | grep -E 'wwan|usb'
```

Check IP/routing:

```bash
ip -br addr show wwan0; ip route show dev wwan0
```

Check registration through ModemManager:

```bash
mmcli -m 0 | grep -Ei 'state|registration|operator|signal|access'
```

---

# 66. AT vs QMI vs MBIM vs ModemManager

| Interface | Main purpose | Typical device |
|---|---|---|
| AT | Modem control / diagnostics | `/dev/ttyUSB*` |
| QMI | Qualcomm modem protocol | `/dev/cdc-wdm*` |
| MBIM | Mobile broadband protocol | `/dev/cdc-wdm*` |
| ModemManager | High-level modem abstraction | D-Bus |
| NetworkManager | Network configuration | `wwan0` |
| iproute2 | Linux networking | `wwan0` |

Typical production architecture:

```text
Application
    ↓
NetworkManager
    ↓
ModemManager
    ↓
libqmi / libmbim / AT
    ↓
Kernel
    ↓
USB/PCIe
    ↓
Cellular modem
```

---

# 67. Important Practical Rules

### Rule 1 — Identify the control protocol first

Do not blindly run QMI commands against an MBIM modem.

```bash
lsusb -t
ls -l /dev/cdc-wdm*
mmcli -L
```

### Rule 2 — Do not assume ttyUSB numbering

Prefer:

```bash
/dev/serial/by-id/
```

### Rule 3 — Do not run competing managers

Avoid simultaneously controlling the modem with:

```text
ModemManager
qmi-network
manual qmicli
NetworkManager
custom daemon
```

unless you know exactly how they share the device.

### Rule 4 — Registration is not the same as Internet connectivity

```text
registered != data session
data session != IP connectivity
IP connectivity != DNS
DNS != application connectivity
```

### Rule 5 — Use vendor AT documentation

AT commands beyond the 3GPP-defined commands are often vendor-specific.

---

# 68. Recommended First Commands

When receiving a new Linux modem:

```bash
uname -a
lsusb
lsusb -t
dmesg | tail -100
ls -l /dev/ttyUSB*
ls -l /dev/ttyACM*
ls -l /dev/cdc-wdm*
ip link
mmcli -L
```

Then:

```bash
mmcli -m 0
```

If QMI:

```bash
sudo qmicli -d /dev/cdc-wdm0 --dms-get-model
sudo qmicli -d /dev/cdc-wdm0 --nas-get-serving-system
sudo qmicli -d /dev/cdc-wdm0 --nas-get-signal-info
```

If MBIM:

```bash
sudo mbimcli -d /dev/cdc-wdm0 --query-device-caps
sudo mbimcli -d /dev/cdc-wdm0 --query-registration-state
sudo mbimcli -d /dev/cdc-wdm0 --query-signal-state
```

Then inspect:

```bash
ip addr show wwan0
ip route
```

---

## Quick Reference

```bash
# Hardware
lsusb
lsusb -t
dmesg -w

# Serial
ls -l /dev/serial/by-id/
picocom -b 115200 /dev/ttyUSB0

# ModemManager
mmcli -L
mmcli -m 0
mmcli -m 0 --signal-get

# QMI
qmicli -d /dev/cdc-wdm0 --nas-get-serving-system
qmicli -d /dev/cdc-wdm0 --nas-get-signal-info
qmicli -d /dev/cdc-wdm0 --nas-get-system-info
qmicli -d /dev/cdc-wdm0 --wds-get-packet-service-status
qmicli -d /dev/cdc-wdm0 --wds-get-current-settings

# MBIM
mbimcli -d /dev/cdc-wdm0 --query-device-caps
mbimcli -d /dev/cdc-wdm0 --query-registration-state
mbimcli -d /dev/cdc-wdm0 --query-signal-state

# Network
ip link show wwan0
ip addr show wwan0
ip route
ip route get 1.1.1.1

# Debug
journalctl -u ModemManager -f
journalctl -u NetworkManager -f
tcpdump -ni wwan0
```
