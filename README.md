## Telegram VPN
> Install ocserv just for Telegram so easily.

### 0. Prerequisites

```
sudo add-apt-repository ppa:certbot/certbot
sudo apt update
sudo apt install software-properties-common certbot ocserv
```

You can use the standalone plugin to obtain TLS certificate from Let’s Encrypt. Run the following command. Don’t forget to set A record for your domain name.

```
sudo certbot certonly --standalone --preferred-challenges http --agree-tos --email <your-email-address> -d <vpn.example.com>
```

Edit root user’s crontab file.

```
sudo crontab -e
```

Add the following line at the end of the file. It’s necessary to restart ocserv service for the VPN server to pick up new certificate and key file.

```
@daily certbot renew --quiet && systemctl restart ocserv
```

### 1. Editing OpenConnect VPN Server Configuration File

Edit ocserv configuration file.

```
sudo nano /etc/ocserv/ocserv.conf
```

First, configure password authentication. By default, password authentication through PAM (Pluggable Authentication Modules) is enabled, which allows you to use Ubuntu system accounts to login from VPN clients. This behavior can be disabled by commenting out the following line.

```
auth = "pam[gid-min=1000]"
```

If we want users to use separate VPN accounts instead of system accounts to login, we need to add the following line to enable password authentication with a password file.

```
auth = "plain[passwd=/etc/ocserv/ocpasswd]"
```

Then find the following two lines. We need to change them.

```
server-cert = /etc/ssl/certs/ssl-cert-snakeoil.pem
server-key = /etc/ssl/private/ssl-cert-snakeoil.key
```

Replace the default setting with the path of Let’s Encrypt server certificate and server key file.

```
server-cert = /etc/letsencrypt/live/<vpn.example.com>/fullchain.pem
server-key = /etc/letsencrypt/live/<vpn.example.com>/privkey.pem
```

Next, find the following line. Change false to true to enable MTU discovery, which can optimize VPN performance.

```
try-mtu-discovery = false
```

After that, set the default domain to vpn.example.com.

```
default-domain = <vpn.example.com>
```

The IPv4 network configuration is as follows by default. This will cause problems because most home routers also set the IPv4 network range to 192.168.1.0/24.

```
ipv4-network = 192.168.1.0
ipv4-netmask = 255.255.255.0
```

We can use another private IP address range (10.10.10.0/24) to avoid IP address collision, so change the value of ipv4-network to

```
ipv4-network = 10.10.10.0
```

After finishing editing this config file, we will see how to use ocpasswd tool to generate the /etc/ocserv/ocpasswd file, which contains a list of usernames and encoded passwords.

Now uncomment the following line to tunnel all DNS queries via the VPN.

```
tunnel-all-dns = true
```

Change DNS resolver address. You can use Google’s public DNS server.

```
dns = 8.8.8.8
dns = 4.4.2.2
```

Add Telegram routes at the end of file

```
route = 91.108.0.0/255.255.0.0
route = 149.154.0.0/255.255.0.0
```

Save and close the file  Then restart the VPN server for the changes to take effect.

```
sudo systemctl restart ocserv
```

### 2. Create OpenConnect user

Now use the ocpasswd tool to generate VPN accounts.

```
sudo ocpasswd -c /etc/ocserv/ocpasswd username
```

You will be asked to set a password for the user and the information will be saved to /etc/ocserv/ocpasswd file. To reset password, simply run the above command again.

### 3. Enable IP Forwarding and Configure Firewall for IP Masquerading

In order for the VPN server to route packets between VPN client and the outside world, we need to enable IP forwarding. Edit sysctl.conf file.

```
sudo nano /etc/sysctl.conf
```

Add the following line at the end of this file.

```
net.ipv4.ip_forward = 1
```

Save and close the file. Then apply the changes with the below command. The -p option will load sysctl settings from /etc/sysctl.conf file. This command will preserve our changes across system reboots.

```
sudo sysctl -p
```

Find the name of your server’s main network interface. Like eth0, ens0

```
ip addr
```

Then run the following command to configure IP masquerading. Replace <eth0> with your own network interface name.

```
sudo iptables -t nat -A POSTROUTING -o <eth0> -j MASQUERADE
```

The above command append (-A) a rule to the end of of POSTROUTING chain of nat table. It will link your virtual private network with the Internet. And also hide your network from the outside world. So the Internet can only see your VPN server’s IP, but can’t see your VPN client’s IP, just like your home router hides your private home network.

Now if you list the rules in the POSTROUTING chain of the NAT table by using the following command:

```
sudo iptables -t nat -L POSTROUTING
```

You can see the Masquerade rule.

Run the following command to open TCP and UDP port 443. If you configured a different port for ocserv, then open your preferred port.

```
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT -p udp --dport 443 -j ACCEPT
```

By default, iptables ruls are lost after reboot. To preserve them, you can switch to root user and then save your rules to a file.

```
sudo iptables-save > /etc/iptables.rules
```

Then create a systemd service file so that we can restore iptables rules at boot time.

```
nano /etc/systemd/system/iptables-restore.service
```

Put the following lines into the file.

```
[Unit]
Description=Packet Filtering Framework
Before=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
ExecStart=/sbin/iptables-restore /etc/iptables.rules
ExecReload=/sbin/iptables-restore /etc/iptables.rules
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Save and close the file. Then reload systemd daemon and enable iptables-restore service.

```
sudo systemctl daemon-reload
sudo systemctl enable iptables-restore
```

Remember to save iptables rules to the file after making changes.

### 4. Optimization

OpenConnect by default uses TLS over UDP protocol (DTLS) to achieve faster speed, but UDP can’t provide reliable transmission. TCP is slower than UDP but can provide reliable transmission. One optimization tip I can give you is to disable DTLS, use standard TLS (over TCP), then enable TCP BBR to boost TCP speed.

To disable DTLS, comment out (add # symbol at the beginning) the following line in ocserv configuration file.

```
udp-port = 443
```

Save and close the file. Then restart ocserv service.

```
sudo systemctl restart ocserv.service
```

To enable TCP BBR, please check out the following tutorial.

- [How to Easily boost Ubuntu Network Performance by enabling TCP BBR](https://www.linuxbabe.com/ubuntu/enable-google-tcp-bbr-ubuntu)

In my test, standard TLS with TCP BBR enabled is two times faster than DTLS.

### 5. Connect to server

```
sudo apt install openconnect
sudo openconnect <vpn.example.com>
```
