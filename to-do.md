- [x] change the initial config of ldap to squash the changes
- [x] add commands to add the base dn (for those who didnt use dpkg-reconfigure slapd)
- [x] Erase the merge blob text on the readme.md
- [x] Add command to test ssl and starttls connections from the server
- [ ] add entries related to sni register in the server certificate

### SAMBA + OpenLDAP Backend

- [x] apt install samba smbldap-tools
- [x] cp /etc/samba/smb.conf /etc/samba/smb.conf.old
- [ ] ldapadd -Y EXTERNAL -H ldapi:/// -f /usr/share/doc/samba/examples/samba.ldif
- [ ] smbldap-config # consult /etc/smbldap-tools/smbldap.conf
- [ ] smbldap-populate
- [ ] edit /etc/samba/smb.conf acordingly, adding ldap and encryption entries
- [ ] smbpasswd -W # set ldap admin password to tdb.secrets
- [ ] Obs: any user added to ldap need to be added to system as 'system user' preferably
- [ ] Obs: be aware of default user gid modifications, it tends to break smbldap-useradd

Take a look on the 'idmap' samba config to map correctly the uid and gid mapped in the ldap and system