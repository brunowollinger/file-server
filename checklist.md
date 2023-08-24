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

### Networking

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
    address 192.168.0.10/24 # IP address / Subnet mask format
    gateway 192.168.0.1

# Save the file and restart the service
systemctl restart networking
```

</details>

- [ ] Basic Firewall Setup

<details>
    <summary>Commands</summary>

```bash
# Backup the default nftables config file
cp /etc/nftables.conf /etc/nftables.conf.old

# Enable and start nftables
systemctl enable nftables && systemctl start nftables

# Add SSH rule to prevent lockout to the server
nft add rule inet filter input ip saddr 192.168.0.0/24 tcp dport 22 accept

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
```

</details>

<hr>

### SSH Hardening

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
scp ~/.ssh/id_rsa.pub user@192.168.0.10:~/.ssh/

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
