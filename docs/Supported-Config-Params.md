# Supported configuration parameters
The playbooks in the [modify-topics.yml](../modify-topics.yml) and [modify-connectors.yml](../modify-connectors.yml) files in this repository pull in a set of default values for many of the configuration parameters that are needed to deploy Kafka from the [vars/kafka-operations.yml](../vars/kafka-operations.yml) file and the default configuration file (the [config.yml](../config.yml) file). The parameters defined in these files define a reasonable set of defaults for operations against a fairly generic Kafka deployment, either to a single node or a cluster, including defaults for the path to the log directory used by the Kafka cluster, the username and group that Kafka was installed under, and the ports where the Kafka services are listening.

While you may never need to change most of these values from their defaults, there are quite a few of these parameters, so a brief summary of what each is and how it is used could be helpful. In this section, we summarize all of these options, breaking them out into:

* parameters used to control the Ansible playbook run
* parameters used during the playbook run, and
* parameters used to control the operations taken on the nodes of the targeted Kafka cluster

Each of these sets of parameters are described in their own section, below.

## Parameters used to control the playbook run
The following parameters can be used to control the `ansible-playbook` run itself, defining things like how Ansible should connect to the nodes involved in the playbook run, which nodes should be targeted, where the Kafka distribution should be downloaded from, which packages must be installed during the deployment process, and where those packages should be obtained from:

* **`cloud`**: this parameter is used to indicate the target cloud for the deployment (either `aws` or `osp`); this controls both the role that is used to create new nodes (when a matching set of nodes does not exist in the target environment) and how the [build-app-host-groups](../roles/build-app-host-groups) role retrieves the list of target nodes for the deployment; if unspecified this parameter defaults to the `aws` value specified in the default configuration file
* **`region`**: this parameter is used to indicate the region that should be searched for matching nodes (and, if no matching nodes are found, the region in which at set of nodes should be created for use as a Kafka cluster); if unspecified the default value of `us-west-2` specified in the [config.yml](../config.yml) file is used
* **`zone`**: this parameter is used to indicate the availability zone that should be used when creating new nodes in an OpenStack environment; since this parameter is not needed for AWS deployments, there is no default value for this parameter (and any value provided during an AWS deployment will be silently ignored)
* **`tenant`**: this parameter is used to indicate the tenant name to use, either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; if unspecified this parameter defaults to the `datanexus` value specified in the default configuration file
* **`project`**: this parameter is used to indicate the project name to use, either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; if unspecified this parameter defaults to the `demo` value specified in the default configuration file
* **`dataflow`**: this parameter is used to indicate the dataflow name to use, either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; the dataflow tag is used to link together the clusters/ensembles (Cassandra, Zookeeper, Kafka, Solr, etc.) that are involved in a given dataflow; if this value is not specified, it defaults to a value of `none` during the playbook run
* **`domain`**: this parameter is used to indicate the domain name to use (eg. test, production, preprod), either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; if unspecified this parameter defaults to the `production` value specified in the default configuration file
* **`cluster`**: this parameter is used to indicate the cluster name to use, either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; this value is used to differentiate clusters of the same type from each other when multiple clusters are deployed for a given application for the same tenant, project, dataflow, and domain; if this value is not specified it defaults to a value of `a` during the playbook run
* **`user`**: the username that should be used when connecting to the target nodes via SSH; the value for this parameter will likely change from one target environment to the next; if unspecified a value of `centos` will be used
* **`config_file`**: used to define the location of a *configuration file* (see the discussion of this topic, below); this file is a YAML file containing definitions for any of the configuration parameters that are described in this section and is more than likely a file that will be created to manage the process of creating a specific ensemble. Storing the settings for a given ensemble in such a file makes it easy to guarantee that all of the nodes in that ensemble are configured consistently. If a value is not specified for this parameter then the default configuration file (the [config.yml](../config.yml) file) will be used; to override this behavior (and not load a configuration file of any kind), one can simply set the value of this parameter to `/dev/null` and specify all of the other, non-default parameters that are needed as extra variables during the playbook run
* **`private_key_path`**: used to define the directory where the private keys are maintained when the inventory for the playbook run is being managed dynamically; in these cases, the scripts used to retrieve the dynamic inventory information will return the names of the keys that should be used to access each node, and the playbook will search the directory specified by this parameter to find the corresponding key files. If this value is not specified then the current working directory will be searched for those keys by default


