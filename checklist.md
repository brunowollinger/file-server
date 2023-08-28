# Server Setup

The purpose of this checklist is provide a guide line to develop a automation script. This Guide focus on Debian 11 Bullseye.

## Checklist

Although not required adding /usr/sbin to the PATH environment variable makes it easier to call some binaries in this guide

```bash
# This folder have some binaries that will not be recognized otherwise
export PATH=$PATH:/usr/sbin

# This change is only for the current session, to make it 
# permanent to the current user edit the .bashrc file
echo "PATH=$PATH:/usr/sbin" >> ~/.bashrc
```

### Network Setup

- [ ] Static IP Configuration

<details>
    <summary>Commands</summary>

```bash
# Check the available interface name, usually eth0
ip addr

# Backup the configuration file before editing
cp /etc/network/interfaces /etc/network/interfaces.old

# Edit the interfaces config file
vim /etc/network/interfaces

# Append or edit the interface entry
auto eht0
iface etho inet static
    address 192.168.15.180/24 # IP address / Subnet mask format
    gateway 192.168.15.1

# Save the file and restart the service
systemctl restart networking
```

</details>

- [ ] Basic Firewall Setup

<details>
    <summary>Commands</summary>

```bash
# Backup the original configuration file
cp /etc/nftables.conf /etc/nftables.conf.old

# Enable and start nftables
systemctl enable nftables && systemctl start nftables

# Add SSH rule to prevent lockout to the server
nft add rule inet filter input ip saddr 192.168.15.0/24 tcp dport 22 accept

# Replace the default input chain policy to drop all other packets not specified
nft chain inet filter input '{ type filter hook input priority filter ; policy drop ; }'

# Replace the default forward chain policy to drop all other packets since the server is not a router
nft chain inet filter forward '{ type filter hook forward priority filter ; policy drop ; }'

# Add neighbour discovery rule for ipv6
nft add rule inet filter input icmpv6 type { nd-neighbor-solicit, nd-router-advert, nd-neighbor-advert } accept

# Add rule for established, related and invalid packets
nft add rule inet filter input ct state vmap { established : accept, related : accept, invalid : drop } 

# Add rule to comunicate with localhost
nft add rule inet filter input iifname "lo" accept

# Make the changes persistent
nft list ruleset > /etc/nftables.conf
```

</details>

<hr>

### SSH Server Setup

- [ ] Public Key Creation

<details>
<summary>Commands</summary>

```bash
# On the server create the file containing the public keys for every computer the user will access from
mkdir ~/.ssh && touch ~/.ssh/authorized_keys

# Generate the private and public key on the client computer
ssh-keygen

# Add the newly created key to the ssh-agent, backup you private key and delete it from the computer
ssh-add ~/.ssh/id_rsa # Linux and Windows Powershell

# Transfer the public key from your client (Windows or Linux) to the server
scp ~/.ssh/id_rsa.pub user@192.168.15.180:~/.ssh/

# On the server send content of the .pub file to the file containing all authorized keys
cat ~/.ssh/id_rsa.pub >> authorized_keys
```

</details>

- [ ] Public Key Authentication

<details>
<summary>Commands</summary>

```bash
# Backup the configuration file before making any changes
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.old

# Uncomment the entries related to public key authentication
sed -i '/#AuthorizedKeysFile/s/^#//' /etc/ssh/sshd_config
sed -i '/#PermitEmptyPasswords/s/^#//' /etc/ssh/sshd_config

# Disable password authentication
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Enable and restart the SSH Server
systemctl enable sshd && systemctl restart sshd
```

</details>

- [ ] Disable Root Login

<details>
<summary>Commands</summary>

```bash
# Prevent root user from login in directly from ssh
sed -i '/#PermitRootLogin/s/^#//' /etc/ssh/sshd_config &&
sed -i 's/prohibit-password/no/' /etc/ssh/sshd_config

# Restart the service to apply changes
systemctl restart sshd
```

</details>

- [ ] Login Attempt Duration 

<details>
<summary>Commands</summary>

```bash
# Uncomment the LoginGraceTime and change to a reasonable time
sed -i '/#LoginGraceTime/s/^#//' /etc/ssh/sshd_config

# Restart the service to apply changes
systemctl restart sshd
```

</details>

- [ ] Limit Access User Access

<details>
<summary>Commands</summary>

```bash
# Limit Access by username. change <user> for desired user
echo -e "\nAllowUsers <user>" >> /etc/ssh/sshd_config

# Or limit access by groups. change <group> for desired group
echo -e "\nAllowGroups <group>" >> /etc/ssh/sshd_config

# Restart the service to apply changes
systemctl restart sshd
```

</details>

<hr>

### LDAP Server Setup

- [ ] Installation

<details>
    <summary>Commands</summary>

```bash
# Install the ldap server and utilities
apt update && apt install -y slapd ldap-utils

# Enable and start the daemon
systemctl enable slapd && systemctl start slapd
```

</details>

