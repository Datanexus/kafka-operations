# Managing Connectors

The [modfy-connectors.yml](../modify-connectors.yml) playbook in this repository can be used to modify the connectors that are deployed to a Kafka instance or cluster. Specifically, the playbook in this playbook can be used to:

* Add new connector plugins to the nodes in the cluster
* Manage workers in the connector framework on those nodes, and
* Manage the connectors on on the workers that are running within that connector framework

We will describe these capabilities more in the sections that follow. The only assumption made is that the target nodes for the playbook run are actually members of a Kafka cluster and that the code for the connector framework was included in the deployment of that cluster (this is true, by default, for all recent versions of Kafka).

## Loading Connector Plugins

The playbook in this repository supports the deployment of a bundle containing the code for a given connector plugin in any of the following formats:

* As an archive file containing the JAR file for the connector (and the JAR files for all of its dependencies, if there are any), where those JAR files are all contained in a single subdirectory
* As a package in an RPM file
* As a single JAR file that contains the code for the plugin and all of its dependencies

Which format is best will likely depend on the connector in question and how it was built. There are a number of connectors that include an RPM file as part of their distribution. Others (for example the connectors that are built from Scala source code) will automatically build what they call an **Assembly file** (or what Confluent refers to as an **Uber JARfile**, a single JAR file that contains the class files for the connector code and all of it's dependencies. The last format supported (and the first in the list, above) is that of an archive file (a gzipped tarfile or tarfile) that contains the JAR file for the connector code itself and separate JAR files for all of that connector's dependencies. In this case, the only assumption is that all of these JAR files are bundled in a single subdirectory in that archive file.

The easiest way to add a new connector to all of the nodes that make up the cluster is to first create a custom configuration file. If our custom configuration file looks something like this:

```bash
$ cat config-load-plugins.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when configuring the connectors
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# define the action_list for this operation
action_list:
  - action: load
    plugin_list:
      - name: kafka-connect-solr
        local_file: '/tmp/kafka-connect-solr/kafka-connect-solr-0.1.7.rpm'
      - name: kafka-connect-cassandra
        local_file: '/tmp/kafka-connect-cassandra/kafka-connect-cassandra-assembly-0.0.7.jar'
```

Then the `ansible-playbook` command to deploy those two plugins to the target nodes of the playbook run would look something like this:

```bash
ansible-playbook -e "{ \
    config_file: 'config-load-plugins.yml' \
}" modify-connectors.yml
```

Note that the deployment is actually driven by the parameters that are defined in the `action_list` variable shown above. The first parameter in the `action_list` entry shown above declares the action that should be taken (loading plugins) and the second parameter defines a list of dictionary entries containing the names of the plugins that should be loaded (`kafka-connect-solr` for the first connector shown above) and the local file on the Ansible host that the connector plugin code should be loaded from (the playbook also supports specifying this as a `url` instead of a `local_file`; which you use will likely depend on the connector plugin that you are adding to the cluster and the environment that the cluster is running in).

The other fields in this custom configuration file should be familiar to those of you who have already used a custom configuration file to control the deployment of Kafka to one or more target nodes, and their values should match the values that were used when the target Kafka cluster was provisioned intially. If you need more details on the definitions of these parameters, you can find a description of each of them [here](Supported-Config-Params.md).

The fields required for a given `action_list` entry will vary depending on the action being defined by that entry, so it's worth summarizing the supported fields for loading new plugins here:

* **`action`**: the name of the action to take
* **`plugin_list`**: a list of dictionary entries, each of which contains the information needed to perform the defined action on a single plugin; the playbook will iterate over each of the entries to this list, deploying each of the plugins in turn based on the information provided in the dictionary (hash map) for that entry. The fields currently supported for these entries include the following:
    * **`name`**: the name to use when deploying the plugin in question; this name will be used as the name of the directory created on the target machine to contain the code for that connector plugin, so it should be unique. While this field is required, it is not used when installing a connector as a package from an RPM file
    * **`local_file`**: the name of the local file (on the Ansible host) containing the code for the named connector plugin; if this parameter is defined, then the named file will be copied over and installed locally on each of the target nodes
    * **`url`**: the URL that should be used to download the file containing the code for the named connector; if this parameter is defined, then the named file will be downloaded to each of the target nodes and installed locally; it should be noted that if both the `local_file` and `url` are specified for the named connector, then the `local_file` will be used

If an `action_list` is not provided, or if any of the fields mentioned above are not included in that `action_list` or any of the defined `plugin_list` entries, then no action will be taken during the playbook run.

### Supported bundle file formats

As was mentioned previously, the playbook in this repository supports a number of different formats when it comes to the files referenced by the `local_file` or `url` parameters shown above. Specifically, the following file formats are supported:

* An archive file containing the JAR file for the connector (and the JAR files for all of its dependencies, if there are any), where those JAR files are all contained in a single subdirectory; this archive would typically be a tarfile or gzipped tarfile
* A package contained in an RPM file
* A single JAR file that contains the code for the plugin and all of its dependencies

The installation process for each of these file types is described more completely, below

### Installation from an archive file

If the connector plugin is provided as an archive file, the assumption is that the file in question is a tarfile (or a gzipped tarfile), and that the contents of that archive file contain the JAR file for the connector and the JAR files for any other packages that it depends on in a single subdirectory. For example, here are contents of the `kafka-connect-solr` plugin when it is distributed as a gzipped tarfile:

```bash
$ tar ztvf /tmp/kafka-connect-solr-0.1.7.tar.gz
drwxrwxr-x  0 vagrant vagrant     0 Apr 26 14:43 ./kafka-connect-solr/
-rw-rw-r--  0 vagrant vagrant 1007970 Apr 25 14:09 ./kafka-connect-solr/solr-solrj-6.3.0.jar
-rw-rw-r--  0 vagrant vagrant  208700 Apr 25 14:09 ./kafka-connect-solr/commons-io-2.5.jar
-rw-rw-r--  0 vagrant vagrant  720931 Apr 25 14:09 ./kafka-connect-solr/httpclient-4.4.1.jar
-rw-rw-r--  0 vagrant vagrant  322234 Apr 25 14:09 ./kafka-connect-solr/httpcore-4.4.1.jar
-rw-rw-r--  0 vagrant vagrant   40631 Apr 25 14:09 ./kafka-connect-solr/httpmime-4.4.1.jar
-rw-rw-r--  0 vagrant vagrant  792964 Apr 25 14:09 ./kafka-connect-solr/zookeeper-3.4.6.jar
-rw-rw-r--  0 vagrant vagrant  161867 Apr 25 14:09 ./kafka-connect-solr/stax2-api-3.1.4.jar
-rw-rw-r--  0 vagrant vagrant  486013 Apr 25 14:09 ./kafka-connect-solr/woodstox-core-asl-4.4.1.jar
-rw-rw-r--  0 vagrant vagrant   26720 Apr 25 14:09 ./kafka-connect-solr/noggit-0.6.jar
-rw-rw-r--  0 vagrant vagrant   16519 Apr 25 14:09 ./kafka-connect-solr/jcl-over-slf4j-1.7.7.jar
-rw-rw-r--  0 vagrant vagrant   29257 Apr 25 14:09 ./kafka-connect-solr/slf4j-api-1.7.7.jar
-rw-rw-r--  0 vagrant vagrant   70982 Apr 26 12:13 ./kafka-connect-solr/connect-utils-0.2.63.jar
-rw-rw-r--  0 vagrant vagrant 1170801 Apr 25 14:09 ./kafka-connect-solr/jackson-databind-2.6.3.jar
-rw-rw-r--  0 vagrant vagrant   46968 Apr 25 14:09 ./kafka-connect-solr/jackson-annotations-2.6.0.jar
-rw-rw-r--  0 vagrant vagrant  258875 Apr 25 14:09 ./kafka-connect-solr/jackson-core-2.6.3.jar
-rw-rw-r--  0 vagrant vagrant 2256213 Apr 25 14:09 ./kafka-connect-solr/guava-18.0.jar
-rw-rw-r--  0 vagrant vagrant 1493680 Apr 25 14:09 ./kafka-connect-solr/freemarker-2.3.25-incubating.jar
-rw-rw-r--  0 vagrant vagrant   16495 Apr 26 14:43 ./kafka-connect-solr/kafka-connect-solr-0.1.7.jar
```

Furthermore, there is an assumption that the filename of the archive file includes a suffix that indicates its file type in a standard way (eg. `.tar`, `.tar.gz`, or `.tgz`). A combination of the suffix information for the file and it's MIME type will be used to determine how best to handle the file during the connector installation process, and if the suffix doesn't match the MIME type, then the file in question will be skipped.

If the MIME type and filename suffix do match, then the archive file will be unpacked and the JAR files that it contains will be copied over to a directory that the playbook run creates based on the `name` that was specified in the `plugin_list` entry for that connector plugin in the `action_list`. The location of that directory will depend on the `kafka_distro` that was specified in the local variables file (if this parameter is not specified, then the playbook run will assume that the target nodes are part of a Confluent Kafka cluster, and the connector code will be copied over to a subdirectory of the `/usr/share/java` subdirectory; if, on the other hand, this parameter is defined and has a value of `apache`, then the connector code will be copied over to a subdirectory of the `{kafka_dir}/libs` directory).

### Installation from a package in an RPM file

If the connector is provided to the playbook as an RPM file, then the `package` module will be used to install that package locally on each of the target machines. The target directory where the code will be placed will depend on the target defined in the RPM file; there is no attempt make to map this target directory to a more "appropriate" directory baseed on the Kafka distribution thas been deployed to the target nodes.

### Installation from a JAR file

If the connector is provided to the playbook as a JAR (Java ARchive) file, then that file will simply be copied over to a directory that the playbook run creates based on the `name` that as specified in the `plugin_list` entry for that connector plugin in the `action_list`. As was the case with the archive file scenario (described above), the location of that subdirectory will depend on the Kafka distribution that was deployed to the target nodes.

## Managing Workers in the Connector Framework

Once the desired connector plugins have been installed locally on the nodes that make up our cluster, the next step necessary (before we can deploy instances of those connectors locally) is to start workers in the connector framework on our target nodes (or restart the workers running the connector framework if we are adding new connectors to the framework and there are already workers running locally). Fortunately, the [modfy-connectors.yml](../modify-connectors.yml) playbook in this repository supports the management of the workers running in the connector framework on a set of one or more target nodes via an `ansible-playbook` run. As was the case with adding new connector plugins to our nodes (above), a custom configuration file can be created to control this process. Here is an example of such a file:

```bash
$ cat config-start-workers.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when configuring the connectors
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# define the action_hash for this operation
action_list:
  - action: start-workers
    worker_list:
      - name: kafka-connect-solr
        connector_name: kafka-connect-solr
        group_id: solrcloud-group
        worker_port: 8084
```

As was the case with the previous example (above), we are using the parameters defined in the `action_list` in this file to control the actions taken during the playbook run. In this case, the `action` parameter in our `action_list` has been set to `start-workers`, which indicates that we should invoke the tasks involved in starting a new worker (or a new set of workers) in the connector framework. When managing the worker instances running in the connector framework using the playbook in this repository, the `action` parameter in the `action_list` can take any of the following values:

* **`start-workers`**: if they are not running already, then the named workers will be started in the connector framework (in distributed mode)
* **`stop-workers`**: if they are running, then the named workers will be stopped in connector framework
* **`restart-workers`**: if they are running, then the named workers will be stopped, then started again in the connector framework; there is a pause of about 5 seconds imposed between starting and stopping the named workers to ensure these two tasks do not run to quickly (i.e. to ensure that the workers have had time to stop completely before they are restarted again)

When starting a worker instance in the connector framework, there are a number of configuration parameters (in addition to the `action` parameter shown above) that can be added to the `action_list` entry for that worker in order to specify how the worker instance should be configured:

* **`key_converter`**: the class to use when serializing key Connect data; defaults to the `org.apache.kafka.connect.json.JsonConverter` class if not specified
* **`value_converter`**: the class to use when serializing value Connect data; defaults to the `org.apache.kafka.connect.json.JsonConverter` class if not specified
* **`internal_key_converter`**: the class to use internally when serializing key Connect data that implements the `Converter` interface (used to convert data like offsets and configs); defaults to the `org.apache.kafka.connect.json.JsonConverter` class if not specified
* **`internal_value_converter`**: the class to use internally when serializing value Connect data that implements the `Converter` interface (used to convert data like offsets and configs); defaults to the `org.apache.kafka.connect.json.JsonConverter` class if not specified
* **`offset_topic`**: the Kafka topic to store offset data for connectors in; defaults to `connect-offsets` if not specified
* **`config_topic`**: the Kafka topic to store connector and task configuration data in; defaults to `connect-configs` if not specified
* **`status_topic`**: the Kafka topic to store status updates for connectors and tasks in; defaults to `connect-status` if not specified
* **`rest_port`**: the port that the worker's REST API should listen on; defaults to `8083` if not specified
* **`group_id`**: a unique string that identifies the Connect cluster group that this worker belongs to; defaults to `default_group` if not specified

It should be noted here that when starting a new worker instance in the connector framework, the topics named in the configuration will be created automatically, if they do not currently exist, as part of the `ansible-playbook` run. It should also be noted that when starting a new worker instance, if there is already an instance running at the named port that response to a `GET /connectors` RESTful query, the playbook will not attempt to restart that instance.

Stopping or restarting a worker instance is a much simpler process. The only parameter from the list shown above that is important for these two commands is the `name` parameter. If we use a custom configuration file that looks something like the following in our `ansible-playbook` run:

```bash
$ cat config-stop-workers.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when configuring the connectors
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# define the action_hash for this operation
action_list:
  - action: stop-workers
    worker_list:
      - name: kafka-connect-solr
```

then the process corresponding to the worker instance named `kafka-connect-solr` will be stopped. Similarly, a file that looks something like this:

```bash
$ cat config-restart-workers.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when configuring the connectors
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# define the action_hash for this operation
action_list:
  - action: restart-workers
    worker_list:
      - name: kafka-connect-solr
```

can be used to restart those same instances (first stopping them, then starting them up again). It should be noted that any other parameters that are included in the `action_list` entries useed to stop or restart worker instances will be silently ignored (there is no attempt made to rewrite the worker configuration file that is used when either stopping or restarting worker instances on the target nodes).

## Managing Connectors

Finally, in addition to adding new connnector plugins to and managing workers running in the connector framework on the target nodes of an `ansible-playbook` run, the playbook in this repository also supports management of the connectors that are deployed to workers running in that framework. More specifically, the playbook in this repository supports the following `action` values in the defined `action_list`:

* **`create`**: creates a new (named) instance of a given connector on one of the workers in the connector framework
* **`restart`**: restarts a running (named) instance of a connector running on one of the workers in the connector framework
* **`update`**: updates the configuration of an existing (named) instance of a connector running on one of the workers in the connector framework
* **`pause`**: pauses an existing (named) instance of a connector running on one of the workers in the connector framework; this action is useful when the underlying system that the connector communicates with must be taken down for maintenance, for example
* **`resume`**: resumes an existing (named, paused) instance of a connector running on one of the workers in the connector framework
* **`delete`**: deletes an existing (named) instance of a connector from one of the workers in the connector framework

Each of these actions relies on a `name` that must be defined as part of the `action_list` ehtries that are passed into the playbook run (each containing one of the `action` values described above). In addition, when creating a new instance of a given connector or updating the configuration of that connector, a `config` hash map must be provided. The details of that `config` hash map will vary greatly depending on the connector type and the specific implementation that is being used. For example, when creating a new instance of the `kafka-connect-solr` instance available provided by [jcustenborder](https://github.com/jcustenborder/kafka-connect-solr), a local variables file that looks something like this would be used:

```bash
cat config-create-connectors.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when configuring the connectors
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# define the action_list for this operation
action_list:
  # - action: update
  - action: create
    connector_list:
      - name: solrcloud-metrics
        worker_port: 8084
        config:
          "connector.class": "com.github.jcustenborder.kafka.connect.solr.CloudSolrSinkConnector"
          topics: metrics
          "collection.name": "kafka_metrics"
          "tasks.max": 1
          "solr.zookeeper.hosts": "{{((zk_nodes | default(['localhost'])) | join(':2181,')) + ':2181'}}"
          "solr.zookeeper.chroot": "/lwfusion/3.1.2/solr"
          "solr.commit.within": 1000
          "solr.ignore.unknown.fields": false
      - name: solrcloud-logs
        worker_port: 8084
        config:
          "connector.class": "com.github.jcustenborder.kafka.connect.solr.CloudSolrSinkConnector"
          topics: logs
          "collection.name": "kafka_logs"
          "tasks.max": 1
          "solr.zookeeper.hosts": "{{((zk_nodes | default(['localhost'])) | join(':2181,')) + ':2181'}}"
          "solr.zookeeper.chroot": "/lwfusion/3.1.2/solr"
          "solr.commit.within": 1000
          "solr.ignore.unknown.fields": false
```

As was the case when adding new connectors to the nodes, the `action_list` used when creating new instance of a given connector type relies on an `action` (which was given a value of `create` in this example) and an array of `connector_list` entries, each of which contains a `name` to use when creating the connector instance, the `worker_port` of the worker where we'll be starting that connector instance, and a `config` containing the configuration values required for that connector type. Other than the `connector.class` field that is shown, above, everything else in that `config` hash is implementation specific and the values that are supported and/or required will vary from one connector implementation to another (even for connectors that are intended to talk wtih the same sort of subsystem, a Solr server for example).

The parameters in each of the `connectors` entries will be used to construct the POST request that will be used to create the named connector instance. It should be noted here that since the RESTful API provided by the workers running in the connector framework must already be up and running for this POST request to succeed, we are assuming in this example that worker is already running in the the connector framework on the target nodes and is listening on the named `worker_port`. If the worker is not already running in the connector framework, then any attempt to create new connector instances will fail.

Updating the configuration of an existing connector (or a set of existing connectors) looks quite similar. In that case, the only real change is in the `action` value that is passed in via the `action_list`; for example, the following local variables file could be used to increase the time the connectors wait before committing changes in the named Kafka topics to the named Solr collection from one to two seconds:

```bash
cat config-update-connectors.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when configuring the connectors
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# define the action_list for this operation
action_list:
  # - action: update
  - action: create
    connector_list:
      - name: solrcloud-metrics
        worker_port: 8084
        config:
          "connector.class": "com.github.jcustenborder.kafka.connect.solr.CloudSolrSinkConnector"
          topics: metrics
          "collection.name": "kafka_metrics"
          "tasks.max": 1
          "solr.zookeeper.hosts": "{{((zk_nodes | default(['localhost'])) | join(':2181,')) + ':2181'}}"
          "solr.zookeeper.chroot": "/lwfusion/3.1.2/solr"
          "solr.commit.within": 2000
          "solr.ignore.unknown.fields": false
      - name: solrcloud-logs
        worker_port: 8084
        config:
          "connector.class": "com.github.jcustenborder.kafka.connect.solr.CloudSolrSinkConnector"
          topics: logs
          "collection.name": "kafka_logs"
          "tasks.max": 1
          "solr.zookeeper.hosts": "{{((zk_nodes | default(['localhost'])) | join(':2181,')) + ':2181'}}"
          "solr.zookeeper.chroot": "/lwfusion/3.1.2/solr"
          "solr.commit.within": 2000
          "solr.ignore.unknown.fields": false
```

The remaining actions (restarting, pausing, resuming, and deleting named connector instances) are much simpler since they only require that the name of the connector and worker port be specified in the `connector_list` entries; for example this local variables file can be used to pause the two connector instances shown above:

```bash
cat config-pause-connectors.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when configuring the connectors
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# define the action_list for this operation
action_list:
  - action: pause
    connector_list:
      - name: solrcloud-metrics
        worker_port: 8084
      - name: solrcloud-logs
        worker_port: 8084
```

resuming, restarting, or removing these same instances would only require that the `action` shown above be modified (to `resume`, `restart`, or `delete`, respectively).
