# MinIO configured with LDAP

### I run the FreeIPA (IdM) Identity Management Platform for LDAP(s) within my redcloud.land domain. In these steps, I configure MinIO using the 'mc' client to support LDAP. 
##### NOTE: Once you set LDAP, local user accounts are disabled

1. DOWNLOAD MINIO CLIENT
  * wget https://dl.min.io/client/mc/release/linux-amd64/mc
  * chmod +x mc
  * mv mc /usr/local/bin/
2. I copied my ldap CA and installed it (Fedora)
  * scp ldap-ca.crt minio:/tmp
  * ssh minio
  * mv /tmp/ldap-ca.crt /etc/pki/ca-trust/source/anchors/
  * update-ca-trust
3. Set an system alias, you will use this alies to interact with minio server.
  *  mc alias set $aliasname $minioURL $admin_user $admin_password \
    * example: mc alias set gitminio https://minio.redcloud.land minioadmin minioadmin
4. Configure ldap for FreeIPA
  ```
   mc admin config set gitminio/ identity_ldap \
   tls_skip_verify="on" \
   server_addr="ldap.redcloud.land:636" \
   lookup_bind_dn="uid=readonly,cn=users,cn=accounts,dc=redcloud,dc=land" \
   lookup_bind_password="SoMePaSsWoRd" \
   user_dn_search_base_dn="cn=users,cn=accounts,dc=redcloud,dc=land" \
   user_dn_search_filter="(uid=%s)" \
   group_search_base_dn="cn=gitadmins,cn=groups,cn=accounts,dc=redcloud,dc=land;cn=gitusers,cn=groups,cn=accounts,dc=redcloud,dc=land" \
   group_search_filter="(&(objectclass=groupOfNames)(member=%d))"
   ```
   ##### Your LDAP settings may be differnent, find the MinIO ldap doc's [here](https://docs.min.io/minio/baremetal/reference/minio-mc-admin/mc-admin.config.html#)
 5. Set an admin policy for your admin group `gitadmins`
   * mc admin policy set gitminio/ consoleAdmin group='cn=gitadmins,cn=groups,cn=accounts,dc=redcloud,dc=land'
 6. Set a rw policy for your user group `gitusers`
   * mc admin policy set gitminio/ readwrite,diagnostics group='cn=gitusers,cn=groups,cn=accounts,dc=redcloud,dc=land'
 7. Restart ldap
   * mc admin service restart gitminio/

Now, visit your login and test your ldap credentials!
