# Managing Topics

The [modfy-topics.yml](../modify-topics.yml) playbook in this repository can be used to modify the topics within a deployed Kafka instance or cluster. Specifically, the playbook in this playbook can be used to:

* Add new topics to the cluster
* Remove existing topics from the cluster

In addition, the cluster (and associated Zookeeper ensemble) that should be targeted by a given [modfy-topics.yml](../modify-topics.yml) playbook run can be defined dynamically (by defining a set of matching tags that should be searched for in an AWS or OSP environment) ro statically (using a static inventory file to define the cluster and ensemble). We will show examples of each of these use cases in the sections that follow. The only assumption made is that the target nodes for the playbook run are actually members of a Kafka cluster.

## Adding Topics to the Cluster

As was mentioned previously, the [modfy-topics.yml](../modify-topics.yml) playbook provides users with the ability to specify the cluster that should be targeted by the playbook run either dynamically (using a set of tags that should be used to search for a matching set of Kafka cluster and Zookeeper ensemble nodes in an AWS or OSP based cloud environment) or statically (using a static inventory file to specify the targeted cluster and it's associated ensemble. Below, we will show examples of both of these use cases.

## Adding topics to a cluster via dynamic inventory

The easiest way to control the process of adding topics to a Kafka node or cluster is to create a custom configuration file to control the playbook run. If we create a static inventory file that looks something like this:


Then the custom configuration file that would be used to add two new topics to the cluster might look something like this:

```bash
$ cat config-dynamic-add-topics.yml
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
# actions to take and inputs for those actions
action_list:
  - action: create
    topics_list:
      - { name: test1, repl_factor: 2, partitions: 5 }
      - { name: test2, repl_factor: 2, partitions: 10 }
```

The first few parameters (the `cloud`, `tenant`, `project`, `domain`, `dataflow`, `cluster`, `internal_subnet`, `external_subnet`, `user`, and `region`) that are defined in this custom configuration file might look familiar to those of you who have used the [provision-kafka.yml](../provision-kafka.yml) playbook to provision Kafka to a set of target nodes and configure them as a Kafka cluster. However, the process of adding new topics to an existing cluster requires that we define a few of additional parameters. These additional parameters are contained in the `action_list` that is defined at the end of the file shown above. This `action_list` is used to define a list of *actions* that should be taken. In this example, we have a single `action` listed that will result in the creation of multiple topics (the `test1` and `test2` topics), and these two topics will be created in a single playbook run. As you can quite clearly see, the `create` action requires that a `topics_list` argument also be passed in as part of that action, and it is this `topics_list` argument (a list of dictionary entries) that contains the additional parameters that must be defined for this action; for each entry in this list we must specify the `name` of the topic as well as the the replication factor (`repl_factor`) and the number of `partitions` that should be used when creating that topic. With these parameters defined in our custom configuration file, adding these two topics to our cluster via an `ansible-playbook` run is as simple as the following command:

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
    config_file: 'config-dynamic-add-topics.yml' \
}" modify-topics.yml
```

During the playbook run, Ansible will connect to the target cloud environment (an AWS environment in this example) and determine which nodes should be targeted based on the tags that were passed into the playbook using the custom configuration file shown above (the `tenant`, `project`, `domain`, `dataflow`, and `cluster` tags). Once the target nodes have been identified, the playbook will then connect to those nodes and run the necessary commands to create the requested topics in that cluster. When that playbook run completes, the list of topics available in the cluster will include those that were defined in the custom configuration file shown above.

## Adding topics to a cluster via static inventory

As is the case with all of our playbooks, the target nodes could also be defined by passing a static inventory file containing a list of the nodes in the Kafka cluster (along with the associated Zookeeper ensemble and the parameters needed to connect to both) rather than passing in the tags that should be used to search for the target nodes. If we create a static inventory file that looks something like this:

```bash
$ cat combined-inventory
# example combined inventory file for clustered deployment

192.168.34.8 ansible_ssh_host=192.168.34.8 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.9 ansible_ssh_host=192.168.34.9 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.10 ansible_ssh_host=192.168.34.10 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.18 ansible_ssh_host=192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host=192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host=192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

[kafka]
192.168.34.8
192.168.34.9
192.168.34.10

[zookeeper]
192.168.34.18
192.168.34.19
192.168.34.20

```

Then the custom configuration file that would be used to add two new topics to the cluster might look something like this:

```bash
$ cat config-static-add-topics.yml
---
# network configuration used when configuring the connectors
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# actions to take and inputs for those actions
action_list:
  - action: create
    topics_list:
      - { name: test1, repl_factor: 2, partitions: 5 }
      - { name: test2, repl_factor: 2, partitions: 10 }