## Parameters used during the playbook run
These parameters are used to control the playbook run, defining things like the directory where the distribution was unpacked into (if Apache Kafka is being deployed), and the list of topics that should be created at the conclusion of the playbook run.

* **`kafka_dir`**: the path to the directory that the (Apache) Kafka distribution should be unpacked into; defaults to `/opt/kafka` if unspecified
* **`kafka_distro`**: the name of the distribution to install/configure. Only `apache` or `confluent` are supported; defaults to `confluent` if not specified
* **`kafka_user`**: the username under which Kafka should be installed and run; defaults to `kafka`
* **`kafka_group`**: the name of the user group under which Kafka should be installed and run; defaults to `kafka`
* **`internal_subnet`**: the CIDR block describing the subnet that any nodes being created by the playbook run should attach as a private network (`eth0`); this network is used for internode communications between the nodes of the cluster being deployed (and between nodes of the clusters/ensembles that make up the dataflow that this cluster is a member of); if it is not specified, then the default value of `10.10.1.0/24` from the [config.yml](../config.yml) file is used; if the deployment is an OpenStack deployment then a value for the associated **`internal_uuid`** parameter must also be provided, and that value must be the UUID for an existing internal network in the targeted OpenStack environment
* **`external_subnet`**: the CIDR block describing the subnet that any nodes being created by the playbook run should attach as a "public" network (`eth1`); this network is used to support client connections to the various services that make up the cluster being deployed (and between nodes of the clusters/ensembles that make up the dataflow that this cluster is a member of); if it is not specified, then the default value of `10.10.2.0/24` from the [config.yml](../config.) file is used; if the deployment is an OpenStack deployment then a value for the associated **`external_uuid`** parameter must also be provided, and that value must be the UUID for an existing external network in the targeted OpenStack environment

## Parameters used to control the operations taken
These parameters are used control the operations taken during a playbook run, defining things like the interfaces that Kafka should be listening on for requests and the directory where Kafka should store its data.

* **`kafka_data_dir`**: the name of the directory used by Kafka to store its data; defaults to `/data` if unspecified; this parameter is required when removing topics from the cluster and otherwise is silently ignored; it should also be noted here that the value for this parameter must match the value that was used for this same parameter when the target Kafka node/cluster was deployed
* **`schema_registry_port`**: the port that the schema registry service is listening on (if enabled); defaults to port 8081 if unspecified; this parameter should be set to the same value that was used when the Kafka cluster was deployed
* **`rest_port`**: the port used to access the Connector framework's REST API; defaults to 8083 if not specified
* **`action_list`**: an array of dictionary entries, where each entry in the list is an action that should be taken during the course of the playbook run; valid actions supported by the [modify-topics.yml](modify-topics.yml) and [modify-connectors.yml](modify-connectors.yml) (and the parameters that must be defined for each) are described in detail, below.

### Actions supported when modifying/managing topics

The [modify-topics.yml](modify-topics.yml) playbook supports the following actions:

* **`create-topics`**: used to create new topics in the cluster
* **`delete-topics`**: used with the [modify-topics.yml](modify-topics.yml) playbook to remove existing topics from the cluster

### Actions supported when modifying/managing connectors

The [modify-connectors.yml](modify-connectors.yml) playbook is slightly more complicated than the  [modify-topics.yml](modify-topics.yml) playbook, since it has to support the entire connector/worker lifecycle, including:

* adding connectors to the cluster
* creating new instances of a connector, managing those instances over time, and removing them when they are no longer needed
* creating workers for connector instances to handle user requests, managing those workers over time, and removing them when they are no longer needed

