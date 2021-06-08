 # 389ds

This role installs the 389 Directory Server (389ds) on RedHat or CentOS servers.

## Features
* Installation of
  * Standalone LDAP servers
  * Supplier nodes (incl. multi master configurations)
  * Consumer nodes
* Setup of TLS and encryption settings
  * Management of supported ciphers
  * Enforce minimal and maximal TLS version
  * Interface for certificate requests and renewals
* Support for multiple nsslapd instances per server
* Support for multiple suffixes per instance
* Management of any plugin (configure plugin attributes, enable/disable plugin on all or some named hosts)
* Add custom schema files
* LDAP settings under in these containers are Ansible managed:
  * cn=config
  * cn=config,cn=ldbm database,cn=plugins,cn=config
  * cn=bdb,cn=config,cn=ldbm database,cn=plugins,cn=config
* Creation of LDAP indexes
* Creation of VLV indexes
* Management of replication topology

## Requirements
* Ansible 2.9 or newer
* CentOS 8.3 or RHEL 8.3


## Example

> **Note:** You should consider to embed the 389ds role in your own host_ldap role, which contains the company specific configuration (e.g. schema files, certificate scripts and your tested 389ds configuration). 

```yaml
# this variable defines the list of ldap supplier nodes
# it is used in the ds_app variable and not by the role itself 
host_ldap_supplier:
  - name: 'master1.example.com'
    replicaid: 1
    initialmaster: true
  - name: 'master2.example.com'
    replicaid: 2
  - name: 'master3.example.com'
    replicaid: 3
    
# this variable holds the whole configuration
ds_app:
  # DN for directory manager
  dm_dn: 'cn=directory manager'

  # Password for directory manager
  dm_pasword: 'my_secret'

  # Here you can define one or more 389ds instances
  instance:
    # Instance name
    - name: 'example'

      # Overwrites the password for this instance
      dm_password: 'my_secret_for_this_instance'

      # Password for the replication manager account (not necessary for standalone LDAP servers)
      initial_replication_manager_password: 'repl_pw'
      secure_port: 636
      port: 389
      tls: 'on'

      encryption:
        selfsigned_certs: false
        ca_files:
          - '{{ host_ldap_role_path }}/templates/certs/ca_files/MyCompanyRootCA.cer'

        request_certscript: '{{ host_ldap_role_path }}/templates/certs/bin/request_cert.sh'
        get_certscript: ''{{ host_ldap_role_path }}/templates/certs/bin/get_cert.sh'

        directory_certs: '{{ host_ldap_role_path }}/templates/certs/hosts'

        certificate:
          alternates:
            - "{{ inventory_hostname.split('.', 1)[0] }}"
            - "ldap1.example.com"
            - "{{ inventory_hostname }}"

          subject: 'CN={{ inventory_hostname }},O=My Company,L=Klagenfurt,ST=Kärnten,C=AT'
          renew_before_exp_days: "180"


        attributes:
          nsSSL3Ciphers: '+all,-fortezza_null,-fortezza,-rsa_null_sha,-fortezza_rc4_128_sha,-rsa_null_md5,-TLS_RSA_WITH_AES_128_GCM_SHA256,-TLS_RSA_WITH_AES_128_GCM_SHA256'
          sslVersionMin: 'TLS1.0'
          nsSSLClientAuth: 'allowed'
          nsTLS1: 'on'

      # Schema files for this instance
      custom_schema_files:
        - '{{ host_ldap_role_path }}/templates/schema/94ssh.ldif'
        - '{{ host_ldap_role_path }}/templates/schema/98kerberos.ldif'

      # settings for cn=config
      config:
        nsslapd-ssl-check-hostname: 'on'
        nsslapd-maxdescriptors: 16384
        #nsslapd-sizelimit: -1
        #nsslapd-pagedsizelimit: 0
        nsslapd-sizelimit: "{{ nsslapd_sizelimit | default(1000) }}"
        nsslapd-pagedsizelimit: "{{ nsslapd_pagedsizelimit | default(1001) }}"
        nsslapd-timelimit: -1
        nsslapd-idletimeout: 3600
        nsslapd-ndn-cache-max-size: 209715200
        nsslapd-auditlog-logging-enabled: 'on'
        nsslapd-errorlog-logging-enabled: 'on'
        nsslapd-accesslog-logging-enabled: 'on'
        nsslapd-auditlog-maxlogsperdir: 10
        nsslapd-auditlog-logexpirationtime: 4
        nsslapd-auditlog-logmaxdiskspace: 1000
        nsslapd-auditlog-logminfreediskspace: 500
        nsslapd-errorlog-level: 16384
        nsslapd-errorlog-maxlogsperdir: 10
        nsslapd-errorlog-logexpirationtime: 4
        nsslapd-errorlog-logmaxdiskspace: 1000
        nsslapd-errorlog-logminfreediskspace: 500
        nsslapd-accesslog-maxlogsperdir: 10
        nsslapd-accesslog-logexpirationtime: 4
        nsslapd-accesslog-logmaxdiskspace: 1000
        nsslapd-accesslog-logminfreediskspace: 500
        nsslapd-pwpolicy-local: 'on'
        passwordLockout: 'on'
        passwordExp: 'on'
        passwordHistory: 'on'
        passwordCheckSyntax: 'on'
        passwordMaxAge: 31536000
        passwordMinLength: 15
        passwordWarning: 0
        passwordInHistory: 5
        passwordMinCategories: 4
        passwordLockoutDuration: 1800
        passwordMaxFailure: 10

      # settings for cn=config,cn=ldbm database,cn=plugins,cn=config
      ldbm_database:
        config:
          nsslapd-lookthroughlimit: -1
          nsslapd-idlistscanlimit: 25000
        # settings for cn=bdb,cn=config,cn=ldbm database,cn=plugins,cn=config
        bdb:
          # see RH ticket #02857953  
          nsslapd-db-locks: 100000
          nsslapd-import-cache-autosize: 20

      # Plugin configuration. The name must match the plugin name in 389ds
      plugin:
        - name: 'Retro Changelog Plugin'
          attributes:
            nsslapd-changelogmaxage: '7d'
          status:
            enabled: "{{ host_ldap_supplier | map(attribute='name') | list }}"
            disabled: []

        - name: 'MemberOf Plugin'
          attributes:
            memberofgroupattr: uniqueMember
            memberofattr: memberOf
            memberOfEntryScope: 'dc=example,dc=com'
          status:
            enabled: "{{ host_ldap_supplier | map(attribute='name') | list }}"

      # suffix configuration for the instance
      suffix:
        - name: 'dc=example,dc=com'

          replication:
            supplier:
              server: "{{ host_ldap_supplier }}"
              excluded_attrs:
                - memberOf
                - accountUnlockTime
                - passwordRetryCount
                - retryCountResetTime
              changelogmaxage: '30d'
            consumer:
              inventorygroup: ldapreplicas
              excluded_attrs:
                - accountUnlockTime
                - passwordRetryCount
                - retryCountResetTime
            replicapurgedelay: 604800

          db:
            # database name for the suffix
            # at the moment this role supports only one database for a suffix
            - name: userroot

              # Here you can specify additional indexes
              index:
                - name: automountinformation
                  type:
                    - eq
                    - pres
                    - sub

                - name: homedirectory
                  type:
                    - eq
                    - pres

                - name: ipserviceport
                  type:
                    - eq
                    - pres

                - name: automountmapname
                  type:
                    - eq
                    - pres

                - name: sudouser
                  type:
                    - eq
                    - pres
                    - sub

              # also VLV indexes are allowed
              # make sure, that the index base exists
              vlv_index:
                - name: objectclass
                  filter: '(objectClass=*)'
                  base: 'dc=example,dc=com'
                  scope: 1
                  index:
                  - name: getobjectclass
                    sort: 'cn'

                - name: hosts
                  filter: '(objectClass=ipHost)'
                  base: 'ou=hosts,{{ host_ldap_basedn }}'
                  scope: 1
                  index:
                  - name: gethostent
                    sort: 'cn uid'

                - name: networks
                  base: 'ou=networks,{{ host_ldap_basedn }}'
                  scope: 1
                  filter: '(objectClass=ipNetwork)'
                  index:
                    - name: getnetent
                      sort: 'cn uid'

                - name: group
                  base: 'ou=group,{{ host_ldap_basedn }}'
                  scope: 1
                  filter: '(objectClass=posixGroup)'
                  index:
                    - name: getgrent
                      sort: 'cn uid'

                - name: passwd
                  base: 'ou=people,{{ host_ldap_basedn }}'
                  scope: 1
                  filter: '(objectClass=posixAccount)'
                  index:
                    - name: getpwent.uid
                      sort: uid
                    - name: getpwent
                      sort: 'cn uid'
```


## Dependencies


## License

GNU General Public License v3.0


## Author Information

Maintainer: Horst Thaller
