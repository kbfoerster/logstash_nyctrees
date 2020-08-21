# logstash_nyctrees
The purpose of this project is to send a modified version of data from the [nyctrees project](https://github.com/kbfoerster/nyctrees) into logstash. From logstash the data will be sent to an Elasticsearch. Afterwards, the data will be queried to validate  data was successfully imported. 

# Project Outline
* Docker will be used to create an Elasticsearch cluster, a kibana node, and a logstash node
	* The data will be read in from the logstash pipeline automatically
* Initial index settings will be configured for the data
* Queries will be performed on the data
* Visualizations will be created


## Docker-Compose File
The docker-compose file that is used is a modified version of the [default](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html), supplied by Elasticsearch. Specifically, we've added a kibana node and a logstash node, along with necessary configurations for each.
