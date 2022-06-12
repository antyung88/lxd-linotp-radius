# lxd-radius-linotp
How to install LinOTP &amp; Radius Server on Ubuntu 20.04 | Integrating FreeRADIUS MFA with AWS

```
+--------+---------+---------------------+-----------------------------------------------+-----------+-----------+
|  NAME  |  STATE  |        IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+--------+---------+---------------------+-----------------------------------------------+-----------+-----------+
| proxy  | RUNNING | 10.54.193.63 (eth0) | fd42:8b30:78e5:d434:216:3eff:feb3:a35f (eth0) | CONTAINER | 0         |
+--------+---------+---------------------+-----------------------------------------------+-----------+-----------+
| radius | RUNNING | 10.54.193.49 (eth0) | fd42:8b30:78e5:d434:216:3eff:fe2b:c40b (eth0) | CONTAINER | 0         |
+--------+---------+---------------------+-----------------------------------------------+-----------+-----------+
```

## Prerequisites
- An AWS Directory Service such as AWS Managed AD or AD Connector
- An EC2 instance based on the Ubuntu:20.04
- LinOTP for Active Directory integration (self-service portal)
- MariaDB or MySQL (We use MySQL)
- FreeRADIUS (RADIUS server) GPLv2 License
- Google Authenticator (secret generation tool) Apache License 2.0 installed on a mobile device
- FQDN (Fully Qualified Domain Name)

# Installation

Connect to the EC2 instance via SSH. Make sure that it is up-to-date.

