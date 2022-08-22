# infra

# Kubernetes cluster



# Prerequisite
	
1. Install oracle virtual box
		https://www.virtualbox.org/wiki/Downloads
		
2. Install CentOS-8
		http://isoredirect.centos.org/centos/8-stream/isos/x86_64/
		



|Resources    |Master        |Worker-1      |	Worker-2|							
|-----------|-------------|-----------------------------|---------|
|OS		  |CentOS-8     |CentOS-8     |    CentOS-8        | |
|Storage   |20 GB    |  20 GB          |    20 GB        |                ||
|vCPU		|		4	| 4    | 4
RAM   |8 GB        |4 GB|					4GB	 |								|

# steps for creating k8s cluster

## 1.  Disable swap
For ALL NODE
 
	swapoff -a
Once done, we also need to make sure swap is not re-assigned after node reboot so comment out the swap filesystem entry from `/etc/fstab`
## 2. Disable SELinux
For ALL NODE

	sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
## 3. Enable Firewall
For master node
	
6443 - API server
2379-2380 - etcd
10250 - kubelet api
10251 - kube-scheduler
10252- kube-controller-manager


		firewall-cmd --add-port 6443/tcp --add-port 2379-2380/tcp --add-port 10250-10252/tcp --permanent
		firewall-cmd --reload
		firewall-cmd --list-ports
	
For worker node
	
10250 - kubelet API
30000-32767 - NodePort
 
		firewall-cmd --add-port 10250/tcp --add-port 30000-32767/tcp --permanent
		firewall-cmd --reload
		firewall-cmd --list-ports
	
## 4. configure networking 
For ALL NODE

	Make sure that the `br_netfilter` module is loaded.
		lsmod | grep br_netfilter
	
	If `br_netfilter` not loaded then
		modprobe br_netfilter

	you should ensure `net.bridge.bridge-nf-call-iptables` is set to 1 in your `sysctl` config
		sysctl -a | grep net.bridge.bridge-nf-call-iptables

	Activate the newly added changes
		sysctl --system	
	
## 5. Install container runtime
For ALL NODE

		
Add the docker repository to be able to install docker on all the nodes :

			yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
	 
Install Docker CE package on all the nodes:

			yum install containerd.io docker-ce docker-ce-cli -y

Create a new directory on all the nodes:

			mkdir -p /etc/docker

Set up the Docker daemon on all the nodes by creating a new file `/etc/docker
/daemon.json` and adding following content in this file:

		cat /etc/docker/daemon.json
		{
		  "exec-opts": ["native.cgroupdriver=systemd"],
		  "log-driver": "json-file",
		  "log-opts": {
			  "max-size": "100m"
		  },
		  "storage-driver": "overlay2",
		  "storage-opts": [
		    "overlay2.override_kernel_check=true"
		  ]
		}

Restart docker daemon:
		
		systemctl daemon-reload
		systemctl enable docker --now
## 6. install Kubernetes component

	Create the kubernetes repository file on all the nodes which will be used to download the packages:
	
		cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
		[kubernetes]
		name=Kubernetes
		baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
		enabled=1
		gpgcheck=1
		repo_gpgcheck=1
		gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg 
		https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
		exclude=kubelet kubeadm kubectl
		EOF

Install the Kubernetes component packages using package manager on all the nodes:

	yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
## 7. Initialize master node

	kubeadm init --apiserver-advertise-address=<ip-address> --pod-network-cidr=10.244.0.0/16

If you are root user 

	export KUBECONFIG=/etc/kubernetes/admin.conf

If you are not root user

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
## 8. Install pod network plugin

	kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml


## 9. Join worker node 

To add new nodes to your cluster,as root user execute the "kubeadm join" command that you get it from kubeadm init command output

	kubeadm join <--token><--discovery-token-ca-cert-hash>



## Access cluster with kubectl

If you are in same network then
1. Copy admin.conf file from (/etc/kubernetes/admin.conf) master node 
2. Paste that file over  ~/.kube location (local host) 

