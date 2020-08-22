# Wireguard Cloud Server Tutorial by agavrel

I explain here the steps that will allow you to have your own Wireguard (Ubuntu) server running in no more than 30mn.  
Please bear in mind that this is valuable knowledge as it took me about 2 days to figure out all these steps as some steps are not well explained on internet.  

### Summary : What is Wireguard ?

Read this interesting article: https://restoreprivacy.com/wireguard/

> WireGuard is a new, experimental VPN protocol that aims to offer a simpler, faster, and more secure solution for VPN tunneling than the existing VPN protocols. WireGuard has some major differences when compared to OpenVPN and IPSec, such as the code size (under 4,000 lines!), speed, and encryption standards.

The developer behind WireGuard is Jason Donenfeld, the founder of Edge Security. (The term “WireGuard” is also a registered trademark of Donenfeld.) In one interview I watched, Donenfeld said the idea for WireGuard came when he was living overseas and needed a VPN for Netflix.

This tutorial will explain how to do it. I did with Amazon AWS EC2 service (free for 1 year as long as you don't exceed the 10gb bandwidth per month) but any cloud service would actually work.
* 1) Create an EC2 instance on Amazon (cloud server free for 1 year)
* 2) Setup the server to automatically allow your computer to connect to it
* 3) Install WireGuard on the server
* 4) Configure Wireguard
* 5) Generate the client config files (one for each user or device)

---
### 1) Create an EC2 instance (Amazon) Subscription

Register to Amazon, select USA for the region.

#### Create the Virtual Machine
Once registered click on "Services" and choose "EC2"  
Then click on "Launch Instance" in order to create a server.  