### LXC LinOTP
Launch LXC Ubuntu 14.04 and enable the [repository](https://linotp.org/doc/2.9.1/part-installation/server-installation/deb_install.html). 
```
lxc launch ubuntu:14.04 radius
lxc exec radius add-apt-repository ppa:linotp/stable
lxc exec radius apt update
lxc exec radius apt install mysql-server
lxc exec radius apt install linotp
lxc exec radius rm /etc/apache2/sites-enabled/000-default.conf
lxc exec radius service apache2 restart
```


### Nginx Reverse Proxy
Launch LXC Ubuntu 20.04
```
lxc launch ubuntu:20.04 proxy
lxc config device add proxy myport80 proxy listen=tcp:0.0.0.0:80 connect=tcp:127.0.0.1:80
lxc config device add proxy myport443 proxy listen=tcp:0.0.0.0:443 connect=tcp:127.0.0.1:443
lxc exec proxy apt update
lxc exec proxy apt install nginx
lxc exec proxy rm /etc/nginx/sites-enabled/default
```
Configure your FQDN name & email below this script
```
cat <<EOF| lxc exec proxy bash
cat <<\EOF_C >/etc/nginx/sites-available/radius
server {
        listen 80;
        listen [::]:80;
        server_name <FQDN>;
        location / {
                proxy_set_header Host \$host;
                proxy_set_header X-Real-IP \$remote_addr;
                proxy_set_header X-Forwarded-Proto \$scheme;
                proxy_pass https://radius.lxd/;
        }
}
EOF_C
ln -s /etc/nginx/sites-available/radius /etc/nginx/sites-enabled/radius
snap install certbot --classic
certbot --nginx -d <FQDN> -m <EMAIL> --agree-tos -n
EOF
```

### Configure LinOTP and integrate with your Active Directory.

1. Browse to: https://<FQDN>/manage Log in with the user name of “admin” and the password you entered in LXC LinOTP
2. Select LinOTP Config menu, choose UserIdResolvers
3. Select New, then choose LDAP
4. Populate the Server Configuration fields. See sample fields below;
- **Resolver name:** Any text to describe the resolver
- **Server-URI:** ldap://[AD_DNS_IP]. You can have multiple IP addresses separated by comma.
- **BaseDN:** The is the root of your domain for LDAP search. E.g DC=example,DC=com. We recommend you use an OU where your users are situated as BaseDN rather than the root of your domain.
- **BindDN:** This is the distinguished name for a domain user with necessary permissions to perform LDAP search on your Active Directory. For example CN=MFAUser,OU=users,DC=example,DC=com
- **Bind Password:** Password of the BindDN user account above.
5. Choose Test LDAP Server connection. On success, choose Preset Active Directory then choose Save.
6. Create a new Realm. Select the UserIdResolver created above. Take note of the Realm name as you need this later.
7. Choose Save. Select save as default.
8. Select the User View tab, you can see the list of users in your Active Directory

![screen-1](https://user-images.githubusercontent.com/99248117/173243818-d4aaed8d-94c4-42f0-9207-698f435a7b96.png)
  
9. Create a file called samplepolicy.cfg. Copy and apste this simple configuration into your simplepolicy.cfg and save the file.

```
[Limit_to_one_token]
realm = *
name = Limit_to_one_token
action = maxtoken=1
client = *
user = *
time = * * * * * *;
active = True
scope = enrollment
[OTP_to_authenticate]
realm = *
name = OTP_to_authenticate
action = otppin = token_pin
client = *
user = *
time = * * * * * *;
active = True
scope = authentication
[Require_MFA_at_Self_Service_Portal]
realm = *
name = Require_MFA_at_Self_Service_Portal
active = False
client = *
user = *
time = * * * * * *;
action = mfa_login
scope = selfservice
[Default_Policy]
realm = *
name = Default_Policy
active = True
client = *
user = *
time = * * * * * *;
action = "enrollTOTP, reset, resync, setOTPPIN, disable"
scope = selfservice
```

10. Select the Policies tab. Choose Import Policies. Import the samplepolicy.cfg you created. You can customize settings to your environment. Refer here for more customization options – [https://www.linotp.org/doc/latest/part-management/policy/index.html](https://www.linotp.org/doc/latest/part-management/policy/index.html)

![screen-2](https://user-images.githubusercontent.com/99248117/173243799-4332f602-8642-417c-b590-6160ee318860.png)

### Enroll Users

- Ensure web ports 443, 80 and UDP 1812 are open on the security group of the instance.
- Users can navigate to https://<FQDN> (Replace IP with the EIP/public IP of your instance). There is a warning about the self-signed certificate. In a production environment, as security best practice, you should put this instance behind an Elastic Load Balancer, and configure Route 53.
- Users should login with their Active Directory user name and password
- On the Enrol TOTP token screen, choose “Generate Random Seed” and “Google Authenticator compliant”. Follow the prompts. When the QR code appears, scan with the authenticator (Google or Microsoft Authenticator) on your mobile

![screen-3](https://user-images.githubusercontent.com/99248117/173243775-93b9072b-3949-45ea-b45b-d15e843b61c2.png)

- Test the token by visiting https://<FQDN>/validate/check?user=USERNAME&pass=PINOTP

If you get below response, then authentication is successful (note: the value is ‘true’)

```
{
"version": "LinOTP 2.11.2",
"jsonrpc": "2.0802",
"result": {
"status": true,
"value": true
},
"id": 0
}
```
  
### Install and configure FreeRADIUS.

```
lxc config device add radius myport1812 proxy listen=tcp:0.0.0.0:1812 connect=tcp:127.0.0.1:1812
lxc exec radius add-apt-repository ppa:freeradius/stable-3.0
lxc exec radius apt install freeradius git
lxc exec radius mv /etc/freeradius/clients.conf /etc/freeradius/clients.conf.back
lxc exec radius mv /etc/freeradius/users /etc/freeradius/users.back
sudo git clone https://github.com/LinOTP/linotp-auth-freeradius-perl.git /usr/share/linotp/linotp-auth-freeradius-perl 
```
```
cat <<EOF| lxc exec radius bash
rm /etc/freeradius/mods-available/perl
cat <<\EOF_C >/etc/freeradius/mods-available/perl
perl {
filename = /usr/share/linotp/linotp-auth-freeradius-perl/radius_linotp.pm
}
EOF_C
ln -s /etc/freeradius/mods-available/perl /etc/freeradius/mods-enabled/perl
EOF
```

Create a file ```/etc/freeradius/clients.conf```. Open the file and copy the example configuration shown below. Replace MYSECRET with your own super secure secret. Replace [CIDR of Directory Service] with your AWS Directory Service Subnets or VPC and [YOUR-NETMASK] to allow VALID request from within the VPC. For example, CIDR = 172.31.10.0 and Netmask =24. Note MYSECRET should be inside single quotes. This is the secret you will use to enable MFA on AWS Directory Service console.

```
cat <<EOF| lxc exec radius bash
cat <<\EOF_C >/etc/freeradius/clients.conf
client localhost {
ipaddr  = 127.0.0.1
netmask= 32
secret  = 'MYSECRET'
}
client adconnector {
ipaddr  = <CIDR of Directory Service subnets or VPC>
netmask = <YOUR-NETMASK>
secret  = 'MYSECRET'
}
EOF_C
EOF
```

Create the file ```/etc/linotp2/rlm_perl.ini``` with below contents. Change YOUR-REALM to the one you created

```
cat <<EOF| lxc exec radius bash
cat <<\EOF_C >/etc/linotp2/rlm_perl.ini
#IP of the linotp server
URL=https://localhost/validate/simplecheck
#optional: limits search for user to this realm
REALM=<YOUR-REALM>
#optional: only use this UserIdResolver
#RESCONF=flat_file
#optional: comment out if everything seems to work fine
Debug=True
#optional: use this, if you have selfsigned certificates, otherwise comment out
SSL_CHECK=False
EOF_C
EOF
```
        
```
lxc exec radius rm /etc/freeradius/sites-enabled/{inner-tunnel,default}
lxc exec radius rm /etc/freeradius/mods-enabled/eap
```

Activate ‘linotp’ within ‘FreeRADIUS’. Create a new file ‘/etc/freeradius/sites-available/linotp‘ with the following content:

```
cat <<EOF| lxc exec radius bash
cat <<\EOF_C >/etc/freeradius/sites-available/linotp
server default {
listen {
type = auth
ipaddr = *
port = 0
limit {
max_connections = 16
lifetime = 0
idle_timeout = 30
}
}
listen {
ipaddr = *
port = 0
type = acct
}authorize {
preprocess
IPASS
suffix
ntdomain
files
expiration
logintime
update control {
Auth-Type := Perl
}
pap
}authenticate {
Auth-Type Perl {
perl
}
}preacct {
preprocess
acct_unique
suffix
files
}accounting {
detail
unix
-sql
exec
attr_filter.accounting_response
}session {
}
post-auth {
update {
&reply: += &session-state:
}
-sql
exec
remove_reply_message_if_eap
}
}
EOF_C
ln -s /etc/freeradius/sites-available/linotp /etc/freeradius/sites-enabled/linotp
service freeradius stop
service freeradius start
EOF
```
