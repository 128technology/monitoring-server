# 128T Monitoring Server #

This repository contains files and instructions that can be used to setup a basic monitoring dashboard for a 128T deployment.  The dashboard is built on top of Elastic Stack and is fed by data from the 128T Monitoring Agent.

## Topology ##

![Monitoring Framework](./monitoring_framework.png)

The above drawing illustrates the architecture of the Monitoring Server.  In this architecture, we will leverage Telegraf's kafka output to have the 128T Monitoring Agent send data to a kafka broker as a producer.  Logstash will connect to the Kafka broker as a consumer in order to pull data from the telegraf topic, perform some pre-processing of the data, and output the data into an Elasticsearch index.  Kibana will then be used to produce visualizations of the data in Elasticsearch.

Both Kafka and Elastic Stack are highly scalable solutions.  However, details on how to scale these products is beyond the scope of this documentation.  In this guide, we will provide instructions to setup a simple, simplex deployment for rapidly exploring the 128T Monitoring Data.  Additional information for scaling of Kafka or Elastic Stack for large deployments should be found in the respective solutions' documentation.

## Monitoring Server Setup ##

These instructions are based on a host system installed from a CentOS 1804 image.  Other Linux OS variants should be usable provided the setup instructions are modified appropriately.

Any software or external firewalls should be configured to allow the following traffic in to the Linux host:
* TCP/9092 for Kafka
* TCP/5601 for Kibana
* TCP/22 for SSH

The following sections should be run as root

### Install git and clone the repo ###
```
yum install git
git clone https://github.com/128technology/monitoring-server.git
```

### Install docker and docker compose ###
We will use docker to rapidly deploy the setup.  Please use the following commands to install docker:
```
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io
```

Once docker is installed, start and enable the docker service:
```
systemctl start docker
systemctl enable docker
```

Install docker-compose to be able to build the monitoring server from the provided docker-compose.yml file:
```
curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

Create symbolic link to make docker-compose command executable:
```
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

### Setup the environment advertised host IP for the Kafka container ###
Kafka must be informed of the address the producers will use to connect. Please define ADVERTISED_HOST in docker-compose.yaml file. by using editor such as vi or vim, in the kafka line, find the environment section. Modify KAFKA_ ADVERTISED_ HOST_ NAME: IP to your external IP address.
```
cd monitoring-server
KAFKA_ ADVERTISED_ HOST_ NAME: IP 
```
Where `X.X.X.X` is the address that the producers will use to connect to the kafka broker.  If this host sits behind an elastic IP or other NAT, be sure to use the external NAT address and not the host's local IP address.  Most scenarios aside from this should use the host's local IP address.

### Bring up the monitoring server ###
Use docker-compose to build the monitoring server by running the following command from root directory where the docker-compose file is present:
```
docker-compose up -d
```

Provide permissions on the local file system where Elasticsearch will store data:
```
chown -R 1000:1000 /var/lib/128t-monitoring-es
```


