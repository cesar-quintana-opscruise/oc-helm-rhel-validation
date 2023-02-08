# Opscruise helmchart installation readme

### Introduction

This chart installs the [OpsCruise](https://www.opscruise.com/) Autonomous Application Performance system.

### Prerequisites

#### Versions

| Item         | Version              | Provided |
| ------------ | -------------------- | -------- |
| Kubernetes   | V1.11 +              | n        |
| cAdvisor     | V0.35.0 +            | y        |
| NodeExporter | node-exporter1.1.0 + | y        |
| KSM Exporter | V1.6.0 +             | y        |
| Prometheus   | V2.17.1 +            | y        |
| Helm         | V3.2.4 +             | n        |
| PP Exporter  | V2.4.0 +             | y        |
| PM Exporter  | V0.10.0 +            | y        |

#### Outbound Connections

The following outbound connections must be allowed:

```
<yourname>.opscruise.io:443,9093
auth.opscruise.io:8443
```

#### Credentials
You should have received a yaml file `opscruise-values.yaml`.  
Update your aws credentials and region in the `opscruise-values.yaml` AWS Credentials section.  

###  We do having following tags and conditions:
Added tags and conditions to support running individual opscruise gateway or collector.  
By default all tags are enabled.  

```sh
Tag: opscruise
    conditions:
        awsgw.enabled
        k8sgw.enabled
        promgw.enabled
        opscruise-node-exporter.enabled

Tag: collectors:
    conditions:
        cadvisor.enabled
        kube-state-metrics.enabled
        prometheus.enabled

```



### Installing the Chart
To install the chart with release name `opscruise-bundle`

#### Add the opscruise repo:
helm repo add oc-repo https://opscruise-helm.bitbucket.io

#### Run both opscruise gateways and collectors:
```sh
helm upgrade --install opscruise-bundle oc-repo/opscruise --namespace opscruise --create-namespace -f opscruise-values.yaml

```

#### Run all opscruise gateways alone:
```sh
helm upgrade --install opscruise-bundle oc-repo/opscruise --namespace opscruise --create-namespace --set tags.collectors=false -f opscruise-values.yaml

```

### Run individual gateway(aws-gateway):
```sh
helm upgrade --install oc-awsgw oc-repo/opscruise --namespace opscruise --create-namespace --set tags.collectors=false --set tags.opscruise=false --set awsgw.enabled=true -f opscruise-values.yaml

```

#### Run all opscruise collectors alone:
```sh
helm upgrade --install collectors-bundle oc-repo/opscruise --namespace collectors --create-namespace --set tags.opscruise=false

```

### Run all collector excluding a single collector(promethues collector):

```sh
helm upgrade --install collectors-bundle oc-repo/opscruise --namespace collectors --create-namespace --set tags.opscruise=false --set prometheus.enabled=false

```

### How to pass gcp credentials to gcpgw:

```sh
helm upgrade --install opscruise-bundle oc-repo/opscruise --namespace opscruise --create-namespace --set tags.collectors=false --set prometheus.enabled=false
--set-file gcpgw.gcpCreds=gcpcreds.json

```


### Verify installation
`helm list -A`

### Helm Uninstall
```sh
helm uninstall collectors-bundle -n collectors

helm uninstall opscruise-bundle -n opscruise

```
### Prometheus Postgres Exporter
Q. What is the correct version?

  - Version "2.4.0"
  - Postgres-exporter is frequently updated and released. Postgres helm chart releases are less frequent. So, versioning shouldn't pose much of a problem.

Q. What are the Inputs that needs to be updated/validated by users/customers while deploying?

```
prometheus-postgres-exporter:
  config:
    datasource:
      host: "<POSTGRESQL_SERVICENAME.NAMESPACE.svc.cluster.local>"
      user: "<POSTGRES_EXPORTER_USER>"
      # Only one of password and passwordSecret can be specified
      password: "<POSTGRES_EXPORTER_PASSWORD>"
      # Specify passwordSecret if DB password is stored in secret.
	  passwordSecret: {}
	  #  name: <Secret name>
	  #  key: <Password key inside secret>
	  port: "5432"
      database: ""
      sslmode: disable
    autoDiscoverDatabases: false/true
    excludeDatabases: ['cold']
    includeDatabases: ['wind']
```

Q. One postgres exporter can handle how many PostgresDB instances? How to install/setup postgres exporter?

  - As per the [doc](https://docs.google.com/document/d/1XozuE8xHljOvENQA1wCIsT-xP1gzwO_TRxsBtTJNIbQ/edit#heading=h.30j0zll), its been agreed to install one exporter per DB instance.
  - We can install any number of exporters separately when needed. Detailed steps for installing postgres-exporter is documented in [Setup exporter](https://docs.google.com/document/d/1sk1_Sq1wV1zu7kdjAsOwNWfb5ook6Vb_Tb7i46Rk9rI/edit#heading=h.ssi92vprbodm)

Q. How to enable and disable the exporter by helm command?

  - By default, Postgres Exporter is disabled in Values.yaml
  - We can enable it by adding *"--set prometheus-postgres-exporter.enabled=true"* while running the Helm command

Q. How to create the user for this exporter in postgresdb?

  - Login to the VM/Pod where Postgresdb is running
  - Get into the DB, which we want to monitor and run these commands
```sh
$ psql --host <POSTGRES_INSTANCE> -U postgres -p 5432 -d <DB_NAME>
CREATE EXTENSION pg_stat_statements;
SELECT * FROM pg_available_extensions WHERE name = 'pg_stat_statements' and installed_version is not null;
```
  - Create a file with following contents and replace "<<PASSWORD>>" with any password you need in below file
```sh
$ cat > pg_exporter_user_pass.sql
-- To use IF statements, hence to be able to check if the user exists before
-- attempting creation, we need to switch to procedural SQL (PL/pgSQL)
-- instead of standard SQL.
-- More: https://www.postgresql.org/docs/9.3/plpgsql-overview.html
-- To preserve compatibility with <9.0, DO blocks are not used; instead,
-- a function is created and dropped.
CREATE OR REPLACE FUNCTION __tmp_create_user() returns void as $$
BEGIN
  IF NOT EXISTS (
          SELECT                       -- SELECT list can stay empty for this
          FROM   pg_catalog.pg_user
          WHERE  usename = 'postgres_exporter') THEN
    CREATE USER postgres_exporter;
  END IF;
END;
$$ language plpgsql;

SELECT __tmp_create_user();
DROP FUNCTION __tmp_create_user();

ALTER USER postgres_exporter WITH PASSWORD '<<PASSWORD>>';
ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter,pg_catalog;

-- If deploying as non-superuser (for example in AWS RDS), uncomment the GRANT
-- line below and replace <MASTER_USER> with your root user.
-- GRANT postgres_exporter TO <MASTER_USER>;
CREATE SCHEMA IF NOT EXISTS postgres_exporter;
GRANT USAGE ON SCHEMA postgres_exporter TO postgres_exporter;
GRANT CONNECT ON DATABASE postgres TO postgres_exporter;

CREATE OR REPLACE FUNCTION get_pg_stat_activity() RETURNS SETOF pg_stat_activity AS
$$ SELECT * FROM pg_catalog.pg_stat_activity; $$
LANGUAGE sql
VOLATILE
SECURITY DEFINER;

CREATE OR REPLACE VIEW postgres_exporter.pg_stat_activity
AS
  SELECT * from get_pg_stat_activity();

GRANT SELECT ON postgres_exporter.pg_stat_activity TO postgres_exporter;

CREATE OR REPLACE FUNCTION get_pg_stat_replication() RETURNS SETOF pg_stat_replication AS
$$ SELECT * FROM pg_catalog.pg_stat_replication; $$
LANGUAGE sql
VOLATILE
SECURITY DEFINER;

CREATE OR REPLACE VIEW postgres_exporter.pg_stat_replication
AS
  SELECT * FROM get_pg_stat_replication();

GRANT SELECT ON postgres_exporter.pg_stat_replication TO postgres_exporter;

CREATE OR REPLACE FUNCTION get_pg_stat_statements() RETURNS SETOF pg_stat_statements AS
$$ SELECT * FROM public.pg_stat_statements; $$
LANGUAGE sql
VOLATILE
SECURITY DEFINER;

CREATE OR REPLACE VIEW postgres_exporter.pg_stat_statements
AS
  SELECT * FROM get_pg_stat_statements();

GRANT SELECT ON postgres_exporter.pg_stat_statements TO postgres_exporter;
```
  - Then run this SQL script into the respective DB

```sh
$ psql --host <POSTGRES_INSTANCE> -U postgres -p 5432 -d <DB_NAME> -c '\i ./pg_exporter_user_pass.sql'
```

Q. How to create the password as secret if incase customer prefer to use passwordSecret to store the password?

  - In opscruise-values.yaml, users can either directly specify the password or the Secret and Key name
  - Say for example, user is having a Secret with following details
```sh
apiVersion: v1
kind: Secret
metadata:
  name: pg-db-secret
type: Opaque
data:
  pg-db-pass: YXZnMTIz
```
  - Then in opscruise-values.yaml, user can specify the password or the Secret and Key name like below

##### With Password:
```sh
prometheus-postgres-exporter:
  config:
    datasource:
      host: "<POSTGRESQL_SERVICENAME.NAMESPACE.svc.cluster.local>"
      user: "<POSTGRES_EXPORTER_USER>"
      # Only one of password and passwordSecret can be specified
      password: "test123"
      # Specify passwordSecret if DB password is stored in secret.
	  passwordSecret: {}
	  #  name:
	  #  key:
	  port: "5432"
      database: ''
      sslmode: disable
    autoDiscoverDatabases: false
    excludeDatabases: []
    includeDatabases: []
```
##### With Secret:
```sh
prometheus-postgres-exporter:
  config:
    datasource:
      host: "<POSTGRESQL_SERVICENAME.NAMESPACE.svc.cluster.local>"
      user: "<POSTGRES_EXPORTER_USER>"
      # Only one of password and passwordSecret can be specified
      password: ""
      # Specify passwordSecret if DB password is stored in secret.
	  passwordSecret:
	    name: pg-db-secret
	    key: pg-db-pass
	  port: "5432"
      database: ''
      sslmode: disable
    autoDiscoverDatabases: false
    excludeDatabases: []
    includeDatabases: []
```

### Prometheus Mongodb Exporter
Q. What is the correct version?

  - Chart Version 2.8.1 and App Version 0.10.0

Q. How to create user and password for mongodb exporter and what are all the permissions the user should have?

  - In below example, a user "mongodb_exporter" is created and required READ access has been provided for "admin" DB
For reference:
```
---
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: mongodb-replica-set
  namespace: mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "4.4.0"
  security:
    authentication:
      modes: ["SCRAM"]
  users:
    - name: mongodb_exporter
      db: admin
      passwordSecretRef:
        name: mongodb-ko-secret
      roles:
        - name: clusterAdmin
          db: admin
        - name: userAdminAnyDatabase
          db: admin
      scramCredentialsSecretName: mongodb-replica-set
```

Q. User inputs needed to be validated/added and their default values?

```
prometheus-mongodb-exporter:
  # Only one of mongodb.uri or Secret can be specified
  mongodb:
    uri: "mongodb://${USERNAME}:${PASSWORD}@<SERVICE_NAME.<NAMESPACE>.svc.cluster.local"
  # Name of an externally managed secret (in the same namespace) containing the connection uri as key `mongodb-uri`.
  # If this is provided, the value mongodb.uri is ignored.
  existingSecret:
    name: ""
    key: "mongodb-uri"
  port: "9216"
```

Q. One mongodb exporter can handle how many mongoDB instances? How to install/setup mongodb exporter?

  - Its been agreed to install one exporter per DB instance.
  - We can install any number of exporters separately when needed. Detailed steps for installing mongodb-exporter is documented in [Setup exporter](https://docs.google.com/document/d/1sk1_Sq1wV1zu7kdjAsOwNWfb5ook6Vb_Tb7i46Rk9rI/edit#heading=h.ssi92vprbodm)

Q. How to enable and disable the exporter by helm command?

  - By default, MongoDB Exporter is disabled in Values.yaml
  - We can enable it by adding *"--set prometheus-mongodb-exporter.enabled=true"* while running the Helm command
### Kafka Exporter
Q. What is the correct version?

- Current stable appversion is v1.4.2

Q. What are the Inputs that needs to be updated/validated by users/customers while deploying?

- Users need to provide "kafka" and "zookeeper" service and the their port in opscruise-values.yaml
```
##### Kafka Exporter #####
kafka-exporter:
  args:
    - --kafka.server=<KAFKA_SERVICE>.<NAMESPACE>.svc.cluster.local:<KAFKA_SERVICE_PORT>
    - --zookeeper.server=<ZOOKEEPER_SERVICE>.<NAMESPACE>.svc.cluster.local:<ZOOKEEPER_SERVICE_PORT>
```

Q. How to enable and disable the exporter by helm command?

- By default, Kafka Exporter is disabled in Values.yaml
- We can enable it by adding â€œâ€“set kafka-exporter.enabled=trueâ€ while running the Helm command

Q. How to deploy Kafka Exporter alone?

```
helm upgrade --install opscruise-bundle oc-repo/opscruise -n collectors --create-namespace --set tags.opscruise=false --set tags.collectors=false -f opscruise-values.yaml --set kafka-exporter.enabled=true
```

### Prometheus MySQL Exporter
Q. What is the correct version?

  - Chart Version v1.5.0 and App Version v0.12.1

Q. How to create user and password for mysql exporter and what are all the permissions the user should have?

  - In below example, a user "mysql_exporter" is created and required access has been provided
For reference:
```
CREATE USER 'mysql_exporter'@'%' IDENTIFIED BY '<PASSWORD>' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysql_exporter'@'%';
FLUSH PRIVILEGES;
```

Q. User inputs needed to be validated/added and their default values?

```
##### Prometheus MYSQL Exporter #####
prometheus-mysql-exporter:
  mysql:
    db: "<DB_NAME>"
    host: "<MYSQL_SERVICE>.<NAMESPACE>.svc.cluster.local"
    param: ""
    # If "existingPasswordSecret" is specified, "pass" can be ignored
    pass: ""
    port: 3306
    user: "mysql_exporter"
    # If "pass" is specified, "existingPasswordSecret" can be ignored
    existingPasswordSecret:
      name: ""
      key: ""
```

Q. One mysql exporter can handle how many mysql instances? How to install/setup mysql exporter?

  - Its been agreed to install one exporter per DB instance.
  - We can install any number of exporters separately when needed. Detailed steps for installing mysql-exporter is documented in [Setup exporter](https://docs.google.com/document/d/1sk1_Sq1wV1zu7kdjAsOwNWfb5ook6Vb_Tb7i46Rk9rI/edit#heading=h.ssi92vprbodm)

Q. How to enable and disable the exporter by helm command?

  - By default, MySQL Exporter is disabled in Values.yaml
  - We can enable it by adding *"--set prometheus-mysql-exporter.enabled=true"* while running the Helm command


### Loki Promtail Configuration

  - Loki Promtail configuration to get SysLogs & AppLogs
  - This config can be added in opscruise-values.yaml during the deployment

```
loki-stack:
  promtail:
    extraVolumes:
    - name: syslog
      hostPath:
        path: /var/log/syslog
    extraVolumeMounts:
    - name: syslog
      mountPath: /var/log/syslog
      readOnly: true
    extraScrapeConfigs:
    - job_name: syslog
      pipeline_stages:
      - regex:
          expression: '(?P<month>[^ ]*) (?P<day>[^ ]*) (\d+:\d+:\d+) (?P<hostname>[^ ]*) (?P<syslog_type>[^ ]*)\b(\[(?P<procid>[^ ]*)\])?: .*'
      - labels:
          hostname:
          syslog_type:
          procid:
      static_configs:
      - targets:
        - localhost
        labels:
          job: syslog
          namespace: oslog
          logtype: syslog
          __path__: /var/log/syslog*
```

  - Verify data is visible in North

```
$ curl -G -s  "http://<NORTH_LOKI_POD_IP>:3100/api/prom/query" --data-urlencode 'query={job="syslog"}' | jq | head -n 20
{
  "streams": [
    {
      "labels": "{filename=\"/var/log/syslog\", host=\"ip-172-16-0-36\", job=\"syslog\"}",
      "entries": [
        {
          "ts": "2022-03-10T15:53:03.807918798Z",
          "line": "Mar 10 15:53:03 ip-172-16-0-36 systemd[2511]: run-docker-runtime\\x2drunc-moby-2b02509696ba52c0ac73f10e1abd81617186b4963448e8aec4c27e46e5dbf6c2-runc.hBK5rw.mount: Succeeded."
        },
        {
          "ts": "2022-03-10T15:53:03.807914696Z",
          "line": "Mar 10 15:53:03 ip-172-16-0-36 systemd[1]: run-docker-runtime\\x2drunc-moby-2b02509696ba52c0ac73f10e1abd81617186b4963448e8aec4c27e46e5dbf6c2-runc.hBK5rw.mount: Succeeded."
        }
```
# [South] Pre and Post deployment checks

This bash script can be used to perform Pre and Post deployment checks in South.

## Download script

```sh
> wget https://opscruise-helm.bitbucket.io/ops_sanity_check.sh -O ops_sanity_check.sh
```

## Usage

```sh
> bash ops_sanity_check.sh --help
Usage: bash ops_sanity_check <COMMAND> <SUB_COMMAND>
Available Commands:
    pre-check                 Perform Pre-checks
    post-check                Perform Post-checks
    --help                    Display this text
    --cleanup                 To explicitly run the cleanup. Default action
Available Sub-Commands:
    --disable-cleanup         To disable cleanup
```

## Examples

* `bash ops_sanity_check.sh pre-check`: Perform Pre-checks before deploying `Opscruise` components
* `bash ops_sanity_check.sh post-check`: Perform Post-checks after deploying `Opscruise` components
* `bash ops_sanity_check.sh --cleanup`: Cleanup the environment


## Requirements to run the plugin
* Ensure `kubectl` and `helm` commands are present
* K8s cluster should be accessible using `kubectl` command from where you run this plugin
* Ensure `opscruise-values.yaml` file is present under the location from where you run the plugin


# Steps to deploy Loki with Fluent-bit instead of Promtail


1. Update required Fluent-bit configuration in opscruise-values.yaml


```sh
$ vim opscruise-values.yaml
loki-stack:
  fluent-bit:
    config:
      removeKeys:
        - kubernetes
        - stream
        - filename
      labels: '{logging_agent="fluent-bit"}'
      memBufLimit: 100MB
      labelMap:
        kubernetes:
          container_name: container
          container_image: image
          host: node
          labels:
            app: app
            k8s-app: app
            opscruiseGroup: opscruiseGroup
            opscruisePerimeter: opscruisePerimeter
            opscruiseProduct: opscruiseProduct
            opscruiseStream: opscruiseStream
            pod-template-generation: pod_template_generation
            release: release
          namespace_name: namespace
          pod_name: pod
        stream: stream
        filename: filename
```

2. Deploy using helm


```sh
helm upgrade --install opscruise-bundle oc-repo/opscruise --namespace collectors --create-namespace -f opscruise-values.yaml --version 3.4.7 --set tags.opscruise=false --set jaeger.enabled=false --set loki-stack.promtail.enabled=false --set loki-stack.fluent-bit.enabled=true
```

3. After deploying need to edit ConfigMap of Fluent-bit for a slight change.


```sh
apiVersion: v1
data:
  fluent-bit.conf: |-
    # Add "Path_key" in [INPUT] to get log filename in parsed log stream
    [INPUT]
        Path_key       filename
	# By default, OUTPUT plugin name is "grafana-loki" which is invalid. Change it to "loki"
    [Output]
        Name loki
```

# Steps to deploy InfluxDB Service to expose prometheus metrics


1. By default InfluxDB itself exposes prometheus metrics and we don't need any additional service/exporters to be deployed. So, users can just add a label "opscruisePerimeter: opscruise" to the existing/default InfluxDB service present in the K8s cluster. (Preferred)

2. If users don't want to disturb/modify their K8s service, then we can deploy our service to expose the metrics as "common_dns" endpoint type

```sh
vim opscruise-values.yaml
##### InfluxDB Exporter #####
influxdb-exporter:
  enabled: false
  # Options for "endpoint_type"
  # common_dns: If InfluxDB is running within cluster but different namespace/externally on a VM and is accessible with DNS
  # external_ip: If InfluxDB is running externally on a VM and is accessible with IP address only
  endpoint_type: common_dns
  # If "endpoint_type" is common_dns, provide DNS_NAME and PORT below
  common_dns_name: "influx-service.influx.svc.cluster.local" ## this is just an example
  common_dns_port: "8086"
  # If "endpoint_type" is external_ip, provide ID_ADDRESS and PORT below
  external_ip_address: ""
  external_ip_port: ""

  serviceLabels: {}
  annotations: {}
```

```sh
helm upgrade --install opscruise-bundle oc-repo/opscruise --namespace collectors --create-namespace -f opscruise-values.yaml --version 3.5.0 --set tags.opscruise=false --set jaeger.enabled=false --set influxdb-exporter.enabled=true
```

3. If the InfluxDB is running as Systemd service on a VM, then we can deploy our service to expose the metrics as "external_ip" endpoint type

```sh
vim opscruise-values.yaml
##### InfluxDB Exporter #####
influxdb-exporter:
  enabled: false
  # Options for "endpoint_type"
  # common_dns: If InfluxDB is running within cluster but different namespace/externally on a VM and is accessible with DNS
  # external_ip: If InfluxDB is running externally on a VM and is accessible with IP address only
  endpoint_type: external_ip
  # If "endpoint_type" is common_dns, provide DNS_NAME and PORT below
  common_dns_name: ""
  common_dns_port: ""
  # If "endpoint_type" is external_ip, provide ID_ADDRESS and PORT below
  external_ip_address: "1.2.3.4"
  external_ip_port: "8086"

  serviceLabels: {}
  annotations: {}
```

```sh
helm upgrade --install opscruise-bundle oc-repo/opscruise --namespace collectors --create-namespace -f opscruise-values.yaml --version 3.5.0 --set tags.opscruise=false --set jaeger.enabled=false --set influxdb-exporter.enabled=true
```

# Steps to deploy x509 Certificate exporter


1a. Use the following configuration to monitor TLS certs that are present in the form of K8s secrets

```sh
vim opscruise-values.yaml
##### x509 Certificate Exporter #####
x509-certificate-exporter:
  enabled: true
  service:
    create: false
  secretsExporter:
    enabled: true
    secretTypes:
    - type: kubernetes.io/tls
      key: tls.crt
  prometheusServiceMonitor:
    create: false
  prometheusPodMonitor:
    create: false
  prometheusRules:
    create: false
  hostPathsExporter:
    daemonSets:
      master: null
      nodes: null
```

1b. Use the following configuration to monitor TLS certs that are present on the hosts

```sh
vim opscruise-values.yaml
##### x509 Certificate Exporter #####
x509-certificate-exporter:
  enabled: true
  service:
    create: false
  secretsExporter:
    enabled: false
  prometheusServiceMonitor:
    create: false
  prometheusPodMonitor:
    create: false
  prometheusRules:
    create: false
```

2. Deploy the exporter using Helm command

```sh
helm upgrade --install opscruise-bundle oc-repo/opscruise --namespace collectors --create-namespace -f opscruise-values.yaml --version <HELM_VERSION> --set tags.opscruise=false --set jaeger.enabled=false --set x509-certificate-exporter.enabled=true
```