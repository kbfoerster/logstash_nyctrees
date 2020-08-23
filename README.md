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

Before using the docker-compose file, it's necessary to change the virtual memory settings of the host OS. The command `sysctl -w vm.max_map_count=262144` will prevent the containers from failing with an `Exit Code 78` error. This is due to the default memory being too low for the Elasticsearch node settings (see [this thread](https://github.com/laradock/laradock/issues/1699) for more details). If you're doing this on Windows, this command needs to be run in the default WSL terminal. The container can then be successfully started. 

## Import Verification and Index Configuration via Kibana

Once our containers are brought up, we can navigate to the index management page in Kibana to see our newly created index and verify it has the expected number of entries. ![We can see this assumption is true](/images/initial_index.PNG) 

Since this is a one-time import of static data, we'll create a new index pattern without a specified timestamp. After doing so, we can get a closer look at the logs from the 'Discover' tab in Kibana. This will give us a good idea for how our data is structured. ![](/images/discover_exploration.PNG). 

Let's assume that we are asked to run these initial queries and create a visualization, after which no new data will be added. We will create a polciy for this index that will allow the data to stay in the "warm" phase for 30 days before migrating to the "cold" phase where it will be frozen. This will give us plenty of time to write our queries and build our dashboard before moving on. You can see how the ILM policy was configured [here](/kibana/ilm_policy.txt). This will move the data to a single container in our cluster, es03, that we've identified as "rack_two". 

## Data Queries
