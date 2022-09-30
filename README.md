- [Install DataDog on Kubernetes Clusters](#datadog-install-on-eks-for-cfk)
  - [Create API key](#create-api-key)
  - [Install via helm](#install-via-helm)
  - [Site (none US folks)](#site-none-us-folks)
- [CR annotations for CFK](#cr-annotations-for-cfk)
  - [Example:](#example)
  - [Install the Confluent Platform integration](#install-the-confluent-platform-integration)
  - [Validation](#validation)
    - [Check the UI](#check-the-ui)
    - [Check the dashboard](#check-the-dashboard)
- [Review live logs of kafka-2](#review-live-logs-of-kafka-2)
- [Clean-up](#clean-up)
- [Troubleshooting](#troubleshooting)
  - [Inside the pod kafka-0](#inside-the-pod-kafka-0)
    - [Expected output](#expected-output)
  - [Outside the pod loop over all pods](#outside-the-pod-loop-over-all-pods)
      - [Expected output](#expected-output-1)
  - [Check JVM for agents and ports of Jolokia and prometheus](#check-jvm-for-agents-and-ports-of-jolokia-and-prometheus)
    - [Notice the followings](#notice-the-followings)
      - [Example](#example-1)
      
This  guide provides an introduction to Data Dog and CP integration when deployed on Kubernetes.  It walks through an example Agent installation and how you can use it to send system level metrics to the Datadog platform. 

# Install DataDog on K8s 

First we need to install a DD agent on every node of the K8s cluster. The Datadog Agent is software that runs on your hosts. It collects events and metrics from hosts and sends them to Datadog, where you can analyze your monitoring and performance data. It can run on your local hosts (Windows, MacOS), containerized environments (Docker, Kubernetes), and in on-premises data centers. You can install and configure it using configuration management tools (Chef, Puppet, Ansible).

For  information on how to install Datadog agent on Kubernetes Follow this(It walks through an example Agent installation) guide

## Create DataDog API key 


![UI](Screencaptures/dd-api-1)
![UI](Screencaptures/dd-api-2)

Navigate to the Organisational settings in your DataDog UI and scroll to the API keys section. Create a new key, Save it for future usage in CP integration on kubernetes nodes

[Create API key](https://app.datadoghq.eu/organization-settings/api-keys) for the next steps. 


## Install Datadog using  helm 

To install the chart for Datadog, identify the right release name:

1. Install Helm.
2. Using the Datadog values.yaml configuration file as a reference, create your values.yaml. Datadog recommends that your values.yaml only contain values that need to be overridden, as it allows a smooth experience when upgrading chart versions.
If this is a fresh install, add the Helm Datadog repo:
```
helm   repo add datadog https://helm.datadoghq.com
helm   repo update

```
3. Retrieve your Datadog API key from your Agent installation instructions and run:

[Steps](https://docs.datadoghq.com/containers/kubernetes/installation/?tab=helm)

```
helm repo add datadog https://helm.datadoghq.com
helm repo update
export ddkey=<DATADOG_API_KEY> 

helm install mbldd -f values.yaml  \
--set datadog.apiKey=$ddkey datadog/datadog \
--set targetSystem=linux \
--set datadog.logs.enabled=true \
--set datadog.logs.containerCollectAll=true \
--set agents.image.tagSuffix=jmx \
--set clusterChecksRunner.image.tagSuffix=jmx \
--set datadog.site=datadoghq.eu  # only for none us folks 
```



## Site (none US folks)  

Set your Datadog site to datadoghq.com using the DD_SITE environment variable in the datadog-agent.yaml manifest.
Note: If the `DD_SITE` environment variable is not explicitly set, it defaults to the US site `datadoghq.com`.  
If you are using one of the other sites (EU, US3, or US1-FED) this will result in an invalid API key message. Use the documentation site selector to see documentation appropriate for the site you’re using.

# Annotations for each Kafka Component  in CP deployment yaml file


This tutorial assumes you are aware of the process of deploying a CP cluster using CFK. If not, please get started using the quickstart CFK repository under Confluent for Kubernetes examples repo. 
Modify the [CFK quickstart-deploy](confluent-platform-DD.yaml)  

Add the followings to each CRD (used for events) so [Autodiscovery](https://docs.datadoghq.com/agent/guide/autodiscovery-with-jmx/?tab=containerizedagent#autodiscovery-annotations) will work, this example shows `kafka` after the `/`, this is the `name` of the CR.  

```
spec:
  podTemplate:
    annotations:
      ad.datadoghq.com/kafka.check_names: '["confluent_platform"]'
      ad.datadoghq.com/kafka.init_configs: '[{"is_jmx": true, "collect_default_metrics": true, "service_check_prefix": "confluent", "new_gc_metrics": true, "collect_default_jvm_metrics": true}]'
      ad.datadoghq.com/kafka.instances: '[{"host": "%%host%%","port":"7203","max_returned_metrics":3000}]'
      ad.datadoghq.com/kafka.logs: '[{"source":"confluent_platform","service":"confluent_platform"}]'
```

## Example: 

```
spec:
  podTemplate:
    annotations:
      ad.datadoghq.com/zookeeper.check_names: '["confluent_platform"]'
      ad.datadoghq.com/zookeeper.init_configs: '[{"is_jmx": true, "collect_default_metrics": true, "service_check_prefix": "confluent", "new_gc_metrics": true, "collect_default_jvm_metrics": true}]'
      ad.datadoghq.com/zookeeper.instances: '[{"host": "%%host%%","port":"7203","max_returned_metrics":3000}]'
      ad.datadoghq.com/zookeeper.logs: '[{"source":"confluent_platform","service":"confluent_platform"}]'
--
spec:
  podTemplate:
    annotations:
      ad.datadoghq.com/kafka.check_names: '["confluent_platform"]'
      ad.datadoghq.com/kafka.init_configs: '[{"is_jmx": true, "collect_default_metrics": true, "service_check_prefix": "confluent", "new_gc_metrics": true, "collect_default_jvm_metrics": true}]'
      ad.datadoghq.com/kafka.instances: '[{"host": "%%host%%","port":"7203","max_returned_metrics":3000}]'
      ad.datadoghq.com/kafka.logs: '[{"source":"confluent_platform","service":"confluent_platform"}]'
--
spec:
  podTemplate:
    annotations:
      ad.datadoghq.com/connect.check_names: '["confluent_platform"]'
      ad.datadoghq.com/connect.init_configs: '[{"is_jmx": true, "collect_default_metrics": true, "service_check_prefix": "confluent", "new_gc_metrics": true, "collect_default_jvm_metrics": true}]'
      ad.datadoghq.com/connect.instances: '[{"host": "%%host%%","port":"7203","max_returned_metrics":3000}]'
      ad.datadoghq.com/connect.logs: '[{"source":"confluent_platform","service":"confluent_platform"}]'
--
spec:
  podTemplate:
    annotations:
      ad.datadoghq.com/ksqldb.check_names: '["confluent_platform"]'
      ad.datadoghq.com/ksqldb.init_configs: '[{"is_jmx": true, "collect_default_metrics": true, "service_check_prefix": "confluent", "new_gc_metrics": true, "collect_default_jvm_metrics": true}]'
      ad.datadoghq.com/ksqldb.instances: '[{"host": "%%host%%","port":"7203","max_returned_metrics":3000}]'
      ad.datadoghq.com/ksqldb.logs: '[{"source":"confluent_platform","service":"confluent_platform"}]'
--
spec:
  podTemplate:
    annotations:
      ad.datadoghq.com/controlcenter.check_names: '["confluent_platform"]'
      ad.datadoghq.com/controlcenter.init_configs: '[{"is_jmx": true, "collect_default_metrics": true, "service_check_prefix": "confluent", "new_gc_metrics": true, "collect_default_jvm_metrics": true}]'
      ad.datadoghq.com/controlcenter.instances: '[{"host": "%%host%%","port":"7203","max_returned_metrics":3000}]'
      ad.datadoghq.com/controlcenter.logs: '[{"source":"confluent_platform","service":"confluent_platform"}]'
--
spec:
  podTemplate:
    annotations:
      ad.datadoghq.com/schemaregistry.check_names: '["confluent_platform"]'
      ad.datadoghq.com/schemaregistry.init_configs: '[{"is_jmx": true, "collect_default_metrics": true, "service_check_prefix": "confluent", "new_gc_metrics": true, "collect_default_jvm_metrics": true}]'
      ad.datadoghq.com/schemaregistry.instances: '[{"host": "%%host%%","port":"7203","max_returned_metrics":3000}]'
      ad.datadoghq.com/schemaregistry.logs: '[{"source":"confluent_platform","service":"confluent_platform"}]'
--
spec:
  podTemplate:
    annotations:
      ad.datadoghq.com/kafkarestproxy.check_names: '["confluent_platform"]'
      ad.datadoghq.com/kafkarestproxy.init_configs: '[{"is_jmx": true, "collect_default_metrics": true, "service_check_prefix": "confluent", "new_gc_metrics": true, "collect_default_jvm_metrics": true}]'
      ad.datadoghq.com/kafkarestproxy.instances: '[{"host": "%%host%%","port":"7203","max_returned_metrics":3000}]'
      ad.datadoghq.com/kafkarestproxy.logs: '[{"source":"confluent_platform","service":"confluent_platform"}]'

```

Refer to the completed CP yaml here [CP platform config](confluent-platform-with-DD.yaml) 

## Install the [Confluent Platform integration](https://app.datadoghq.eu/integrations?search=confluent)

Just click the UI in the DD dashboards 

![integ](images/integration.jpg)


## Validation
Run the Agent's status subcommand and look for confluent_platform under the JMXFetch section.
```
   ========
   JMXFetch
   ========

  Information
  ==================
    runtime_version : 11.0.16
    version : 0.46.0
   Initialized checks
  ==================
    confluent_platform
      instance_name : confluent_platform-10.92.6.5-7203
      message : <no value>
      metric_count : 115
      service_check_count : 0
      status : OK
   Failed checks
   =============
    no checks
```

```

#Execute into one of the DD agent pods 

kubectl get pods -l app.kubernetes.io/component=agent 
kubectl exec -it <any of the pods above> -- bash             
```

# Check the Datadog agent status
```
agent status 
```
### Check the UI 

![UI](images/MetricsSummaryDatadog.jpg)

### Check the dashboard 



# Review live logs of kafka-2
Change the DD URL as needed: 

https://app.datadoghq.eu/logs/livetail?query=pod_name%3Akafka-2


# Clean-up 

```
helm delete mbldd
```



# Troubleshooting 

## Inside the pod kafka-0  
```
kubectl --namespace=confluent exec -it kafka-0 -- bash
hostname
curl localhost:7778/metrics  |  grep ActiveControllerCount  | grep -v ^#   | grep .
exit
```

### Expected output  
```
bash-4.4$ hostname
kafka-0
bash-4.4$ curl localhost:7778/metrics  |  grep ActiveControllerCount  | grep -v ^#   | grep .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 4395k  100 4395k    0     0  7147k      0 --:--:-- --:--:-- --:--:-- 7136k
kafka_controller_kafkacontroller_value{name="ActiveControllerCount",} 0.0
bash-4.4$ exit
exit
```

## Outside the pod loop over all pods 
```
for i in {0..2}; do echo "kafka-$i" && \
kubectl --namespace confluent exec kafka-$i  -it -- bash -c "curl localhost:7778/metrics  |  grep ActiveControllerCount  | grep -v ^#   | grep ." ; done
```
#### Expected output 
```
% for i in {0..2}; do echo "kafka-$i" && \
kubectl --namespace confluent exec kafka-$i  -it -- bash -c "curl localhost:7778/metrics  |  grep ActiveControllerCount  | grep -v ^#   | grep ." ; done

kafka-0
Defaulted container "kafka" out of: kafka, config-init-container (init)
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 4395k  100 4395k    0     0  7906k      0 --:--:-- --:--:-- --:--:-- 7906k
kafka_controller_kafkacontroller_value{name="ActiveControllerCount",} 0.0
kafka-1
Defaulted container "kafka" out of: kafka, config-init-container (init)
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 4393k  100 4393k    0     0  6348k      0 --:--:-- --:--:-- --:--:-- 6339k
kafka_controller_kafkacontroller_value{name="ActiveControllerCount",} 0.0
kafka-2
Defaulted container "kafka" out of: kafka, config-init-container (init)
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 4651k  100 4651k    0     0  6722k      0 --:--:-- --:--:-- --:--:-- 6712k
kafka_controller_kafkacontroller_value{name="ActiveControllerCount",} 1.0
```

## Check JVM for agents and ports of Jolokia and prometheus

```
kubectl --namespace=confluent exec -it kafka-0 -- bash
hostname
ps -ef | grep kafka
```
### Notice the followings 

```
 -Dcom.sun.management.jmxremote.port=7203
jolokia-jvm-1.7.1.jar=port=7777
-javaagent:/usr/share/java/cp-base-new/jmx_prometheus_javaagent-0.14.0.jar=7778:/mnt/config/shared/jmx-exporter.yaml
```

#### Example 
```
 % kubectl --namespace=confluent exec -it kafka-0 -- bash
bash-4.4$ hostname
kafka-0
bash-4.4$ ps -ef | grep kafka
1001         1     0  6 13:19 ?        00:10:16 java -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dkafka.logs.dir=/var/log/kafka -Dlog4j.configuration=file:/opt/confluentinc/etc/kafka/log4j.properties -cp /usr/bin/../ce-broker-plugins/build/libs/*:/usr/bin/../ce-broker-plugins/build/dependant-libs/*:/usr/bin/../ce-auth-providers/build/libs/*:/usr/bin/../ce-auth-providers/build/dependant-libs/*:/usr/bin/../ce-rest-server/build/libs/*:/usr/bin/../ce-rest-server/build/dependant-libs/*:/usr/bin/../ce-audit/build/libs/*:/usr/bin/../ce-audit/build/dependant-libs/*:/usr/bin/../share/java/kafka/*:/usr/bin/../share/java/confluent-metadata-service/*:/usr/bin/../share/java/rest-utils/*:/usr/bin/../share/java/confluent-common/*:/usr/bin/../share/java/ce-kafka-http-server/*:/usr/bin/../share/java/ce-kafka-rest-servlet/*:/usr/bin/../share/java/ce-kafka-rest-extensions/*:/usr/bin/../share/java/kafka-rest-lib/*:/usr/bin/../share/java/confluent-security/kafka-rest/*:/usr/bin/../share/java/confluent-security/schema-validator/*:/usr/bin/../support-metrics-client/build/dependant-libs-2.13.6/*:/usr/bin/../support-metrics-client/build/libs/*:/usr/bin/../share/java/confluent-telemetry/*:/usr/share/java/support-metrics-client/* -Djava.rmi.server.hostname=kafka-0.kafka.confluent.svc.cluster.local -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.port=7203 -Dcom.sun.management.jmxremote.rmi.port=7203 -Dcom.sun.management.jmxremote.ssl=false -Djava.awt.headless=true -Djdk.tls.ephemeralDHKeySize=2048 -Djdk.tls.server.enableSessionTicketExtension=false -XX:+ExplicitGCInvokesConcurrent -XX:+PrintFlagsFinal -XX:+UnlockDiagnosticVMOptions -XX:+UseG1GC -XX:ConcGCThreads=1 -XX:G1HeapRegionSize=16 -XX:InitiatingHeapOccupancyPercent=35 -XX:MaxGCPauseMillis=20 -XX:MaxMetaspaceFreeRatio=80 -XX:MetaspaceSize=96m -XX:MinMetaspaceFreeRatio=50 -XX:ParallelGCThreads=1 -server -javaagent:/usr/share/java/cp-base-new/disk-usage-agent-7.2.1.jar=/opt/confluentinc/etc/kafka/disk-usage-agent.properties -javaagent:/usr/share/java/cp-base-new/jolokia-jvm-1.7.1.jar=port=7777,host=0.0.0.0 -javaagent:/usr/share/java/cp-base-new/jmx_prometheus_javaagent-0.14.0.jar=7778:/mnt/config/shared/jmx-exporter.yaml kafka.Kafka /opt/confluentinc/etc/kafka/kafka.properties
1001       510   502  0 15:54 pts/0    00:00:00 grep kafka
```

