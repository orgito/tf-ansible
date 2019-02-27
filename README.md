# Ansible Playbooks

## Requirements

To work with the playbooks you need to install the following requirements:
- Ansible 2.7.7
- boto 2.49.0
- boto3 1.9.90

### Installing the requirements (virtualenv)

The recommended way is to use a virtual environment. From the root of the repo run the following commands:

```bash
python -m virtualenv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Activating the virtualenv

Before using these playbooks you need to activate the virtualenv for your current session. Change the root of the repo and run that command;

```bash
source venv/bin/activate
```

### Installing the requirements (globally)

If you feel that that using the virtuaenv is incovenient you can install them globally. Beware that this might interfere with other packages in your system.

```bash
sudo pip install -r requirements.txt
```

This way it will not be necessary to activate the virtuaenv before using the playbooks.

## The dynamic environment

Instead of using a static file as the ansible environment, we will use the `ec2.py` script as our dynamic environent. This require some configuration.

### Filtering the instances

Inside the inventory folder, copy the file `ec2.ini.sample` as `ec2.ini` and adjust the following variables:

- **`regions`**: The AWS region where the instances are hosted. eg: `us-east-1`, `eu-west-1`, etc
- **`instance_filters`**: If we don't filter the intances you inventory will contain all the instances of you AWS account. This is unnecessary and create peformance impact. Use the value `tag:Provision=terraform&tag:Environment=<stage>`, replacing '<stage>' with the value that you used in the Terraform config. That is the Environment that you want to manage. eg: `prod`, `dev`, etc.

Either clone the repo in separate a separte forlder for each environment and adjust `instance_filters` or remember to always change the environment according to your needs.

### Credentials

The dynamic filter needs your credentials to query AWS. You can pass the credentias setting the appropriate environment variables:

```bash
export AWS_ACCESS_KEY_ID='AKXXXXXXXXXXXXX'
export AWS_SECRET_ACCESS_KEY='XXXXXXXXXXXXXXXXXXX'
```

Or you can set the variables `aws_access_key_id` and `aws_secret_access_key` in the `ec2.ini` file.

## The Playbooks

Before running any of the playbooks in production test them in a test environment. Remember to adjust the `instance_filters` variable to point to the correct environment.

### ELK Playbooks

A set of playbooks to manage your Elastic stack.

#### ELK Upgrade

Use the playbook `elk/upgrade/main.yml` to perform an upgrade of you elk stack. Use the variable `elk_version` to pass a valid target version for your environment.

```bash
ansible-playbook elk/upgrade/main.yml -e elk_version=6.6.0
```

#### ELK Settings

There is a pair of playbooks that should be used for making changes to the configuration of the Elastic stack components. The first one retrieves the current configuration files for Elasticsearc, Logstash and Kibana and stores them in the `conf` folder. Then you can change those files and apply the configuration with the second playbook.

To retrieve the current configuration run the command:

```bash
ansible-playbook elk/settings/retrieve.yml
```

Make your changes to the files `elasticsearch.yml`, `logstash.conf` and/or `kibana.yml` under the `conf` folder and apply the new configuration with the command:

```bash
ansible-playbook elk/settings/apply.yml
```

The `apply.yml` playbook will copy configuration and restart the services. For Logstash the configuration will be validated before applying. Elasticsearch and Kibana do not have an option to validate the configuraiton, so the change will be applied regardless of validation.

### Neo4j Playbooks

A set of playbooks to manage your Neo4j Casual Cluster.

#### Neo4j Upgrade

Before upgrading Neo4j make sure you have an updated backup. As always perform the upgrade on a test environment as similar to the production environment as possible. That way you can make sure that the upgrade is safe and will have a good estimate of the time it will take.

The playbook will perform a rolling upgrade. This means that the Neo4j cluster will remain operational during the upgrade. Still try to perform the upgrade when the cluster is not in use or at least not under heavy load.

Always check the [release notes](https://neo4j.com/release-notes/) to make sure the upgrade path is supported for a rolling upgrade and that there is no breacking changes.

The playbook need to check the status of the node before upgrading. Therefore it will need admin credentials to the cluster. You can pass the credentials via the variables `neo4j_user` and `neo4j_password`.

To upgrade the Neo4j Casual Cluster run the following command:

```bash
ansible-playbook neo4j/upgrade/main.yml -e neo4j_user=neo4j -e neo4j_password=senha -e neo4j_version=3.5.3
```

The playbook was tested for upgrading from 3.5.2 to 3.5.3.

#### Neo4j Settings

To retrieve the current configuration run the command:

```bash
ansible-playbook neo4j/settings/retrieve.yml
```

This will create 2 files under the `conf` folder: `neo4j_core.conf` for the core nodes and `neo4j_replica.conf` for the replica nodes.

Those 2 files are actually templates. Pay attention to the line:

```
dbms.connectors.default_advertised_address={{ ansible_default_ipv4.address }}
```

**Do not change that line.** Ansible will replace it with the right configuration.

Make your changes to one or both files and apply the configuration with the command:

```bash
ansible-playbook neo4j/settings/apply.yml
```

The configuration will be updated and the services restarted.

### Redis Playbooks

A set of playbooks to manage your Redis Cluster.

#### Redis Upgrade

To perform a rolling upgrade of the Redis cluster run the command below passing the appropriate `redis_version`:

```bash
ansible-playbook redis/upgrade/main.yml -e redis_version=5.0.3
```

#### Redis Settings

To retrieve the current configuration run the command:

```bash
ansible-playbook redis/settings/retrieve.yml
```

This will dowload the file `/etc/redis.conf` from your Redis clouster to the `conf` folder.

After editing the appropriate settings apply the new configuration with the command:

```bash
ansible-playbook redis/settings/apply.yml
```

The configuration will be changed and the services restarted.

### RabbitMQ Playbooks

A set of playbooks to manage your RabbitMQ Cluster.

#### RabbitMQ Upgrade

Depending on what versions are involved in an upgrade, you should perform an rolling upgrade or a full stop upgrade.

If rolling upgrades are not possible, the entire cluster should be stopped, then restarted.

Rolling upgrades from one patch version to another (i.e. from 3.7.x to 3.7.y) are supported except when indicated otherwise in the release notes. It is strongly recommended to consult release notes before upgrading.

If the release notes indicates that a rolling upgrade is possible, use the following command to performm the upgrade:

```bash
 ansible-playbook rabbitmq/upgrade/rolling.yml -e rabbitmq_version=3.7.12