As such, the actions supported by the [modify-connectors.yml](modify-connectors.yml) playbook can be broken down into three groups (each with their own associated parameters).

#### Loading plugins into the cluster

The single action associated with this process is an action that can be used to load the bundled code for a connector plugin to an existing Kafka cluster:

* **`load`**: used to add a connector to the cluster (from an gzipped tarfile or package containing the code needed to instantiate that connector)

To accomplish this, the user must provide a list of connector plugins that should be added to the cluster as part of the `action_list` in their custom configuration file. The parameters that must be included in each entry in the `plugin_list` included in each `action_list` entry include the following:

* **`name`**: the name that should be used for this connector when adding it to the cluster; this name must be unique amongst the connectors that are added to the repository, since a directory will be created in the `plugin.path` of the cluster instances and the corresponding archive or package file will be unpacked into that directory; if there is already a connector with the same name provided here then an error will be thrown by the playbook run
* **`local_file`**: the path (on the Ansible host) to the archive or package containing the bundled code for the connector plugin; if both a `local_file` and a `url` (see below) are provided, then the `local_file` will be used
* **`url`**: a URL that points to the archive or package containing the bundled code for the connector plugin; if both a `local_file` (see above) and a `url` are provided, then the `local_file` will be used

An example of a custom configuration file that might be used to install the `kafka-connect-cassandra` connector might look something like this:

```
$ cat config-load-plugin.yml
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
# define the action_list for this playbook run
action_list:
  - action: load
    plugin_list:
      - name: kafka-connect-solr
        local_file: '/Users/tjmcs/tmp/local-connector-files/kafka-connect-solr/kafka-connect-solr-0.1.7.rpm'
```

#### Managing connectors

* **`create`**: used to create a new instance of a connector withing the cluster
* **`restart`**: used to restart a running connector instance
* **`update`**: used to update the configuration of a running connector instance
* **`pause`**: used to pause a running connector instance
* **`resume`**: used to resume a paused connector instance
* **`delete`**: used to delete an existing connector instance

#### Managing workers

* **`start**`**: used to start a worker for a given connector instance
* **`restart**`**: used to restart the given worker instance
* **`stop**`**: used stop the given worker instance

## A note on interface names
The playbooks in this repository will dynamically determine the names of the interfaces that correspond to the defined  `internal_subnet` and `external_subnet` CIDR block values and configure the members of the cluster being deployed to listen on those interfaces, either for communication between the nodes that make up the cluster or for client requests. This is accomplished by dynamically constructing an `iface_description_array` parameter within the playbook, then using that parameter to determine the names of the corresponding interfaces and their IP addresses.

Put quite simply, the `iface_description_array` lets you specify a description for each of the networks that you are interested in, then retrieve the names of those networks on each machine in a variable that can be used elsewhere in the playbook. To accomplish this, the `iface_description_array` is defined as an array of hashes (one per interface), each of which include the following fields:

* **`type`**: the type of description being provided, currently only the `cidr` type is supported
* **`val`**: a value describing the network in question; since only `cidr` descriptions are currently supported, a CIDR value that looks something like `192.168.34.0/24` should be used for this field
* **`as_var`**: the name of the variable that you would like the interface name returned as

With these values in hand, the playbook will search the available networks on each machine and return a list of the interface names for each network that was described in the `iface_description_array` as the value of the fact named in the `as_var` field for that network's entry. For example, given this description:

```
    iface_description_array: [
        { as_var: 'data_iface', type: 'cidr', val: '192.168.34.0/24' },
        { as_var: 'api_iface', type: 'cidr', val: '192.168.44.0/24' },
    ]
```

In this example, the playbook will determine the name of the network that matches the CIDR blocks `192.168.34.0/24` and `192.168.44.0/24`, returning those interface names as the values of the `data_iface` and `api_iface` facts, respectively (eg. `eth0` and `eth1`). These two facts are then used later in the playbook to correctly configure the nodes to talk to each other (over the `data_iface` network) and listen on the proper interfaces for user requests (on the `api_iface` network).