If you are in different network then
	

 - Configure ssh tunnel 
	Run this command on local host
	
		ssh -fNL port:< API SERVER IP ADDRESS >:6443 username@server
 - check local machine listen on which port
			
			 netstat -nltp | grep <port>
 - Copy admin.conf file from (/etc/kubernetes/admin.conf) master node 
 - Paste that file over  ~/.kube location (local host) 
 - rename "admin.conf" file as "config"
 - configure DNS of local host

		sudo vi /etc/hosts
	
		127.0.0.1 kubernetes
	
 - change file in "server" section 
	 
		apiVersion: v1
		clusters:
		- cluster:
		    certificate-authority-data:
			    server: https://127.0.0.1:6443			<------ change ip with "kubernetes"
		  name: kubernetes
		contexts:
		-	context:
			    cluster: kubernetes
			    user: kubernetes-admin
			  name: kubernetes-admin@kubernetes
		current-context: kubernetes-admin@kubernetes
		kind: Config
		preferences: {}
		users:
		 - name: kubernetes-admin
		  user:
		    client-certificate-data:


## References

https://docs.docker.com/engine/install/centos/
https://kubernetes.io/docs/reference/setup-tools/kubeadm/


# Prometheus

# Installation

install the `prometheus`, `promauto`, and `promhttp` libraries

```
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promauto
go get github.com/prometheus/client_golang/prometheus/promhttp

```

# For custom application specific metrics (counter metrics)
 
 ## 1. Import libraries
```
github.com/prometheus/client_golang/prometheus
github.com/prometheus/client_golang/prometheus/promauto
github.com/prometheus/client_golang/prometheus/promhttp
```

## 2. Creating counter metric and registering counter metric to prometheus

## example - 
```
var  REQUEST_COUNT = promauto.NewCounter(prometheus.CounterOpts{
Name: "name of metric",
Help: "description of metric",
})
```
## 3. Add counter to  particular HTTP endpoint 

## example - 

````
REQUEST_COUNT.Inc()
````


## 4.  Expose /metrics HTTP endpoint

````
http.Handle("/metrics", promhttp.Handler())
````

## 5. Configure prometheus 

````
prometheus.yml

global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']			<------- add URL of your application

````

## References

https://prometheus.io/docs/instrumenting/clientlibs/


# Metrics visualization using grafana



# Prerequisite
	
1. Install telegraf
		https://docs.influxdata.com/telegraf/v1.21/introduction/installation/
		
2. Install influxdb
		https://docs.influxdata.com/influxdb/v1.7/introduction/installation/
3. install grafana
 https://grafana.com/docs/grafana/latest/installation/debian/
		
		





# Steps 

## 1.  Configure Telegraf Agent to Collect Metrics -
edit the configuration file:

``nano /etc/telegraf/telegraf.conf``

I. Add URL to scrape metrics from prometheus
II. specify metrics version (metrics_version = 2 recommended)

````# Read metrics from one or many prometheus clients

[[inputs.prometheus]]

## An array of urls to scrape metrics from.

urls = ["http://localhost:8080/monitoring/build/v1/metrics"]  <--- scrape metrics from this URL

  

## Metric version controls the mapping from Prometheus metrics into

## Telegraf metrics. When using the prometheus_client output, use the same

## value in both plugins to ensure metrics are round-tripped without

## modification.

##

## example: metric_version = 1; deprecated in 1.13

metric_version = 2  <-- metrics version 

# metric_version = 1

  

## An array of Kubernetes services to scrape metrics from.

# kubernetes_services = ["http://my-service-dns.my-namespace:9100/metrics"]

  

## Kubernetes config file to create client from.

# kube_config = "/path/to/kubernetes.config"

  

## Scrape Kubernetes pods for the following prometheus annotations:

## - prometheus.io/scrape: Enable scraping for this pod

## - prometheus.io/scheme: If the metrics endpoint is secured then you will need to

## set this to `https` & most likely set the tls config.

## - prometheus.io/path: If the metrics path is not /metrics, define it with this annotation.

## - prometheus.io/port: If port is not 9102 use this annotation

# monitor_kubernetes_pods = true

## Restricts Kubernetes monitoring to a single namespace

## ex: monitor_kubernetes_pods_namespace = "default"

# monitor_kubernetes_pods_namespace = ""

# label selector to target pods which have the label

# kubernetes_label_selector = "env=dev,app=nginx"

# field selector to target pods

# eg. To scrape pods on a specific node

# kubernetes_field_selector = "spec.nodeName=$HOSTNAME"

  

## Use bearer token for authorization. ('bearer_token' takes priority)

# bearer_token = "/path/to/bearer/token"

## OR