```

As you can see, the custom configuration file shown in this example is just a simplified version of the custom configuration file that was shown in the AWS based example that was shown previously. Since we have identified the target nodes using a static inventory file, the first few parameters from the dynamic inventory example that was shown above (the `cloud`, `region`, `tenant`, `project`, `domain`, `dataflow`, and `cluster` tags) are not needed here. The parameters that remain (the `internal_subnet`, `external_subnet`, `user`, and `action_list`) are the only parameters that we need to define when adding topics to a Kafka cluster that was defined using a static inventory file. With these parameters defined in our custom configuration file, adding these two topics to our cluster via an `ansible-playbook` run is as simple as the following command:

```bash
$ ansible-playbook -i combined_inventory -e "{ \
    config_file: 'config-static-add-topics.yml' \
}" modify-topics.yml
```

During the playbook run, Ansible will connect to the target nodes in the cluster (using the information defined in the static inventory file) and run the necessary commands to create the requested topics in the cluster. When the playbook run is complete, the list of topics available in the cluster will include those that were defined in the custom configuration file shown above.

## Removing Topics from the Cluster

The process of removing topics from the cluster is a bit more involved. To successfully remove topics from the cluster, we must:

* use the `kafka-topics` command (or the equivalent `kafka-topics.sh` shell script if the Apache Kafka distribution was used to build our cluster instead of the Confluent Kafka distribution) to remove the topics in question from the cluster
* for the rest of this process we must iterate over the nodes of our cluster to ensure that only one of the cluster nodes is offline at any given time; this includes:
    * shutting down the `kafka` service on the cluster node in question
    * if this is the first node in the cluster, connecting to the Zookeeper instance or ensemble that the Kafka instance or cluster is integrated with and removing the meta-data associated with these topics from that instance or ensemble
    * if the user requested a that we make a backup of the directories that were used to store the meta-data and data associated with these topics on the individual Kafka nodes before removing them from the node in question, then we should do so by taking a snapshot (in the form of a gzipped tarfile) of all of the matching directories (if any) that we find in the `kafka_data_dir` on the node in question for each of the topics that we are removing; this decision is based on the value of the `backup_topics` flag defined in the `action_list` (if a value for that flag is not specified in the `action_list` used for a particular playbook run, then no backup is made)
    * removing the directories that were used to store the meta-data and data associated with each of these topics (if any) in the `kafka_data_dir` on the node in question
    * restarting the `kafka` service on the node in question
    * pausing for a configurable amount of time (by default, the playbook pauses for 5 seconds between iterations) to ensure that the node is up before we move on to the next node in the cluster
* finally, if the cluster was built using the Apache Kafka distribution (rather than the Confluent Kafka distribution), then the same `kafka-topics.sh` command that was used in the first step of this process must be re-run to ensure that the topics are really removed; without this additional command, if the a topic with the same name as the topic we are removing is created at some later time it will come up as a topic that has been _marked for deletion_ in clusters that have been built using the Apache Kafka distribution.

As is the case with adding topics (see above), the target nodes for the playbook run can be identified dynamically (using a set of tags) or statically (using a static inventory file). In this example, we will only show the dynamic inventory use case for brevity. Performing the same operation using a static inventory file to control the playbook run is left as an exercise for the reader.

So, with the aforementioned description of the topic removal process in mind, removing the topics that we created in our previous examples (above), is a relatively simple matter. First, we need to create a custom configuration file that includes the paramters needed to remove those two topics from the cluster.


```bash
$ cat config-dynamic-delete-topics.yml
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
# actions to take and inputs for those actions
action_list:
  - action: delete
    topics_list:
      - { name: test1 }
      - { name: test2 }
    remote_zookeeper_dir: '/opt/zookeeper'
    backup_topics: true
```

As you can see (by comparing with our previous example), we've added definitions for the following pair of parameters to our `action_list`:

* **`backup_topics`**: a flag indicating whether or not the log directories associated with each of the topics being removed should be backed up locally on the nodes; if this parameter is unspecified a default value of `false` is used by default
* **`remote_zookeeper_dir`**: the value defined for this parmeter must be the same as the `zookeeper_dir` value that was used when the Zookeeper node or ensemble associated with the Kafka instance or cluster was created; its value points to the directory where the Zookeeper distribution was installed on the Zookeeper node or nodes and it is assumed that the `zkCli.sh` command that is used to remove the topic-related metadata from the Zookeeper node or ensemble can be found in the `{{remote_zookeeper_dir}}/bin` directory on the corresponding Zookeeper node or nodes

In addition, we've changed the `action` in our `action_list` (from `create` to `delete`). The fields that were used to set the replication factor and number of partitions for the listed topics are not necessary for this command so we have not bothered to define them for the `topic_list` shown in this example (if they were defined, they would be silently ignored). With this custom configuration file constructed, removing the two topics defined in that file is as simple as running the following `ansible-playbook` command:

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
    config_file: 'config-dynamic-delete-topics.ymll' \
}" modify-topics.yml
```

This command will remove the topics in question from the Kafka cluster, remove the meta-data associated with those topics from the associated Zookeeper node or nodes, backup the topic-related directories on each of the target nodes (since the `backup_topics` flag was set to `true` in the `delete` action shown above), and remove those topic-related directories from the target nodes. When the playbook run is complete, the topics listed in the `topic_list` of the `delete` action in the custom configuration file shown above will no longer appear in the list of topics being managed by the Kafka instance or cluster.

Finally, it should be noted that, as is the case with the [provision-kafka.yml](../provision-kafka.yml) playbook, the [modfy-topics.yml](../modify-topics.yml) playbook includes a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line at the beginning of the playbook file. As such, the playbook can be executed directly as a shell script (rather than using the file as the final input to an `ansible-playbook` command). This means that the command that was used above to add topics to an AWS cluster that was shown above could be rewritten as:

```bash
AWS_PROFILE=datanexus_west ./modify-topics.yml -e "{ \
    config_file: 'config-dynamic-add-topics.yml' \
}"
```

and the corresponding command to remove those topics from the cluster could be rewritten as:


```bash
AWS_PROFILE=datanexus_west ./modify-topics.yml -e "{ \
    config_file: 'config-dynamic-delete-topics.yml' \
}"
```

both forms (running the playbook as a shell script and using the playbook as the final input to an `ansible-playbook` command) accomplish the same thing; which form you choose to use is a matter of personal preference.
