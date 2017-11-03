# kafka
Playbooks/Roles containing operations-related actions needed to manage and maintain a Kafka cluster; supports management of both [Confluent](https://www.confluent.io/) and [Apache](https://kafka.apache.org/) Kafka clusters.

# Installation
To manage a Kafka cluster using the [modify-topics.yml](modify-topics.yml) and [modify-connectors.yml](modify-connectors.yml) playbooks in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone https://github.com/Datanexus/kafka-operations
```

That command will pull down the repository, which includes the [modify-topics.yml](modify-topics.yml) and [modify-connectors.yml](modify-connectors.yml) playbooks and all of their dependencies.

The only other requirements for using the playbook in this repository are a relatively recent (v2.4 or later) release of Ansible. The easiest way to obtain a recent relese if Ansible is via a `pip install`, which requires that Python and pip are both installed locally. We have performed all of our testing using a recent (2.7.x) version of Python (Python 2); your mileage may vary if you attempt to run the playbook or the attached dynamic inventory scripts under a newer (v3.x) release of Python (Python 3).

# Using this role to manage Kafka clusters
The [modify-topics.yml](modify-topics.yml) and [modify-connectors.yml](modify-connectors.yml) playbooks at the top-level of this repository support management of both single-node (standalone) Kafka deployments and multi-node Kafka clusters. The process of managing these nodes will vary, depending on whether you are managing your inventory dynamically or statically, so examples are provided for both of these use cases in the associated documentation on adding and removing topics from a cluster and managing connectors within a cluster.

Out of the box, the [modify-topics.yml](modify-topics.yml) and [modify-connectors.yml](modify-connectors.yml) playbooks will take no action, so a command like this:

```bash
$ AWS_PROFILE=datanexus_west ./modify-topics.yml
```

will use the tags from the default configuration file (the [config.yml](config.yml) file) to search for a set of target nodes, but no action will be taken once those target nodes have been identified. It is left to the user to define the list of actions that should be taken by the playbook run using a custom configuration file. Examples of how to define the actions that should be taken are shown in the [Managing-Topics.md](docs/Managing-Topics.md) and [Managing-Topics.md](docs/Managing-Connectors.md) files.

It should also be noted here that the `AWS_PROFILE` environment variable value shown above (and in the examples shown in the aforementioned files) will likely need to change to reflect the profiles configured in your local AWS credentials file.

## Controlling the configuration
As was mentioned briefly in the prevision section, this repository includes default values for a number of parameters in the [vars/kafka-operations.yml](vars/kafka-operations.yml) file and the default configuration file (the [config.yml](config.yml) file) that make it possible to perform operations on a multi-node Kafka cluster, out of the box, with few (if any) changes necessary. If you are not happy with the default configuration defined in these two files, there are a number of ways that you can customize the configuration used for your deployment, and which method you use is entirely up to you:

* You can edit the [vars/kafka-operations.yml](vars/kafka-operations.yml) file and/or [config.yml](config.yml) file to modify the default values that are defined in those files or define additional configuration parameters
* You can pass in values as extra variables on the command-line of the Ansible playbook run to override those defined in the [vars/kafka-operations.yml](vars/kafka-operations.yml) file; values defined in the default configuration file cannot be overridden in this manner since that file is loaded last by the plays in the playbooks in this repository (so values defined in a custom configuration file override any values defined elsewhere)
* You can setup a custom *configuration file* on the local filesystem of the Ansible host that contains the values for the parameters you wish to set or customize, then pass the location of that file into your playbook command as an extra variable

We have provided a summary of the configuration parameters that can be set (using any of these three methods) during a playbook run [here](docs/Supported-Config-Params.md). Overall, we have found the last option to be the easiest and most flexible of those three options because:

* It avoids modifying files that are being tracked under version control in the main, [kafka-operations](https://github.com/Datanexus/kafka-operations) repository (the first option); making such changes will, more than likely, lead to conflicts at a later date when these files are modified in the main [kafka-operations](https://github.com/Datanexus/kafka-operations) repository in a way that is inconsistent with the values that you have set locally (in your clone of this repository).
* It lets you maintain your preferred configuration for any given Kafka deployment in the form of a configuration file, which you can easily maintain (along with the configuration files used for other deployments you have made) under version control in a separate repository
* It provides a record of the configuration of any given deployment, which is in direct contrast to the second option (where the configuration parameters for any given deployment are passed in on the command-line as extra variables)

That being said, the second option may be useful for some deployment scenarios (a one-off operation performed in a local test environment, for example), so it remains a viable option for some users. Overall, we would recommend against trying to maintain your preferred cluster configuration using the values defined in the [vars/kafka-operations.yml](vars/kafka-operations.yml) file or the default configuration file (the [config.yml](config.yml) file).

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully). Furthermore, it is assumed that you are working with a cluster that was built based on a relatively recent version of Kafka using this playbook (the current default for Confluent is v3.3 but any v3.x release should do, while the current release for Apache is the v0.10.1.0 release that was built using Scala v2.11).