bearer_token_string = "abc"  <--- use bearer token for user authentication

  

## HTTP Basic Authentication username and password. ('bearer_token' and

## 'bearer_token_string' take priority)

# username = ""

# password = ""

  

## Specify timeout duration for slower prometheus clients (default is 3s)

# response_timeout = "3s"

  

## Optional TLS Config

# tls_ca = /path/to/cafile

# tls_cert = /path/to/certfile

# tls_key = /path/to/keyfile

## Use TLS but skip chain & host verification

# insecure_skip_verify = false 

````

## 2.Configuring Telegraf to send data :


``nano /etc/telegraf/telegraf.conf``

I. Configuration for sending metrics to InfluxDB
II. Add URL for influxdb instance
III. Add database name

````  #### # Configuration for sending metrics to InfluxDB

#### [[outputs.influxdb]]

#### ## The full HTTP or UDP URL for your InfluxDB instance.

#### ##

#### ## Multiple URLs can be specified for a single cluster, only ONE of the

#### ## urls will be written to each interval.

#### # urls = ["unix:///var/run/influxdb.sock"]

#### # urls = ["udp://127.0.0.1:8089"]

urls = ["http://127.0.0.1:8086"]  <----  URL for your InfluxDB instance

  

#### ## The target database for metrics; will be created as needed.

#### ## For UDP url endpoint database needs to be configured on server side.

database = "telegraf"  <----- databasename

  

#### ## The value of this tag will be used to determine the database. If this

#### ## tag is not set the 'database' option is used as the default.

#### # database_tag = ""

  

#### ## If true, the 'database_tag' will not be included in the written metric.

#### # exclude_database_tag = false

  

#### ## If true, no CREATE DATABASE queries will be sent. Set to true when using

#### ## Telegraf with a user without permissions to create databases or when the

#### ## database already exists.

#### # skip_database_creation = false

  

#### ## Name of existing retention policy to write to. Empty string writes to

#### ## the default retention policy. Only takes effect when using HTTP.

#### # retention_policy = ""

  

#### ## The value of this tag will be used to determine the retention policy. If this

#### ## tag is not set the 'retention_policy' option is used as the default.

#### # retention_policy_tag = ""

  

#### ## If true, the 'retention_policy_tag' will not be included in the written metric.

#### # exclude_retention_policy_tag = false

  

#### ## Write consistency (clusters only), can be: "any", "one", "quorum", "all".

#### ## Only takes effect when using HTTP.

#### # write_consistency = "any"

  

#### ## Timeout for HTTP messages.

#### # timeout = "5s"

  

#### ## HTTP Basic Auth

#### # username = "telegraf"

#### # password = "metricsmetricsmetricsmetrics"

  

#### ## HTTP User-Agent

#### # user_agent = "telegraf"

  

#### ## UDP payload size is the maximum packet size to send.

#### # udp_payload = "512B"

  

#### ## Optional TLS Config for use on HTTP connections.

#### # tls_ca = "/etc/telegraf/ca.pem"

#### # tls_cert = "/etc/telegraf/cert.pem"

#### # tls_key = "/etc/telegraf/key.pem"

#### ## Use TLS but skip chain & host verification

#### # insecure_skip_verify = false

  

#### ## HTTP Proxy override, if unset values the standard proxy environment

#### ## variables are consulted to determine which proxy, if any, should be used.

#### # http_proxy = "http://corporate.proxy:3128"

  

#### ## Additional HTTP headers

#### # http_headers = {"X-Special-Header" = "Special-Value"}

  

#### ## HTTP Content-Encoding for write request body, can be set to "gzip" to

#### ## compress body or "identity" to apply no encoding.

#### # content_encoding = "gzip"

  

#### ## When true, Telegraf will output unsigned integers as unsigned values,

#### ## i.e.: "42u". You will need a version of InfluxDB supporting unsigned

#### ## integer values. Enabling this option will result in field type errors if

#### ## existing data has been written.

#### # influx_uint_support = false

````

## 3. Setup Grafana Data Source :

You can access the Grafana Dashboard using the URL [http://localhost:3000](http://your-server-ip:3000)/

To add the data source, 

Configuration→Data source→Search for InfluxDB and click on the Select button-->Set the name of the InfluxDB-->select InfluxQL in Query Language-->define the URL of the InfluxDB data source-->provide InfluxDB database details-->and click on the Save & Test button.






