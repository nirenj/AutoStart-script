# AutoStart-script
Steps to trigger a script each time your CentOS/RHEL server is booted up.

## CREATING THE SCRIPT

1. First we need to create the script. I created mine by the name ```setup-script.sh``` in ```/var/tmp/``` directory.
```shell
#!/bin/bash
echo "The time the script run was --> `date`" > /var/tmp/setup-script.out
echo 'Extracting private ip' >> /var/tmp/setup-script.out
PRIVATE_IP=$(curl http://192.168.0.1/latest/meta-data/local-ipv4)
echo $PRIVATE_IP >> /var/tmp/setup-script.out

echo 'Applying iptable rules' >> /var/tmp/setup-script.out
iptables -A FORWARD -d $PRIVATE_IP -p tcp --dport 443 -j ACCEPT >> /var/tmp/setup-script.out
iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination $PRIVATE_IP:8443 >> /var/tmp/setup-script.out
iptables -t nat -A POSTROUTING -j MASQUERADE >> /var/tmp/setup-script.out
echo 'Appled following iptable rules' >> /var/tmp/setup-script.out
iptables -S >> /var/tmp/setup-script.out
echo 'Appled NAT following iptable rules' >> /var/tmp/setup-script.out
iptables -S -t nat >> /var/tmp/setup-script.out

echo 'Mounting ram disk' >> /var/tmp/setup-script.out
umount /tmp/ram >> /var/tmp/setup-script.out
mount -t tmpfs -o size=512m tmpfs /tmp/ram >> /var/tmp/setup-script.out
service zookeeper start >> /var/tmp/setup-script.out
service ultraesb start >> /var/tmp/setup-script.out
service appserver start >> /var/tmp/setup-script.out
```

2. Then we need to give executable permissions for the script file.
```shell
chmod +x /var/tmp/setup-script.sh
```

## CREATING NEW SYSTEMD SERVICE UNIT

Now we have to create service file. I created a file by the name ```setup.service``` in ```/etc/systemd/system/``` directory.
```shell
[Unit]
Description=This is a setup script
After=network.target

[Service]
Type=simple
ExecStart=/var/tmp/setup-script.sh
TimeoutStartSec=0

[Install]
WantedBy=default.target
```

In this file,
```shell
After= : If the script needs any other system facilities (networking, etc), modify the [Unit] section to include appropriate After=, Wants=, or Requires= directives.
Type= : Switch Type=simple for Type=idle in the [Service] section to delay execution of the script until all other jobs are dispatched
WantedBy= : target to run the sample script in
```

## ENABLE THE SYSTEMD SERVICE UNIT

1. Reload the ```systemd``` process to consider newly created sample.service OR every time when sample.service gets modified.
```shell
systemctl daemon-reload
```

2. Enable this service to start after reboot automatically.
```shell
systemctl enable setup.service
```

3. Start the service.
```shell
systemctl start setup.service
```

4. Reboot the host to verify whether the scripts are starting as expected during system boot.
```shell
systemctl reboot
```

If it’s working we should be able to see the output file in ```setup-script.out``` in ```/var/tmp/``` directory.
