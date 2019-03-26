# Extended OpenShift Monitoring

This project aims to extend Prometheus Cluster Monitoring (PCM) and EFK-Stack that are proposed by Red Hat as monitoring toolset for OpenShift 3.11. Those proposals lack the ability for monitoring software-defined network (SDN) and application performance. Furthermore it's not possible to make PCM scraping external components like a storage system in an automate way. 

In order to address mentioned shortcomings two core components has been developed. One is called **OSE-Monitor-Client** it enables monitoring the SDN which is build upon Open vSwitch by default as well as gathering information about security context constraints. The other core component is called **Monitor-Operator**. As title suggests, it's build by using Operator SDK from CoreOS. This component provides the ability to integrate AWX/Ansible Tower as kind of a service discovery.

The following sections explain how to setup **Extended OpenShift Monitoring** in your own OpenShift 3.11 deployment.

###  Table
- Prerequisites
- Setup OSE-Monitor-Client
- Setup Monitor-Operator
- Optional: Setup Ceph Storage Monitoring
- Additional Notes
- TODO

## Prerequisites
 - Is is assumed you have an **OpenShift/OKD 3.11** deployment running (MiniShift is not supported)
 - As OpenShift 3.11 comes with PCM installed out of the box you only need to install **EFK-Stack** (Red Hat calls it "aggregated logging")
 - You also need **cluster-admin rights**
 - An **AWX** or **Ansible Tower** deployment must be reachable from OpenShift-Cluster

Optional:
 - This setup uses **Ceph** as storage system for OpenShift. So it will gives a proposal on how to monitor **RADOS Block Devices** in an integrated manner. Of course you can adapt this solution for other systems you want to integrate in your OpenShift-Monitoring but it might need some adjustments.

## Setup OSE-Monitor-Client

 - Create a self-signed certificate, let's call it **stunnel-cert** for later use
   - openssl genrsa -out /etc/stunnel/key.pem 4096
   - penssl req -new -x509 -key /etc/stunnel/key.pem -out /etc/stunnel/cert.pem -days 1826
   - cat /etc/stunnel/key.pem /etc/stunnel/cert.pem > /etc/stunnel/private.pem
 - Clone git repo: https://github.com/lz006/ose-monitor-client.git
 - cd ./ose-monitor-client
 - Create resources located in directory **/deploy/central** by using following order
   - namespace "openshift-logging"
     - cluster_role.yaml
     - service_account.yaml
     - cluster_role_binding.yaml
     - config_map_redis_master.yaml
     - config_map_stunnel_server.yaml
     - secret_redis.yaml
     - secret_stunnel.yaml
       - replace value "private.pem" by base64 encoded stunnel-cert
     - secret_awx.yaml
       - replace value "url" by base64 encoded awx url -> "http://<my_awx_ip_or_hostname>/api"
       - replace values "username" & "value" with your awx credentials
     - service_elasticsearch.yaml
     - service_stunnel_server.yaml
     - deployment_ose_monitor_client.yaml
  - Create resources located in directory **/deploy/agents** by using following order
   - namespace "openshift-logging"
     - scc_monitoring.yaml
     - config_map_fluentd.yaml
     - config_map_logstash.yaml
     - config_map_redis_slave.yaml
     - config_map_stunnel_client.yaml
     - daemonset.yaml

AWX Config:

 - Create a template that runs alertactions of your needs
 - Example playbook under https://github.com/lz006/alertactions.git just cleans dir /tmp/emptymy

Kibana Config:

 - Exchange Kibana certs for elasticsearch by elasitcsearch admin certs
 - Create new index pattern "netflow*" & "scc"

 

## Setup Monitor-Operator
 - Clone git repo: https://github.com/lz006/monitor-operator.git
 - cd ./monitor-operator
 - First grant privileges to serviceaccount "monitor-operator-agent" by using following command:
    - oc adm policy add-cluster-role-to-user system:auth-delegator -z monitor-operator-agent -n openshift-monitoring
 - Get password from router pod by using following command:
     - oc describe pod <your_router_pod_name> | grep STATS_
     - Lets call the result **router-pw** for later use
 - Create resources located in directory **/deploy/agents** by using following order
   - namespace "openshift-monitoring"
     - config_map_telegraf.yaml
     - config_map_telegraf_router.yaml
        - Replace password in url from section "inputs.prometheus" with router-pw
     - daemonset.yaml
     - daemonset_telegraf_router.yaml
 - Create resources located in directory **/deploy/central** by using following order
   - namespace "openshift-monitoring"
     - role.yaml
     - service_account.yaml
     - role_binding.yaml
     - secret_stunnel.yaml
	      - replace value "private.pem" by base64 encoded stunnel-cert
	 - secret.yaml
	      - replace value "url" by base64 encoded awx url -> "http://<my_awx_ip_or_hostname>/api"
	      - replace values "username" & "value" with your awx credentials
	 - config_map.yaml
	 - config_map_stunnel_client.yaml
	 - operator.yaml

AWX Config:

 - Make sure your inventory on awx corresponds to fields "awx_hosts_filter_query" & "awx_groups_filter_query" in configmap "monitor-operator-conf"
 - Set variables for a group on awx like example below:
 <pre>FIXME</pre>

Grafana Config:
- By default only a user that is called admin got required rights for modifying grafana dashboards. You can specify different username when by altering grafana config. It's not a big deal but you might take a look at grafana docs first.
- Configure Notification Channel:
   - Type: Webhook
   - TLS
     - Skip verify = true
     - Client auth = true
     - Set "stunnel-cert" from above 
- Create dashboards of your needs
- Configure alerts by using following yaml structure:
<pre>#Example1
action: delete
resource: storage-nodes
reproach: diskpressure
attributes:
    awx_template:
        id: 6</pre>
<pre>#Example2
action: delete
resource: pod
reproach: spy
attributes:
    time:
        range: 96h
    terms:
        geodst: "za;us"
        geosrc: "za;us"
    aggs:
        field: "dstPod.keyword;srcPod.keyword"
        sources: "dstNamespace;srcNamespace"</pre>
## Optional: Setup Ceph Storage Monitoring 
Following steps require a running Ceph cluster: 
 - Clone git repo: https://github.com/lz006/openshift-operations.git
 - cd ./openshift-operations
 - Adjust "/group_vars/all.yml" as well as "/group_vars/ceph_nodes.yml" to your needs
 - Male inventory "virtual_machines.ini" corresponding to your ceph nodes
- Exchange content "/roles/external-node-monitoring/fluentd/certs" with certs of elasticsearch master
- Point  "/roles/external-node-monitoring/fluentd/fluent.conf" to openshift service "logging-es" from namespace openshift-logging
- Make  "/roles/external-node-monitoring/kube_rbac_proxy/config" pointing openshift api server 
- Exchange certs in same dir by certs from kube-rbac-proxy instances that are running inside openshift
- Run playbook "playbook_addhoc.yml" from root dir

## Additional Notes
This projects contains some forks that can be found here:

 - https://github.com/lz006/ceph_exporter.git
 - https://github.com/lz006/extended-awx-client-go.git
 - https://github.com/lz006/app-simulator




## TODO

 - Make OpenShift resources least privileges
 - Configure proper Elasticsearch Index for Netflow data
 - Create ansible project to provide automated setup
