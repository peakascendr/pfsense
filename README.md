# pfsense

freebsd arp behavior is such that arp state table entry ttls are refreshed by successfully communicating with a respective IPv4. Therefore arp requests are only sent if no communication occurs with a respective IP and the arp table entry has expired.

Some ISP's appear to reset the arp tables on their router/gateways frequently or when connectivity is unstable. This feature/behavior combination creates extended outages where pfsense detects gateway down (not responding to ICMP requests) but still sending correctly and the ISP will not respond until an arp request is received. The outage condition is not resolved until the arp entry expires in pfsense and an arp boradcast for the gateway IP is sent. One way to address the conditon would be to lower the arp expiration using system tunable `net.link.ether.inet.max_age` where the default is 1200 seconds, however that could increase arp traffic considerably. 

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
