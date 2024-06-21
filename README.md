# Monitoring Wso2 Carbon logs using Grafana, Elasticsearch, Fluentd

 
* Grafana - Grafana is an open-source software. Grafana is best known as a visualization / dashboarding tool focused on graphing metrics from various data sources, such as Elasticsearch, InfluxDB, prometheus.
 
* Elasticsearch - Elasticsearch is a distributed, open source search and analytics engine for all types of data, including textual, numerical, geospatial, structured, and unstructured. Elasticsearch is built on Apache Lucene.

* Fluentd - Fluentd is an open source data collector, which lets you unify the data collection and consumption for a better use and understanding of data.
 
 ## I’m using Red Hat Enterprise Linux Server version 7.5. To install these products follow as below.:rocket:
 
## Install Grafana

* Follow the below steps to install Grafana in a single instance.
1. Add new Grafana to the YUM repository by executing following command.
```
$ cd /etc/yum.repos.d/
$ nano grafana.rep
```
2. Add the following configurations to the above created “grafana.rep“ repository and save to complete the adding Grafana repository.
```[grafana]
name=grafana
baseurl=https://packagecloud.io/grafana/stable/el/7/$basearch
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packagecloud.io/gpg.key https://grafanarel.s3.amazonaws.com/RPM-GPG-KEY-grafana
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```
3. Once it is done, install Grafana using the following yum command.
```
$ yum -y install grafana
```
4. After completion the above reload the systemd manager configuration and start the Grafana service.
```
$ systemctl daemon-reload
$ systemctl start grafana-server
$ systemctl enable grafana-server
```
## Install Elasticsearch

* PREREQUISITES

Following software should be installed in the instance.
~Java 8

* INSTALLATION GUIDE
1. Create the repository file
```
$ sudo curl https://d3g5vo6xdbdb9a.cloudfront.net/yum/opendistroforelasticsearch-artifacts.repo -o /etc/yum.repos.d/opendistroforelasticsearch-artifacts.repo
```
2. List all available Open Distro for Elasticsearch versions:
```
$ sudo yum list opendistroforelasticsearch — showduplicates
```
3. Choose the version opendistroforelasticsearch-1.3.0 and install
```
$ sudo yum install opendistroforelasticsearch-1.3.0
```
4. Disable security plugin for http configuration -/etc/elasticsearch/elasticsearch.yml
```
$ opendistro_security.disabled: true
```
5. Start Open Distro for Elasticsearch
```
$ sudo systemctl start elasticsearch.service
```
6. Send requests to server to verify that Elasticsearch is up and running:
```
$ curl -XGET https://localhost:9200 -u admin:admin — insecure curl -XGET https://localhost:9200/_cat/nodes?v -u admin:admin — insecure curl -XGET https://localhost:9200/_cat/plugins?v -u admin:admin — insecure
```
* If you encounter following errors you can safely ignore if the elasticsearch service is running, according to the official documentation
```
elasticsearch[3969]: java.security.policy: error adding Entry:
elasticsearch[3969]: java.net.MalformedURLException: unknown protocol: jrt
elasticsearch[3969]: java.security.policy: error adding Entry:
elasticsearch[3969]: java.net.MalformedURLException: unknown protocol: jrt
```
7. Start Open Distro for Elasticsearch
```
$ sudo /bin/systemctl daemon-reload
$ sudo /bin/systemctl enable elasticsearch.service
```
* Where are the files?

### The RPM package installs files to the following locations:


| File type  | Location |
| ------------- | ------------- |
| Elasticsearch home, management scripts, and plugins  | /usr/share/elasticsearch/  |
| Configuration files  | /etc/elasticsearch  |
| Environment variables  | 	/etc/sysconfig/elasticsearch  |
| Logs  | /var/log/elasticsearch  |
| Shard data  | /var/lib/elasticsearch  |


