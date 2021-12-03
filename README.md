## prep
  add your mesh nodes, note their IP addresses, create DHCP reservations, and maybe DNS records for them

  enable ssh access via webui, add your linux server's user's public key.  you can only have one public key

### wifi
  wireless
  - mode: auto
  - check `disable 11b` unless you actually have 802.11b clients on your network.
  - enable 802.11x if you have clients

  wireless / professional / 2.4ghz
  - roaming assistant `-67` dBm
  - Tx power adjust: `Fair`

  wireless / professional / 5ghz
  - roaming assistant `-63` dBm

## node_exporter

  download and unpack [node_exporter](https://github.com/prometheus/node_exporter) armv5

  put node_exporter on your usb stick, either via linux or scp.

  scp `cron_asus.sh` and `node_asus.sh` to your usb mount in the asus node, usually `/tmp/mnt/sda1/`

  `chomd +x node_asus.sh cron_asus.sh node_exporter`

  background - port 9100's already in use, need to choose an alternate port (9101)

  we include all the runtime options are to minimize memory usage

## node_asus.sh startup script
```
#!/bin/sh

FSOPTS="--no-collector.xfs --no-collector.zfs --no-collector.systemd --no-collector.nfs --no-collector.nfsd --no-collector.btrfs --no-collector.mdadm"

HWOPTS="--no-collector.powersupplyclass --no-collector.hwmon --no-collector.schedstat --no-collector.edac"

NETOPTS="--no-collector.infiniband --no-collector.bonding --no-collector.ipvs"

OTHEROPTS="--no-collector.textfile --no-collector.pressure --no-collector.rapl"

NETDEVOPTS="--collector.netdev.device-exclude=\"dpsta|br.*|bcmsw.*|ifb.*|imq.*|ip6tnl.*|sit.*|^.*dummy.*\""

pidof node_exporter 1> /dev/null

if [[ $? -ne 0 ]] ; then
  /tmp/mnt/sda1/node_exporter --web.listen-address=":9101" ${FSOPTS} ${HWOPTS} ${NETOPTS} ${OTHEROPTS} ${NETDEVOPTS} 2>&1 1> /dev/null
fi
```
## cron_asus.sh
```
#!/bin/sh

/usr/sbin/cru a node-exporter "*/5 * * * * sh /tmp/mnt/sda1/node_asus.sh"
```

## disable usb services
  after you insert your usb stick, mesh nodes will autostart cifs, ftp, etc.

  aimesh menu on the left.  click the node you're interested in.  management on the right.  usb application (should pop-up).  turn off all the usb services

## running node_asus.sh on an asus router ##

  because of the embedded ROFS, we have no access to init scripts in the stock firmware.  however, we can put `node_asus.sh` in cron after the asus boots, attempting to start every 5 minutes.


## linux-server side prep ##

  install scp, ssh, fping, nc/netcat, flock

  put `enable-asus-cron.sh` on your server, probably somewhere in `/opt`?  `chmod +x enable-asus-cron.sh`  add it to cron of the ssh user whose public key is in the webui.  maybe adjust that user's ssh_config for key management, etc

  my cron entry looks like `*  *  * * *   nick sleep 15 && /usr/bin/timeout -s 9 30s /usr/bin/flock -xn /var/lib/flock/nodeexporter /usr/local/bin/enable-asus-cron.sh > /dev/null 2>&1`

 `enable-asus-cron.sh` only attempts to add node_exporter to cron if it's not already running/listening on 9101.

 in `enable-asus-cron.sh`, replace `x.y.z.[abc]` with IPs or hostnames of your asus nodes

## enable-asus-cron.sh
```
#!/bin/sh

for i in x.y.z.a x.y.z.b x.y.z.c; do
  echo "pinging ${i}."

  /usr/sbin/fping -q ${i}
  if [ $? -eq "0" ]; then
    echo " checking node_exporter on ${i}"
      /bin/nc -zw1 ${i} 9101 || ssh admin@${i} 'source /etc/profile; /tmp/mnt/sda1/cron_asus.sh'
      if [ $? -eq "0" ]; then
        echo "  already running on ${i}"
      else
        echo "  starting node_exporter on ${i}"
      fi
    else
      echo " ${i} doesnt ping"
    fi
done
```

## validate node_exporter ##

- `curl http://x.y.z.a:9101/metrics` to see if you get data back
- ssh into the asus node, `ps | grep node_exporter`

## validate asus-side cron job

- ssh into the asus node, run `cru l` to list cron entries.  should see `node_asus.sh` there