- [ ] Basic Configuration

<details>
    <summary>Commands</summary>

Update domain name, base dn and password without dpkg-reconfigure

```bash
# Create a .ldif to set the base dn, new root dn (admin account) and its password
cat <<EOF > db.ldif
# Change base dn
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=example,dc=com
-
# Change the root user name
replace: olcRootDN
olcRootDN: cn=ldapadm,dc=example,dc=com
-
# Change the password for the root user
replace: olcRootPW
olcRootPW: 
EOF

# Generate the new password hash and redirect it to the previously created .ldif file
slapdpasswd >> db.ldif

# Make the changes
ldapmodify -Y EXTERNAL -H ldapi:/// -f ./db.ldif

# Test the changes
ldapwhoami -D 'cn=ldapadm,dc=example,dc=com' -W -H ldapi:///

# It should return the root base dn
dn:cn=ldapadm,dc=example,dc=com

# Add the base DN
ldapwhoami -D 'cn=ldapadm,dc=example,dc=com' -W -H ldapi:/// <<EOF
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: example.com
dc: example
EOF

# Confirm the changes
ldapsearch -D cn=ldapadm,dc=example,dc=com -W -H ldapi:/// -b "dc=example,dc=com" "(objectClass=organization)" -LLL

# it should return
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: example.com
dc: example
```

- [ ] Configure SSL/TLS

</details>

- [ ] Enable LDAPS

<details>
    <summary>Commands</summary>

```bash
# Enable LDAPS on port 636
sed -i '/SLAPD_SERVICES.*"$/s/"$/ ldaps:\/\/\/"/' /etc/default/slapd

# Restart the daemon
systemctl restart slapd
```

</details>

- [ ] Generate Internal Certificate Chain

<details>
    <summary>Commands</summary>

```bash
# Generate a private key for the Root CA
openssl genpkey -algorithm RSA -out internal-root-ca.key

# Generate a self-signed certificate for the Root CA
openssl req -x509 -new -nodes -key internal-root-ca.key -sha256 -days 3650 -out internal-root-ca.pem

# Generate a private key for the Sub-CA
openssl genpkey -algorithm RSA -out sub-ca.key

# Generate a CSR (Certificate Signing Request) for the Sub-CA
openssl req -new -key sub-ca.key -out sub-ca.csr

# Sign the Sub-CA CSR with the Root CA to create the Sub-CA certificate
openssl x509 -req -in sub-ca.csr -CA internal-root-ca.pem -CAkey internal-root-ca.key -CAcreateserial -out internal-sub-ca.pem -days 365

# Generate a private key for the server
openssl genpkey -algorithm RSA -out server.key

# Generate a CSR for the server
openssl req -new -key server.key -out server.csr

# Sign the Server CSR with the Sub-CA to create the server certificate
openssl x509 -req -in server.csr -CA internal-sub-ca.pem -CAkey sub-ca.key -CAcreateserial -out server.crt -days 365
```

</details>

- [ ] Adjust file permissions

<details>
    <summary>Commands</summary>

```bash
# After creating the certificate chain adjust file ownership to openldap daemon user
chown openldap:openldap server.key server.pem internal-sub-ca.pem

# Move the files to its respectives directories
mv server.pem internal-sub-ca.pem /etc/ssl/certs && mv server.key /etc/ssl/private

# Install acl to a more fine grained permission control
apt update && apt install -y acl

# Adjust read and execution permissions on the /etc/ssl/private directory I'll be using acl
setfacl -m user:openldap:rX /etc/ssl/private
```

</details>

- [ ] Add necessary firewall rules

<details>
    <summary>Commands</summary>

```bash
# Add rule to allow LDAP
nft add rule inet filter input ip saddr 192.168.15.0/24 tcp dport 389 accept

# Add rule to allow LDAPS
nft add rule inet filter input ip saddr 192.168.15.0/24 tcp dport 636 accept

# Make changes persistent
nft list ruleset > /etc/nftables.conf
```

</details>

- [ ] Add Certificates

<details>
    <summary>Commands</summary>

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/certs/intermediate.pem
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/private/server.key
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/certs/server.pem
EOF
```

</details>

- [ ] Set StartTLS/SSL Only

<details>
    <summary>Commands</summary>

```bash
# Force only secure connections
ldapmodify -Q -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=config
changetype: modify
replace: olcLocalSSF
olcLocalSSF: 128
-
replace: olcSecurity
olcSecurity: ssf=128
EOF
```

</details>

- [ ] Test Connection

<details>
    <summary>Commands</summary>

```bash
# Test secure connection on port 389
LDAPTLS_CACERT=/etc/ssl/certs/internal-sub-ca.pem ldapwhoami -H ldap://192.168.15.180 -ZZ -x

# Test secure connection on port 636
LDAPTLS_CACERT=/etc/ssl/certs/intermediate-sub-ca.pem ldapwhoami -H ldaps://192.168.15.180 -x

# Both commands should return
anonymous
```

</details>



</details>