* REFERENCES : [Elastic search open distro installation guide](https://opendistro.github.io/for-elasticsearch-docs/docs/install/rpm/)

## Install Fluentd

* Before Installation

Please follow the Preinstallation Guide to configure your OS properly. This will prevent many unnecessary problems.
1. This shell script registers a new rpm repository at /etc/yum.repos.d/td.repo and installs the td-agent rpm package.
```
$ curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent3.sh | sh
```
2. Start td-agent.service and check whether fluentd is up.
```
$ sudo systemctl start td-agent.service
$ sudo systemctl status td-agent.service

● td-agent.service — td-agent: Fluentd based data collector for Treasure Data
Loaded: loaded (/lib/systemd/system/td-agent.service; disabled; vendor preset: enabled)
Active: active (running) since Thu 2017–12–07 15:12:27 PST; 6min ago
Docs: https://docs.treasuredata.com/articles/td-agent
Process: 53192 ExecStart = /opt/td-agent/embedded/bin/fluentd — log /var/log/td-agent/td-agent.log — daemon /var/run/td-agent/td-agent.pid (code = exited, statu
Main PID: 53198 (fluentd)
CGroup: /system.slice/td-agent.service
├─53198 /opt/td-agent/embedded/bin/ruby /opt/td-agent/embedded/bin/fluentd — log /var/log/td-agent/td-agent.log — daemon /var/run/td-agent/td-agent
└─53203 /opt/td-agent/embedded/bin/ruby -Eascii-8bit:ascii-8bit /opt/td-agent/embedded/bin/fluentd — log /var/log/td-agent/td-agent.log — daemon /v
Dec 07 15:12:27 ubuntu systemd[1]: Starting td-agent: Fluentd based data collector for Treasure Data…
Dec 07 15:12:27 ubuntu systemd[1]: Started td-agent: Fluentd based data collector for Treasure Data.
```
3. To feed carbon logs from EI or APIM to Open Distro do the following modification. (Apply below configurations to fluentd agents in all EI nodes).
3.1 Add following workers to /etc/td-agent/td-agent.conf
Change worker count at the top of the config.
```
<system>
workers 1 #worker count starts from ‘0’
</system>
```
* Worker for carbon logs
```
##worker 1
<worker 1>
## Source descriptions:
## built-in tail input
## wso2CArbon logs
<source>
  @type tail
  path <PRODUCT_HOME>/repository/logs/wso2carbon.log
  pos_file /var/log/td-agent/wso2.carbonlog.pos #
  tag wso2.service.carbonlogmonitor
    <parse>
       @type regexp
       expression / #use your own regex for filter the logs /
       time_format %Y-%m-%d %H:%M:%S
       keep_time_key true
    </parse>
</source>
## Output descriptions:
## wso2CArbon logs
## Match tag= wso2.service.carbonlogmonitor. Produce to elasticsearch
<match wso2.service.carbonlogmonitor>
  @type copy
   <store>
     @type elasticsearch
     @id output_carbonlog
     host localhost
     port 9200
     include_tag_key true
     tag_key @log_name4
     logstash_format true
     logstash_prefix custom_index_name #optional(put the name you want)
     flush_interval 5s
  </store>
</match>
</worker>
```
4. Restart fluentd
```
$ sudo systemctl restart td-agent.service
```
5. Tail your carbon logs using command line.
```
$ tail -f <PRODUCT_HOME>/repository/logs/wso2carbon.log
```


### Create grafana datasource

To visualize logs, grafana needs to create a datasource to collect logs that are stored in elasticsearch index.

What is an Elasticsearch index?

An Elasticsearch index is a collection of documents that are related to each other. Elasticsearch stores data as JSON documents. Each document correlates a set of keys (names of fields or properties) with their corresponding values (strings, numbers, Booleans, dates, arrays of values, geolocations, or other types of data).
Elasticsearch uses a data structure called an inverted index, which is designed to allow very fast full-text searches. An inverted index lists every unique word that appears in any document and identifies all of the documents each word occurs in.
During the indexing process, Elasticsearch stores documents and builds an inverted index to make the document data searchable in near real-time. Indexing is initiated with the index API, through which you can add or update a JSON document in a specific index.



	
	

	
	
