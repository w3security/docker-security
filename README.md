Docker security configuration

Before deploying to a production environment, you should replace the demo security certificates and configuration YAML files with your own. With the RPM and Debian installations, you have direct access to the file system, but the Docker image requires modifying the Docker Compose file to include the replacement files.

Additionally, you can set the Docker environment variable DISABLE_INSTALL_DEMO_CONFIG to true. This change completely disables the demo installer.
Sample Docker Compose file

```bash
version: '3'
services:
  odfe-node1:
    image: amazon/opendistro-for-elasticsearch:1.13.3
    container_name: odfe-node1
    environment:
      - cluster.name=odfe-cluster
      - node.name=odfe-node1
      - discovery.seed_hosts=odfe-node1,odfe-node2
      - cluster.initial_master_nodes=odfe-node1,odfe-node2
      - bootstrap.memory_lock=true # along with the memlock settings below, disables swapping
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" # minimum and maximum Java heap size, recommend setting both to 50% of system RAM
      - network.host=0.0.0.0 # required if not using the demo security configuration
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536 # maximum number of open files for the Elasticsearch user, set to at least 65536 on modern systems
        hard: 65536
    volumes:
      - odfe-data1:/usr/share/elasticsearch/data
      - ./root-ca.pem:/usr/share/elasticsearch/config/root-ca.pem
      - ./node.pem:/usr/share/elasticsearch/config/node.pem
      - ./node-key.pem:/usr/share/elasticsearch/config/node-key.pem
      - ./admin.pem:/usr/share/elasticsearch/config/admin.pem
      - ./admin-key.pem:/usr/share/elasticsearch/config/admin-key.pem
      - ./custom-elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./internal_users.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/internal_users.yml
      - ./roles_mapping.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/roles_mapping.yml
      - ./tenants.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/tenants.yml
      - ./roles.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/roles.yml
      - ./action_groups.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/action_groups.yml
    ports:
      - 9200:9200
      - 9600:9600 # required for Performance Analyzer
    networks:
      - odfe-net
  odfe-node2:
    image: amazon/opendistro-for-elasticsearch:1.13.3
    container_name: odfe-node2
    environment:
      - cluster.name=odfe-cluster
      - node.name=odfe-node2
      - discovery.seed_hosts=odfe-node1,odfe-node2
      - cluster.initial_master_nodes=odfe-node1,odfe-node2
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - network.host=0.0.0.0
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - odfe-data2:/usr/share/elasticsearch/data
      - ./root-ca.pem:/usr/share/elasticsearch/config/root-ca.pem
      - ./node.pem:/usr/share/elasticsearch/config/node.pem
      - ./node-key.pem:/usr/share/elasticsearch/config/node-key.pem
      - ./admin.pem:/usr/share/elasticsearch/config/admin.pem
      - ./admin-key.pem:/usr/share/elasticsearch/config/admin-key.pem
      - ./custom-elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./internal_users.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/internal_users.yml
      - ./roles_mapping.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/roles_mapping.yml
      - ./tenants.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/tenants.yml
      - ./roles.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/roles.yml
      - ./action_groups.yml:/usr/share/elasticsearch/plugins/opendistro_security/securityconfig/action_groups.yml
    networks:
      - odfe-net
  kibana:
    image: amazon/opendistro-for-elasticsearch-kibana:1.13.3
    container_name: odfe-kibana
    ports:
      - 5601:5601
    expose:
      - "5601"
    environment:
      ELASTICSEARCH_URL: https://odfe-node1:9200
      ELASTICSEARCH_HOSTS: https://odfe-node1:9200
    volumes:
      - ./custom-kibana.yml:/usr/share/kibana/config/kibana.yml
    networks:
      - odfe-net

volumes:
  odfe-data1:
  odfe-data2:

networks:
  odfe-net:
  
  ```

Then make your changes to elasticsearch.yml. For a full list of settings, see Security. This example adds (extremely) verbose audit logging:


```bash
opendistro_security.ssl.transport.pemcert_filepath: node.pem
opendistro_security.ssl.transport.pemkey_filepath: node-key.pem
opendistro_security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
opendistro_security.ssl.transport.enforce_hostname_verification: false
opendistro_security.ssl.http.enabled: true
opendistro_security.ssl.http.pemcert_filepath: node.pem
opendistro_security.ssl.http.pemkey_filepath: node-key.pem
opendistro_security.ssl.http.pemtrustedcas_filepath: root-ca.pem
opendistro_security.allow_default_init_securityindex: true
opendistro_security.authcz.admin_dn:
  - CN=A,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA
opendistro_security.nodes_dn:
  - 'CN=N,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
opendistro_security.audit.type: internal_elasticsearch
opendistro_security.enable_snapshot_restore_privilege: true
opendistro_security.check_snapshot_restore_write_privileges: true
opendistro_security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
cluster.routing.allocation.disk.threshold_enabled: false
opendistro_security.audit.config.disabled_rest_categories: NONE
opendistro_security.audit.config.disabled_transport_categories: NONE

  ```

Use this same override process to specify new authentication settings in ```bash /usr/share/elasticsearch/plugins/opendistro_security/securityconfig/config.yml ```, as well as new internal users, roles, mappings, action groups, and tenants.

To start the cluster, run ```bash docker-compose up ```.

If you encounter any```bash File /usr/share/elasticsearch/config/elasticsearch.yml ```has insecure file permissions (should be 0600) messages, you can use ```bash chmod ``` to set file permissions before running ```bash docker-compose up ```. Docker Compose passes files to the container as-is.

Finally, you can open Kibana at http://localhost:5601, sign in, and use the Security panel to perform other management tasks.
Using certificates with Docker

To use your own certificates in your configuration, add all of the necessary certificates to the volumes section of the Docker Compose file:
```bash
volumes:
- ./root-ca.pem:/full/path/to/certificate.pem
- ./admin.pem:/full/path/to/certificate.pem
- ./admin-key.pem:/full/path/to/certificate.pem
#Add other certificates

```

After replacing the demo certificates with your own, you must also include a custom elasticsearch.yml in your setup, which you need to specify in the volumes section.

```bash

volumes:
#Add certificates here
- ./custom-elasticsearch.yml: /full/path/to/custom-elasticsearch.yml

```

Remember that the certificates you specify in your Docker Compose file must be the same as the certificates listed in your custom elasticsearch.yml file. At a minimum, you should replace the root, admin, and node certificates with your own. For more information about adding and using certificates, see Configure TLS certificates.

```bash

opendistro_security.ssl.transport.pemcert_filepath: new-node-cert.pem
opendistro_security.ssl.transport.pemkey_filepath: new-node-cert-key.pem
opendistro_security.ssl.transport.pemtrustedcas_filepath: new-root-ca.pem
opendistro_security.ssl.http.pemcert_filepath: new-node-cert.pem
opendistro_security.ssl.http.pemkey_filepath: new-node-cert-key.pem
opendistro_security.ssl.http.pemtrustedcas_filepath: new-root-ca.pem
opendistro_security.authcz.admin_dn:
  - CN=admin,OU=SSL,O=Test,L=Test,C=DE
  
 ```

To start the cluster, run ```bash docker-compose up ``` as usual.
