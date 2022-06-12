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

