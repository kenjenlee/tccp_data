# NYC Citi Bike Trips

![img](img/NYC%20Citi%20Bike%20Analysis%20Dashboard.png)

**Tools used:** SQL, Google BigQuery, Google Colab ([noteboook link](https://colab.research.google.com/drive/1LFcC0hvfWi4k8n43K47OstPLYEnjgPfZ?usp=sharing)), Python ([Folium](https://python-visualization.github.io/folium/latest/), pandas), Metabase

**Data:** The NYC Citi Bike Trips is a public dataset available in the Google BigQuery [marketplace](https://console.cloud.google.com/marketplace/product/city-of-new-york/nyc-citi-bike). It contains two tables: `citibike_stations` and `citibike_trips`.

**Metabase:** Just like Power BI and Tableau, Metabase is a business intelligence and analytics tool. Unlike them, however, Metabase is [open-source](https://github.com/metabase/metabase).
In this project, Metabase was self-hosted via its [docker image](https://www.metabase.com/docs/latest/installation-and-operation/running-metabase-on-docker)
and [connected to Google BigQuery](https://www.metabase.com/docs/latest/databases/connections/bigquery) via a service account to enable SQL queries.

**SQL Queries:** This project used SQL queries that are then either visualized or downloaded as a csv to be visualized using Python in a Google Colab notebook. An example is the below query, which was used to get the top 10 most common trips between different bike stations both before and after 12pm.

```{SQL}
with all_trips as (
	select cast(start_station_name as string) as start_station_name
		, cast(end_station_name as string) as end_station_name
		, count(*) as value
		, 1 as afternoon
	from `bigquery-public-data.new_york_citibike.citibike_trips`
	where EXTRACT(HOUR FROM stoptime) >= 12 and 
	start_station_name is not null and end_station_name is not null
	and start_station_name <> '' and end_station_name <> ''
	and start_station_name <> end_station_name
	group by start_station_name, end_station_name
	
	union all
	
	select cast(start_station_name as string) as start_station_name
		, cast(end_station_name as string) as end_station_name
		, count(*) as value
		, 0 as afternoon
	from `bigquery-public-data.new_york_citibike.citibike_trips`
	where EXTRACT(HOUR FROM stoptime) < 12 and 
	start_station_name is not null and end_station_name is not null
	and start_station_name <> '' and end_station_name <> ''
	and start_station_name <> end_station_name
	group by start_station_name, end_station_name
)
, rnked as (
	select all_trips.*
	, start_stations.latitude as start_latitude, start_stations.longitude as start_longitude
	, end_stations.latitude as end_latitude, end_stations.longitude as end_longitude
	, rank() over (partition by afternoon order by value desc, start_station_name) as rnk
	from all_trips join `bigquery-public-data.new_york_citibike.citibike_stations` start_stations
	-- station names where latitude or longitude is null are removed
	on all_trips.start_station_name = start_stations.name
	join `bigquery-public-data.new_york_citibike.citibike_stations` end_stations
	on all_trips.end_station_name = end_stations.name
)
select afternoon
	, start_station_name, end_station_name
	, value
	, start_latitude, start_longitude
	, end_latitude, end_longitude
from rnked
where rnk <= 10
order by afternoon, rnk

```

**Interactive Maps using Folium:** The bottom four maps in the dashboard above were created using the `folium` Python library. The code and interative maps are available on [Google Colab](https://colab.research.google.com/drive/1LFcC0hvfWi4k8n43K47OstPLYEnjgPfZ?usp=sharing).