## Configure Monitoring Agent On Session Smart Network ##
This section contains a snippet of the monitoring agent configuration.  For full details on the monitoring agent configuration, please visit the [128T Documentation Site](https://docs.128technology.com/docs/plugin_monitoring_agent/).  

For deployments running 128T version 5.1.0 or greater on the conductor, install the monitoring agent plugin for configuration management.
Note : The instructions for installing and managing the plugin can be found [here](https://docs.128technology.com/docs/plugin_intro/#installation-and-management).

### Conductor Configuration ##
With the plugin installed, the configuration for the monitoring agent can be managed via the conductor. The general workflow for creating the configuration is as follows:

* Configure the inputs
* Configure the outputs
* Create an agent config profile
* Reference the profile on the router

#### Input Configuration ####
The monitoring agent plugin allows the user to configure a set of inputs to be captured to monitor the routers. The configuration can be found under `config > authority > monitoring > input`

```
input         arp
    name  arp
    type  arp
exit

input         events
    name   events
    type   events

    event
        admin         true
        audit         true
        alarm         true
        traffic       true
        provisioning  true
        system        true
        track-index   true
    exit
exit

input         device-interface
    name  device-interface
    type  device-interface
exit

input         peer-path
    name  peer-path
    type  peer-path
exit

input         metrics
    name     metrics
    type     metrics

    metrics
        use-default-config  true
    exit
exit

input         top-sessions
    name  top-sessions
    type  top-sessions
exit
```

#### Output Configuration ####
The output configuration provides information about the various data sink for the inputs.

```
output        kafka
    name               kafka
    type               kafka
    data-format        json

    kafka

        broker             X.X.X.X 9092
            host  X.X.X.X
            port  9092
        exit
        topic              telegraf
        compression-codec  none
    exit
    additional-config  (text/toml)
exit
```

Where `X.X.X.X` is the address of the Kafka broker as configured in the `ADVERTISED_HOST` environment variable above.
Add `version = "0.10.1.0"` in `additional-config` which is specific to the Kafka output type as follows:

```
admin@node1.conductor (output[name=kafka])# additional-config
Enter toml for additional-config (Press CTRL-D to finish):
version = "0.10.1.0"
```

#### Agent Configuration ####
The agent-config can be leveraged to create a monitoring profile by referencing the various inputs and outputs configured in the previous steps. This allows multiple profiles to be created and applies to different routers.

```
agent-config  my-agent
    name   my-agent

    tags   router
        tag     router
        router
    exit

    input  arp
        name  arp
    exit

    input  events
        name  events
    exit

    input  peer-path
        name  peer-path
    exit

    input  device-interface
        name  device-interface
    exit

    input  metrics
        name                 metrics
        include-all-outputs  true
    exit
exit
```

***
*NOTE*:  By default `include-all-outputs` is enabled for each of the inputs, meaning the input will be sent to all configured outputs. 
***

### Router Configuration ###

Once all the inputs, outputs and agent-config are provisioned, monitoring needs to be enabled and the profile should be referenced on all individual routers where the monitoring data needs to be collected. The monitoring config elements can be found under `authority > router > system > monitoring` 

```
config

    authority

        router  router1
            name    router1

            system

                monitoring
                    enabled       true
                    agent-config  my-agent
                exit
            exit
        exit
    exit
exit
```

***

For deployments running 128T version prior to 5.1.0, monitoring agent can be manually installed on each 128T node along with file based configuration as follows:

The agent can be install using the `dnf` utility.

```
dnf install 128T-monitoring-agent
```

## File based Configuration ##

Any 128T Monitoring Agent that connects to this monitoring server will need a configuration file for the kafka output.  Please put the following contents in `/var/lib/128t-monitoring/outputs/kafka.conf` on these system:
```
[[outputs.kafka]]
  ## URLs of kafka brokers
  brokers = ["X.X.X.X:9092"]
  ## Kafka topic for producer messages
  topic = "telegraf"
  max_retry = 3
  data_format = "json"
```
Where `X.X.X.X` is the address of the Kafka broker as configured in the ADVERTISED_HOST environment variable above.  This output should then be referenced in the `/etc/128t-monitoring/config.yaml` file.  A sample of this file is shown below (please change as appropriate for the metrics you wish to use):
```
enabled: true
tags:
- key: router
  value: ${ROUTER}
- key: node
  value: ${NODE}
sample-interval: 15
push-interval: 180
inputs:
- name: t128_events
- name: t128_metrics
- name: t128_top_analytics
- name: t128_device_state
- name: t128_peer_path
outputs:
- name: kafka
```
This monitoring server setup will understand data from the following inputs: t128_events, t128_top_analytics, t128_device_state, t128_peer_path, t128_lte_metric, and t128_metrics.  The dashboards expect only the default metric set for the t128_metrics.  To update the monitoring agent configuration for these changes and start/restart the services, run the following command:
```
monitoring-agent-cli configure
```

### Import Provided Dashboards ###
In a web browser, navigate to the monitoring server on port 5601.  Click on the gear icon in the menu in the left pane to Management settings.  Under the Kibana section of the menu, click the link for Saved Objects.  Click on the Import link in the upper right.  Navigate to the location of the 128t-monitoring-dashboards.ndjson file provided with this repo and click the Import button.  You should receive a message that the import was successful.  Click Done.

Often after importing the saved objects, Kibana does not immediately operate as expected when navigating through the menu.  It is suggested after import to close your browser tab and open a new session to Kibana.