* 1) Amazon Machine Image: Select Ubuntu "Server 18.04 LTS (HVM), SSD Volume Type"  
* 2) Instance: Select "Free tier eligible" Instance of your choice  
* 3) Instance Details: Check "Protect against accidental termination"  
* 4) Storage: Don't go above 30gb if you want a free use.  
* 5) Tag: As you wish  
* 6) Security:  edit the current security group (don't add one) and setup the following ports for in-bound traffic, accessible from anywhere:  

|  Type | Protocol  | Port Range  | Source  | Description  |
|---|---|---|---|---|
| SSH  | TCP  | 22  | Anywhere /0.0.0.0/0, ::/0 |   |
| HTTPS  | TCP  | 443  | Anywhere /0.0.0.0/0, ::/0 |   |
| Custom TCP  | TCP  | 943  | Anywhere /0.0.0.0/0, ::/0 |   |
| Custom UDP  | UDP  | 1194  | Anywhere /0.0.0.0/0, ::/0 |   |

**Review and launch the EC2 instance, when Amazon ask if you want a key say yes, name it aws_us_east_server and download it.**   

Finally you have to allocate a new elastic IP and associate it to the existing EC2 Instance.

---

### 2) Setting up the cloud server from your computer using ssh access (port 22)

First fix the security issue with the file that you downloaded to make it readable only by root
```
chmod 400 ~/Downloads/aws_us_east_server.pem
```

Add it to the ssh keys of your cpu, this will allow access to remote server without having to specify where to find the key  
```
ssh-add ~/Downloads/aws_us_east_server.pem
```

Connect to your remote server (you can close connection anytime with CTRL+D) using the elastic IP you got from AWS:
```
ssh ubuntu@13.84.227.135
```

Thanks to ssh-add you don't need to use such command anymore:
```
ssh -i ~/Downloads/aws_us_east_server.pem ubuntu@13.84.227.135
```

You will receive this kind of message for the first connection:
```
The authenticity of host '13.84.227.135 (13.84.227.135)' can't be established.
ECDSA key fingerprint is SHA256:R38vmPAOI/o4qTIfcdY+jaffBq6V2SCYj5MUex8gbhQ.
Are you sure you want to continue connecting (yes/no)?
```
Type yes and enter.  

If you have any doubt about the security of your server being compromise you can check the last connections to your server by running:
```
grep sshd /var/log/auth.log
```

---
### 3) Install Wireguard

Pretty straigt forward:
```
sudo add-apt-repository ppa:wireguard/wireguard &&
sudo apt-get update &&
sudo apt install wireguard
```

---
### 4) Configure the Wireguard Cloud Server


#### Create the server public and private keys

The most important key will be the one the clients use to connect to your server. Let's generate the private and public keys:
```
mkdir keys
umask 077
wg genkey | tee keys/server_privatekey | wg pubkey > keys/server_publickey
```

If you check the content of server_privatekey you will have this kind of output:
```
2PSr88GjSkO1y/FPXAdVhHYuZnwpyw5kfhFtps5lflY=
```

#### Setup the config file

Just run the following command:
```
echo "
[Interface]
Address = 10.200.200.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820
PrivateKey = $(cat 'keys/server_privatekey')"  | sudo tee -a /etc/wireguard/wg0.conf > /dev/null
```
*Only one important thing: Check that the Private key correctly matches the one in keys/server_privatekey*  

#### Enable IP forwarding

Still on the remote server, you need to allow ip forward with:
```
sudo sysctl -w net.ipv4.ip_forward=1
```
**Without IP forwarding it will not work.**  

And also edit sysctl.conf to make the change permanent:
```
sudo nano /etc/sysctl.conf
uncomment:  net.ipv4.ip_forward = 1
sysctl -p /etc/sysctl.conf
```

#### Start the server

then start the server with:
```
wg-quick up wg0
```

start the system service and enable it:
```
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

You can check that the server is running smoothly by running the following command:
```
sudo wg show
```

#### (Optional but recommended) Setup the firewall rules

```
sudo apt-get install ufw &&
sudo ufw allow 22/tcp &&
sudo ufw allow 51820/udp &&
sudo ufw status verbose
```

**Be very cautious to have allowed SSH port (22) to access your server again** and also port 51820 as it was set as the listening port for wireguard, but any other port than 51820 should work fine.  

After my warning, and having checked that port 22 is allowed, you may enable firewall using the following command:
```
sudo ufw enable
```
You will get the related warning:
```
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
```
Type the letter 'y', then press Enter to activate the firewall and enable on system startup. You will get disconnected from your current ssh session.


---
### 5) Generating Wireguard Client Config files

#### Give your client a name

Using ```${wgclient}``` we will get the value of the client you are currently creating as a tmp environment variable:
```
wgclient=client1
```
You can call it John, Bobby or whatever. This will be the user. It can also be a guid.

#### Generate client private and public keys

You will store the client public and private keys in the keys directory you created
```
umask 077
wg genkey | tee keys/${wgclient}_privatekey | wg pubkey > keys/${wgclient}_publickey
```

Then you have to register this new client to the server config file:
```
echo "
[Peer] #${wgclient}
PublicKey = $(cat keys/${wgclient}_publickey)
AllowedIPs = 10.200.200.2/32" | sudo tee -a /etc/wireguard/wg0.conf > /dev/null

&& wg addconf wg0 <(wg-quick strip wg0)
```
You have to change 10.200.200.*2*/32 number each time you add a client (increment the number).

**WARNING:** Don't forget that the use >> instead of > is mandatory. You want to append and not recreate the file!  

#### Generate client conf file

create the clients folder which will hold the .conf files:
```
mkdir clients
```

create the client conf:
```
echo "[Interface] #${wgclient}
Address = 10.200.200.2/32
PrivateKey = $(cat 'keys/${wgclient}_privatekey')
DNS = 1.1.1.1

[Peer]
PublicKey = $(cat 'keys/server_publickey')
Endpoint = 13.84.227.135:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 21" > clients/${wgclient}.conf
```
**3 important things:**  
* In [Interface] you have to *replace the address' last number* (before 32) with the one that you previously edited
* Here [Peer] is in fact the Wireguard server. *You should check that this PublicKey (named 'server_publickey' and stored in keys folder in this tutorial) match the one you have on your remote server*. ```cat /etc/wireguard/wg0.conf``` will also show you the PrivateKey that should correspond to server_privatekey.
* You have to change the endpoint to *match the elastic IP* you got from AWS  

#### (Optional) Generate a QR code from the .conf file

You can share QR files (squared bar codes pictures) instead of the .conf file by installing qrencode:
```
sudo apt install qrencode
```

Then create the QR code from the client file:
```
qrencode -o ${wgclient}.png -t png < clients/${wgclient}.conf
```

#### Download the QR code on your laptop

##### First method using scp

On your local cpu:
```
scp ubuntu@13.84.227.135:clients/client1.png Downloads/
```

##### Second method using base64

On the remote server:
```
base64 < clients/${wgclient}.png
```

On your local cpu:
```
base64 -d > clients/client1.png
```

##### Setup client on Linux

Install wireguard:

```
sudo add-apt-repository ppa:wireguard/wireguard &&
sudo apt-get update &&
sudo apt install wireguard
```
Now copy the .conf file you got from your remote server to wireguard folder:
```
sudo cp client1.conf /etc/wireguard/
```

and activate with:
```
sudo wg-quick up client1
```



### Any question? Anything I missed?

Feel free to contact me at gavrel dot antonin at gmail dot com