```

If a rolling upgrade is not possible perform a full stop upgrade:

```bash
 ansible-playbook rabbitmq/upgrade/full_stop.yml -e rabbitmq_version=3.7.12
```

#### RabbitMQ Settings

To retrieve the current configuration run the command:

```bash
ansible-playbook rabbitmq/settings/retrieve.yml
```

This will dowload the file `/etc/rabbitmq/redis.conf` from your Redis clouster to the `conf` folder.

After editing the appropriate settings apply the new configuration with the command:

```bash
ansible-playbook rabbitmq/settings/apply.yml
```

The configuration will be changed and the services restarted.

### Ad Hoc Commands

You can use ansible to quickly perform ad hoc commands against you servers. Here are a few simple examples:

```bash
# Confirm that your control node can communicate with all ec2 instances
ansible all -m ping

# Run a simple command agains a subset of your infra
ansible tag_Role_redis_master -a 'uptime'

# You can use multiple targets
ansible tag_Role_redis_master:tag_Role_redis_slave -a 'uptime'

# Or globbing
ansible tag_Role_neo4j_* -a 'uptime'

# If the command requires root privilege use -b option
ansible tag_Role_kibana -b -a 'fdisk -l'

# If you need to use pipes, redirects, globbing, etc use the shell module
ansible tag_Role_elasticsearch -b -m shell -a 'ps fax | grep java'
```

You are not limited to running comands and can execute any supported Ansible module that way.

### Testing

Always test the playbooks in a test environment first. Remember to wait a few minutes after Terraform finish deployment the enviroment as the instances will still be executing the provisioning scripts after Terraform finishes.
