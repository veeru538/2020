# Install and Configure StrongSwan VPN on Ubuntu 20.04 

A virtual private network is used to create a private network from a public internet connection to protect your identity. VPN uses an encrypted tunnel to send and receive the data securely.

strongSwan is one of the most famous VPN software that supports different operating systems including, Linux, OS X, FreeBSD, Windows, Android, and iOS. It uses IKEv1 and IKEv2 protocol for secure connection establishment. You can extend its functionality with built-in plugins.

In this tutorial, we will explain step by step instructions on how to set up a KEv2 VPN Server with StrongSwan on Ubuntu 20.04.
Prerequisite

• Two systems running Ubuntu 20.04 server
• A root password is configured on both servers
Install StrongSwan

By default, StrongSwan is available in the Ubuntu 20.04 default repository. You can install it with other required components using the following command:
```
apt-get install install strongswan strongswan-pki libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins -y
```
After installing all the packages, you can proceed to generate a CA certificate.
Generate a Certificate for VPN Server

Next, you will need to generate a certificate and key for the VPN server to verify the server’s authenticity on the client side.

First, create a private key for root CA with the following command:
```
ipsec pki --gen --size 4096 --type rsa --outform pem > /etc/ipsec.d/private/ca.key.pem
```
Next, create a root CA and sign it using the above key:
```
ipsec pki --self --in /etc/ipsec.d/private/ca.key.pem --type rsa --dn "CN=My VPN Server CA" --ca --lifetime 3650 --outform pem > /etc/ipsec.d/cacerts/ca.cert.pem
```
Next, create a private key for the VPN server using the following command:
```
ipsec pki --gen --size 4096 --type rsa --outform pem > /etc/ipsec.d/private/server.key.pem
```
Finally, generate the server certificate using the following command:
```
ipsec pki --pub --in /etc/ipsec.d/private/server.key.pem --type rsa | ipsec pki --issue --lifetime 2750 --cacert /etc/ipsec.d/cacerts/ca.cert.pem --cakey /etc/ipsec.d/private/ca.key.pem --dn "CN=vpn.domain.com" --san="vpn.domain.com" --flag serverAuth --flag ikeIntermediate --outform pem > /etc/ipsec.d/certs/server.cert.pem
```
At this point, all certificates are ready for the VPN server.
Configure StrongSwan VPN

The default configuration file of strongswan is /etc/ipsec.conf. We can back up the main configuration file and create a new file:
```
mv /etc/ipsec.conf /etc/ipsec.conf-bak
```
Next, create a new configuration file:
```
nano /etc/ipsec.conf
```
Add the following config and conn settings:
```
config setup
        charondebug="ike 2, knl 2, cfg 2, net 2, esp 2, dmn 2, mgr 2"
        strictcrlpolicy=no
        uniqueids=yes
        cachecrls=no

conn ipsec-ikev2-vpn
      auto=add
      compress=no
      type=tunnel
      keyexchange=ikev2
      fragmentation=yes
      forceencaps=yes
      dpdaction=clear
      dpddelay=300s
      rekey=no
      left=%any
      leftid=@vpn.domain.com
      leftcert=server.cert.pem
      leftsendcert=always 
      leftsubnet=0.0.0.0/0
      right=%any
      rightid=%any
      rightauth=eap-mschapv2
      rightsourceip=10.10.10.0/24
      rightdns=8.8.8.8
      rightsendcert=never
      eap_identity=%identity
```
Save and close the /etc/ipsec.conf file.

Next, you will need to define the EAP user credentials and the RSA private keys for authentication.

You can set up it by editing the file /etc/ipsec.secrets:
```
nano /etc/ipsec.secrets
```
Add the following line:
```
: RSA "server.key.pem"
vpnsecure : EAP "password"
```
Then restart the StrongSwan service as follows:
```
systemctl restart strongswan-starter
```
To enable StrongSwan to start in system boot, type:
```
systemctl enable strongswan-starter
```
Verify the status of the VPN server, type:
```
systemctl status strongswan-starter
```
Enable Kernel Packet Forwarding

Next, you will need to configure the kernel to enable packet forwarding by editing /etc/sysctl.conf file:
```
nano /etc/sysctl.conf
```
Uncomment the following lines:
```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
```
Save and close the file then reload the new settings using the following command:
```
sysctl -p
```
Install and Configure StrongSwan Client

In this section, we will install the StrongSwan client on the remote machine and connect to the VPN server.

First, install all the required packages with the following command:
```
apt-get install strongswan libcharon-extra-plugins -y
```
Once all the packages are installed, stop the StrongSwan service with the following command:
```
systemctl stop strongswan-starter
```
Next, you will need to copy the ca.cert.pem file from the VPN server to /etc/ipsec.d/cacerts/ directory. You can copy it using the SCP command as shown below:
```
scp root@vpn.domain.com:/etc/ipsec.d/cacerts/ca.cert.pem /etc/ipsec.d/cacerts/
```
To setup VPN client authentication, use /etc/ipsec.secrets file:
```
nano /etc/ipsec.secrets
```
Add the following line:
```
vpnsecure : EAP "password"
```
Then edit the strongSwan main configuration file:
```
nano /etc/ipsec.conf
```
Add the following lines that match your domain, password which you have specified in /etc/ipsec.secrets file.
```
conn ipsec-ikev2-vpn-client
    auto=start
    right=vpn.domain.com
    rightid=vpn.domain.com
    rightsubnet=0.0.0.0/0
    rightauth=pubkey
    leftsourceip=%config
    leftid=vpnsecure
    leftauth=eap-mschapv2
    eap_identity=%identity
```
Now start the StrongSwan VPN service using the following command:
```
systemctl start strongswan-starter
```
Next, verify the VPN connection status using the following command:
```
ipsec status
```
You should get the following output:
```
Security Associations (1 up, 0 connecting):
ipsec-ikev2-vpn-client[1]: ESTABLISHED 28 seconds ago, 104.245.32.158[vpnsecure]...104.245.33.84[vpn.domain.com]
ipsec-ikev2-vpn-client{1}: INSTALLED, TUNNEL, reqid 1, ESP in UDP SPIs: ca6f451c_i ca9f9ff7_o
ipsec-ikev2-vpn-client{1}: 10.10.10.1/32 === 0.0.0.0/0
```
The above output indicates that a VPN connection is established between client and server, and the IP address 10.10.10.1 assign to the client machine.

You can also verify your new IP address with the following command:
```
ip a
```
You should get the following output:

```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
link/ether 00:00:68:f5:20:9e brd ff:ff:ff:ff:ff:ff
inet 104.245.32.158/25 brd 104.245.32.255 scope global eth0
valid_lft forever preferred_lft forever
inet 10.10.10.1/32 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::200:68ff:fef5:209e/64 scope link
valid_lft forever preferred_lft forever
```
Conclusion

In the above guide, we learned how to set up a StrongSwan VPN server and client on Ubuntu 20.04. You can now protect your identity and secure your online activities
