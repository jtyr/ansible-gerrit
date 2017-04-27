gerrit
======

Ansible role which helps to install and configure Gerrit.

The configuration of the role is done in such way that it should not be
necessary to change the role for any kind of configuration. All can be
done either by changing role parameters or by declaring completely new
configuration as a variable. That makes this role absolutely
universal. See the examples below for more details.

Please report any issues or send PR.


Examples
--------

```
---

- name: Example of how to install Gerrit with default configuration
  hosts: all
  roles:
    - gerrit

- name: Gerrit with PostgreSQL, GitWeb, LDAP and Nginx
  hosts: all
  vars:
    # Use PostgreSQL as the DB
    gerrit_db_engine: postgresql
    # Change authentication type to LDAP
    gerrit_config_auth_type: LDAP
    # Customize the gerrit.config
    gerrit_config__custom:
      # Listen only on the local network interface
      httpd:
        listenUrl: proxy-http://localhost:8080
      # Configure GitWeb
      gitweb:
        type: gitweb
        cgi: /var/www/git/gitweb.cgi
      # Configure LDAP
      ldap:
        server: ldap://ldap.example.com
        accountBase: ou=people,dc=example,dc=com
        accountPattern: "(&(objectClass=person)(uid=${username}))"
        accountFullName: displayName
        accountEmailAddress: mail
        groupBase: ou=groups,dc=example,dc=com
        groupMemberPattern: "(&(objectClass=group)(member=${dn}))"
    # Install some core plugins
    gerrit_core_plugins:
      - hooks
    # Install some other plugins
    gerrit_other_plugins:
      - https://gerrit-ci.gerritforge.com/view/Plugins-stable-{{ gerrit_release }}/job/plugin-delete-project-stable-{{ gerrit_release }}/lastSuccessfulBuild/artifact/buck-out/gen/plugins/delete-project/delete-project.jar
      - https://gerrit-ci.gerritforge.com/view/Plugins-stable-{{ gerrit_release }}/job/plugin-reviewers-stable-{{ gerrit_release }}/lastSuccessfulBuild/artifact/buck-out/gen/plugins/reviewers/reviewers.jar
    # Configure Nginx
    nginx_vhost_config__default:
      default:
        - server:
            - listen 80
            - location /:
                - proxy_pass http://localhost:8080
  roles:
    - postgresql
    - gitweb
    - gerrit
    - nginx
```


Role variables
--------------

