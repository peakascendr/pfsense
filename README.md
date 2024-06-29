# pfsense

freebsd arp behavior is such that arp state table entry ttl's are refreshed when L3 packets are received. Therefore arp requests are only sent if no communication occurs with a the respective IP within the arp entry expiration period.

Some ISP's reset arp tables on their router/gateways frequently or when connectivity is unstable. This feature/behavior combination creates extended outtages where pfsense detects gateway down (not responding to ICMP requests) and ISP will not respond until an arp request is received. The condition is not resolved until the arp entry expires in pfsense and an arp boradcast for the gateway IP is sent. 

To address this condition I've created script and related cron entry that leverages arping to send arp requests to the gateway on a regular basis

## Cron
`running=$(/bin/ps -aux | /usr/bin/grep "arp-gw.sh igb0" | /usr/bin/grep -v "ps -aux" | /usr/bin/grep -v "grep"); if [ "$running" ]; then exit 0; else sh /root/arp-gw.sh igb0; fi`

## Shell Script

```
#!/bin/sh
while true
do
	# Get Default Gateway from dhcp leases
	DGW=`tail -r "/var/db/dhclient.leases.$1" | grep -m1 "router.*" | sed 's/;$//' | awk '{print \$3}'` > /dev/null

	echo $DGW

	if [ -z "DGW" ]
	then
		echo "No Default Gateway Assigned to $1"
		exit 1
	else
		/usr/local/sbin/arping -c1 $DGW >> /dev/null
	fi
	sleep 15
done
```
