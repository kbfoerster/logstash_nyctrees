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

Let's assume that we are asked to run these initial queries and create a visualization, after which no new data will be added. We will create a polciy for this index that will allow the data to stay in the "warm" phase for 30 days before migrating to the "cold" phase where it will be frozen. This will give us plenty of time to write our queries and build our dashboard before moving on. You can see how the ILM policy was configured [here](/kibana/ilm_policy.txt). This will move the data to a single container in our cluster, es03, that we've identified as "rack_two". We'll also add an alias to our 'nyctrees' index for failover. We'll do this by executing the following: `POST /_aliases{"actions" : [{ "add" : { "index" : "nyctrees", "alias" : "trees" } }]}`. 

When looking at the mappings of our index, we can see they've been set to the default `"type": "text"` as a keyword. Since we're looking to create queries and visualizations that will aggregate certain metrics in the data, we'll have to fix the mappings, then re-import the data. 

First we'll have to delete the current index using `DELETE nyctrees`. Afterwards, we'll create a new index with the same name as `PUT /nyctrees` and apply the mappings as specified [here](/kibana/index_mappings.txt). Important to note that because we have the `config.reload.automatic: true` and `config.reload.interval: 5s` settings configured in our logstash.yml, we can make changes to our pipeline without restarting our logstash container. We will uncomment the `sincedb_clean_after => 0.01` to allow it to clear the CSV from the sincedb log after 15 minutes. 

With our index management completed, we're ready to begin querying our data. 

## Data Queries

For our [first query](/kibana/query_bay_ridge.txt), lets assume we're interested in knowing how many 'Quercus' trees in the Bay Ridge neighborhood have a good healthstatus. Let's also say we're only interested in the most recent information  - 2015. Because we're only looking at attributes that are mapped as keywords, we can use the filter command on each of these to generate an overall count of entries that match these requirements. Our query returns 877 documents, indicating we have just under 1000 trees that fill our requirements. With our scope narrowed, we added some additional exploration with some aggrgation fields. Specifically, we're looking at a statistical breakdown of tree diameters (tree_dbh) as well as a general count of species in the Bay Ridge neighborhood that belong to the 'quercus' genus. 

Our [next query](/kibana/query_neighborhood_income.txt) will focus on sorting neighborhoods by income based on information from 2005. Since we're only intersted in the 'nta_name' and 'income' fields being returned, we'll set our original `"size": 0 and "_source": ["nta_name", "income"]`. We'll then use a filter to only look at `"year.keyword": "2005"`. Then we'll continue to the actual aggregation where we'll query terms on the "nta_name.keyword"

Our [final query](/kibana/query_shoes_guards.txt) will look at getting a count of shoes/guards overall except we're exclusing 'Bay Ridge'. We'll use "must_not" to remove "Bay Ridge" from our query and proceed to aggregate based on terms 'inf_shoes.keyword' and 'inf_guard.keyword'. The output will show there's an overhwelming imbalance in 'No' and 'Yes' entries for shoes across neighborhoods. While there's also an imbalance for the 'guards' attribute, it's not as overwhelming. 

## Data Visualizations

With our three main queries established, we'll create a visualization around each one. 

Our first visualization will be based on our [first query](/kibana/query_bay_ridge.txt). While we couldn't apply as many filters as our initial query, our [final request](/kibana/viz_bay_ridge.txt) has a majority of the same information with the exception of the healthstatus filtering. This is reflected in our visualization below by having slightly higher counts than our initial query. Additionally, the total species displayed is reduced to the top 5. 

![Bay Ridge Quercus Species Visualization](/images/viz_bay_ridge.PNG)

Our next visualization will be based on our [second query](/kibana/query_neighborhood_income). A bar graph would be a good way to visualize this data. From here, we're able to make our visualization match the output from our query without compromising. The below image shows the rankings are the same as [our second query](/kibana/query_neighborhood_income.txt). We can also see our [request return](/kibana/viz_neighborhood_income.txt) has a similar 'aggs' structure to our query. 

![Neighborhood by Income Visualization](/images/viz_neighborhood_income.PNG) 

Our final pair of visualizations will be for the inf_shoes and inf_guard attributes. This will be modeled after our [final query](/kibana/query_shoes_guards.txt). Since the values are binary, a pie chart would be a good way to visualize the proportion within the data. We'll create a filter for our data, then create the pie chart for the appropriate attributes. The request return for [inf_guard](/kibana/viz_inf_guard.txt) and [inf_shoes](/kibana/viz_inf_shoes.txt) are close to our query. 

![inf_guard for Neighborhoods other than Bay Ride](/images/viz_inf_guard.PNG) 
![inf_shoes for Neighborhoods other than Bay Ride](/images/viz_shoes_guard.PNG) 

## Data Dashboard

The final step in this process is to create a dashboard that contains all of these visualizations, with some slight modifications. We've added an additional visualizations that just provides an overall count of trees. While it provides minimul additional insight overall, when combined with neighborhood filters it'll help paint a bigger picture. Next, we changed the ['Quercus' Visualization](/images/viz_bay_ridge.PNG) by removing the neighborhood filter previously. Now, our dashboard has an overall view of our data as a whole, as shown below.

![Dashboard for NYCTrees Data](/images/dashboard.PNG)

From here, we can click on a neighborhood and the dashboard will filter all visualizations to that dashboard. As shown below, we've selected the 'Upper West Side' neighborhood. From here we can see how the data transforms to account for it. 

![Dashboard for Upper West Side Neighborhood](/images/dashboard_filtered.PNG)

## Conclusion

Throughout this project we briefly went over our logstash pipeline that pulled in data from a CSV into an elasticsearch cluster, all within Docker images. Next, we wrote a few queries to analyze our data and give us some insight. Then, we created several visualizations based off of those queries and combined them into a single dashboard. 

For additional information into the NYCTrees database, please see my [other project](https://github.com/kbfoerster/nyctrees).  

# References