```
# Gerrit version
gerrit_version: 2.14

# Gerrit release
gerrit_release: "{{ gerrit_version | regex_replace('^(\\d+\\.\\d+).*', '\\1') }}"

# URL to the Gerritn WAR file
gerrit_war_url: https://www.gerritcodereview.com/download/gerrit-{{ gerrit_version }}.war

# Whether to validation SSL certs (RHEL6 have problem to validate)
gerrit_war_validate_certs: "{{
  true
    if ansible_distribution_major_version | int > 6
    else
  false }}"

# Gerrit user and group
gerrit_user: gerrit
gerrit_group: gerrit

# Path to the main Gerrit directory
gerrit_home: /opt/gerrit

# Path to the Gerrit site directory
gerrit_site: "{{ gerrit_home }}/site"

# Path to the downloads directory
gerrit_downloads: "{{ gerrit_home }}/downloads/{{ gerrit_release }}"

# Packages required for the PostgreSQL DB creation
gerrit_db_pgsql_pkgs:
  - python-psycopg2

# Packages required for the MySQL DB creation
gerrit_db_mysql_pkgs:
  - MySQL-python

# Default list of extra packages
gerrit_deps_pkgs__default:
  - java

# Custom list of extra packages
gerrit_deps_pkgs__custom: []

# Final list of extra packages
gerrit_deps_pkgs: "{{
  (
    gerrit_db_pgsql_pkgs
      if gerrit_db_engine == 'postgresql'
      else
    gerrit_db_mysql_pkgs
      if gerrit_db_engine == 'mysql'
      else
    []
  ) +
  gerrit_deps_pkgs__default +
  gerrit_deps_pkgs__custom }}"

# Service name
gerrit_service: gerrit

# Service file location
gerrit_service_path: /etc/init.d/{{ gerrit_service }}

# List of Gerrit plugins to install
gerrit_core_plugins: []

# List of all available core plugins
# (get the list from: java -jar gerrit.war init --list-plugins)
gerrit_core_plugins_available:
  - commit-message-length-validator
  - download-commands
  - hooks
  - replication
  - reviewnotes
  - singleusergroup

# List of URLs for other plugins
gerrit_other_plugins: []

# Whether to validation SSL certs
gerrit_other_plugins_validate_certs: "{{
  true
    if ansible_distribution_major_version | int > 6
    else
  false }}"

# DB engine [h2|mysql|postgresql]
gerrit_db_engine: h2

# User used to create DB and user
gerrit_db_login_user: "{{
  'postgres'
    if gerrit_db_engine == 'postgresql'
    else
  'root' }}"

# Password used to create DB and user
gerrit_db_login_password: "{{
  'postgres'
     if gerrit_db_engine == 'postgresql'
     else
  None }}"

# DB server configuration options
gerrit_db_host: localhost
gerrit_db_port: "{{
  5432
    if gerrit_db_engine == 'postgresql'
    else
  3306 }}"
gerrit_db_name: gerrit
gerrit_db_user: gerrit
gerrit_db_password: gerrit


# Values of the default options of the gerrit section of the main config
gerrit_config_gerrit_base_path: git
gerrit_config_gerrit_canonical_web_url: http://localhost:8080/
gerrit_config_gerrit_server_id: "{{ ansible_machine_id }}"

# Default options of the gerrit section of the main config
gerrit_config_gerrit__default:
  basePath: "{{ gerrit_config_gerrit_base_path }}"
  canonicalWebUrl: "{{ gerrit_config_gerrit_canonical_web_url }}"
  serverId: "{{ gerrit_config_gerrit_server_id }}"

# Custom options of the gerrit section of the main config
gerrit_config_gerrit__custom: {}

# Custom options of the gerrit section of the main config
gerrit_config_gerrit: "{{
  gerrit_config_gerrit__default.update(gerrit_config_gerrit__custom) }}{{
  gerrit_config_gerrit__default }}"


# Default options for the H2 DB of the database section of the main config
gerrit_config_database__h2:
  type: h2
  database: "{{ gerrit_site}}/db/ReviewDB"

# Default options for the MySQL and PostgreSQL DB of the database section of the main config
gerrit_config_database__sql:
  type: "{{ gerrit_db_engine }}"
  hostname: "{{ gerrit_db_host }}"
  port: "{{ gerrit_db_port }}"
  username: "{{ gerrit_db_user }}"
  database: "{{ gerrit_db_name }}"

# Default options of the database section of the main config
gerrit_config_database__default: "{{
  gerrit_config_database__sql
    if gerrit_db_engine in ['mysql', 'postgresql']
    else
  gerrit_config_database__h2 }}"

# Custom options of the database section of the main config
gerrit_config_database__custom: {}

# Database section of the main config
gerrit_config_database: "{{
  gerrit_config_database__default.update(gerrit_config_database__custom) }}{{
  gerrit_config_database__default }}"


# Values of the Auth section of the main config
gerrit_config_auth_type: OpenID

# Auth section of the main config
gerrit_config_auth:
  type: "{{ gerrit_config_auth_type }}"


# Values of the options in the container section of the main config
gerrit_config_container_java_home: /usr/lib/jvm/jre

# Default options of the container section of the main config
gerrit_config_container__default:
  user: "{{ gerrit_user }}"
  javaHome: "{{ gerrit_config_container_java_home }}"

# Custom options of the container section of the main config
gerrit_config_container__custom: {}

# Final container section of the main config
gerrit_config_container: "{{
  gerrit_config_container__default.update(gerrit_config_container__custom) }}{{
  gerrit_config_container__default }}"


# Values of the default options of the cache section of the main config
gerrit_config_cache_directory: cache

# Default options of the cache section of the main config
gerrit_config_cache__default:
    directory: "{{ gerrit_config_cache_directory }}"

# Custom options of the cache section of the main config
gerrit_config_cache__custom: {}

# Final options of the cache section of the main config
gerrit_config_cache: "{{
  gerrit_config_cache__default.update(gerrit_config_cache__custom) }}{{
  gerrit_config_cache__default }}"


# Default sections of the main config
gerrit_config__default:
  gerrit: "{{ gerrit_config_gerrit }}"
  database: "{{ gerrit_config_database }}"
  auth: "{{ gerrit_config_auth }}"
  container: "{{ gerrit_config_container }}"
  cache: "{{ gerrit_config_cache }}"

# Custom sections of the main config
gerrit_config__custom: {}

# Final sections of the main config
gerrit_config: "{{
  gerrit_config__default.update(gerrit_config__custom) }}{{
  gerrit_config__default }}"


# Values of the default options of the auth section of the secure config
gerrit_secure_auth_register_email_private_key: "{{ ansible_machine_id | b64encode}}"

# Default options of the auth section of the secure config
gerrit_secure_auth__default:
  registerEmailPrivateKey: "{{ gerrit_secure_auth_register_email_private_key }}"

# Custom options of the auth section of the secure config
gerrit_secure_auth__custom: {}

# Final options of the auth section of the secure config
gerrit_secure_auth: "{{
  gerrit_secure_auth__default.update(gerrit_secure_auth__custom) }}{{
  gerrit_secure_auth__default }}"


# Default options of the database section of the secure config
gerrit_secure_db__default:
  password: "{{ gerrit_db_password }}"

# Custom options of the database section of the secure config
gerrit_secure_db__custom: {}

# Final database section of the secure config
gerrit_secure_db: "{{
  gerrit_secure_db__default.update(gerrit_secure_db__custom) }}{{
  gerrit_secure_db__default }}"

# Default sections of the secure config
gerrit_secure__default:
  auth: "{{ gerrit_secure_auth }}"
  database: "{{
    gerrit_secure_db
      if gerrit_db_engine in ['mysql', 'postgresql']
      else
    {}
  }}"

# Custom sections of the secure config
gerrit_secure__config: {}

# Final secure config
gerrit_secure: "{{
  gerrit_secure__default.update(gerrit_secure__config) }}{{
  gerrit_secure__default }}"


# Default service defaults
gerrit_defaults__default:
  gerrit_site: "{{ gerrit_site }}"
  gerrit_war: "{{ gerrit_home }}/gerrit.war"
  no_start: 0
  start_stop_daemon: 1

# Custom service defaults
gerrit_defaults__custom: {}

# Final service defaults
gerrit_defaults: "{{
  gerrit_defaults__default.update(gerrit_defaults__custom) }}{{
  gerrit_defaults__default }}"
```


Dependencies
------------

- [`config_encoder_filters`](https://github.com/jtyr/ansible-config_encoder_filters)
- [`gitweb`](https://github.com/jtyr/ansible-gitweb) (optional)
- [`postgresql`](https://github.com/jtyr/ansible-postgresql) (optional)
- [`nginx`](https://github.com/jtyr/ansible-nginx) (optional)


License
-------

MIT


Author
------

Jiri Tyr
