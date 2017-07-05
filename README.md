# ELK Ansible role

Installs the [ELK stack](https://www.elastic.co/products) (Elasticsearch, Logstash, and Kibana)
for log aggregation and monitoring. Intended for integration with [Riemann](http://riemann.io)
for alerting functionality.

## Requirements

* [freedomofpress.generate-ssl-cert] role
* 2GB of RAM for the logserver
* logclients to ship logs

## Role variables

```yaml
elk_kibana_user: "kibana"
elk_kibana_logfile: "/var/log/kibana.log"

# Provide ability to disable the snapshot functionality. It's not well
# tested, so leaving false as the default now. If set to true on a first
# run, probably should add `meta: flush_handlers` prior to running to ensure
# the `path.repo:` variable is recognized by the running elasticsearch service..
elk_elasticsearch_snapshot: false

elk_elasticsearch_snapshot_directory: /var/lib/elasticsearch/backups
elk_elasticsearch_snapshot_repository: es_backup
elk_elasticsearch_snapshot_initialization:
  type: fs
  settings:
    location: "{{ elk_elasticsearch_snapshot_directory }}"
    compress: yes
  _hack: null

# It'd be nice to use iso8601 instead epoch, but the ElasticSearch API
# throws an invalid_snapshot_name error with the iso8601 format.
elk_elasticsearch_snapshot_name: "snapshot-{{ ansible_date_time.epoch }}"

# Limits to set in /etc/security/limits.conf. Make sure to copy the entire
# list if overriding any of the individual elements.
elk_elasticsearch_pam_limits:
    - domain: elasticsearch
      limit_item: memlock
      limit_type: hard
      value: unlimited

    - domain: elasticsearch
      limit_item: memlock
      limit_type: soft
      value: unlimited

    - domain: elasticsearch
      limit_item: nofile
      limit_type: soft
      value: 65535

    - domain: elasticsearch
      limit_item: nofile
      limit_type: hard
      value: 65535

# Riemann plugin for alerting, de-dot filter for ElasticSearch v2 compatibility.
# See: https://www.elastic.co/blog/introducing-the-de_dot-filter
elk_logstash_plugins:
  - logstash-output-riemann
  - logstash-filter-de_dot

# Interface used for firewall restrictions and IPv4 lookups
elk_network_interface: eth0

elk_cluster_name: elk-logging

# SSL is disabled by default. Set these vars to the fullpaths to SSL
# certs you wish to use, and Nginx will force HTTPS connections.
# You must place the SSL certs there in a separate play.
elk_nginx_ssl_certificate: ""
elk_nginx_ssl_certificate_key: ""
elk_nginx_server_name: localhost

# Not safe for production use! Override to secure logins.
elk_kibana_username: kibana
elk_kibana_password: kibana

# Override to change the landing page, e.g. a custom dashboard:
# "dashboard/Your-Dashboard-Name". You must replace whitespace in
# dashboard names with hyphens, since Kibana expects it.
elk_kibana_default_app: discover

# Enable automatic configuration of IP whitelisting for "logclients".
# Uses ufw. Disable if you're using a different role for firewall config.
elk_configure_firewall: true

# Allow downstream playbooks to utilize custom webserver configuration
# Set this to false in order to skip over this role's nginx rollout
elk_configure_nginx: true

# Allow downstream playbooks to override patterns and filters fileglob list
elk_logstash_patterns:
  - logstash-patterns/*
elk_logstash_filters:
  - logstash-configs/*

# Declare fileglob of GeoIP databases to copy. Off by default.
elk_logstash_geoipdbs: []

# Sets up Curator to automatically delete old Logstash indices and save
# disk space. Off by default.
elk_configure_curator: false

# Indexes older than this number of days will be deleted on a weekly basis
elk_curator_days_to_keep: 90
```

## Usage

Use the role in a playbook like this:

```yaml
- hosts: logserver
  roles:
    - role: elk
      elk_kibana_username: admin
      elk_kibana_password: WowWhatAStrongPassword4
```

## Adding visualizations

The role does not yet create Kibana visualizations automatically. You can however
import saved visualizations that ship with the role in `files/kibana-dashboards.json`.
Go to **Settings -> Objects -> Import** in the Kibana UI and browse to the JSON file.

## Running the tests

This role uses [Molecule] and [ServerSpec] for testing. To use it:

```
pip install molecule
gem install serverspec
molecule test
```

You can also run selective commands:

```
molecule idempotence
molecule verify
```

See the [Molecule] docs for more info.

## Further reading
### Setup

* [Official ElasticSearch docs](https://www.elastic.co/guide/index.html)
* [UCLA ELK configuration](https://www.itsecurity.ucla.edu/elk)
* [DigitalOcean guide to setting up ELK](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elk-stack-on-ubuntu-14-04)

### Developing custom filters
* [Grok Debugger](http://grokdebug.herokuapp.com/)
* [Grok Constructor](http://grokconstructor.appspot.com/)
* [How to develop Logstash configuration files](http://blog.comperiosearch.com/blog/2015/04/10/how-to-develop-logstash-configuration-files/)

See the [examples/writing-filters](examples/writing-filters) directory in this repo
for a preconfigured development environment. Copy that directory to a server with
logstash installed, or use a Vagrant testing VM.

### Maintenance

* [Elasticsearch Curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/command-line.html)
* [Elasticsearch zero-downtime reindexing](https://blog.codecentric.de/en/2014/09/elasticsearch-zero-downtime-reindexing-problems-solutions/)

## License
MIT

[Molecule]: http://molecule.readthedocs.org/en/master/
[ServerSpec]: http://serverspec.org/
[freedomofpress.generate-ssl-cert]: https://github.com/freedomofpress/ansible-role-generate-ssl-cert
