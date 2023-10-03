- [x] change the initial config of ldap to squash the changes
- [x] add commands to add the base dn (for those who didnt use dpkg-reconfigure slapd)
- [x] Erase the merge blob text on the readme.md
- [x] Add command to test ssl and starttls connections from the server
- [ ] add entries related to sni register in the server certificate

### SAMBA + OpenLDAP Backend

- [x] apt install samba smbldap-tools
- [x] cp /etc/samba/smb.conf /etc/samba/smb.conf.old
- [x] ldapadd -Y EXTERNAL -H ldapi:/// -f /usr/share/doc/samba/examples/samba.ldif
- [x] smbldap-config # consult /etc/smbldap-tools/smbldap.conf
- [x] smbldap-populate
- [x] edit /etc/samba/smb.conf acordingly, adding ldap and encryption entries
- [x] smbpasswd -W # set ldap admin password to tdb.secrets
- [x] Obs: any user added to ldap need to be added to system as 'system user' preferably
- [x] Obs: be aware of default user gid modifications, it tends to break smbldap-useradd

Take a look on the 'idmap' samba config to map correctly the uid and gid mapped in the ldap and system

- [x] Remove public key creation (SSH) and certificate chain (OpenLDAP) steps
- [x] Move firewall rule creation (OpenLDAP) higher
- [x] Review the test connection step (OpenLDAP) because it's not working with ssl enabled and anonymous bind disabled
- [ ] Make the configuration files available in the repository root or exclusive folder