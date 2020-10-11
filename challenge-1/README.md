# Challenge 1 - Data Mart

For the data mart challenge I choose to ingest all the tables as-is directly into bigquery and made all the aggregations and transformations with SQL

I`ve made avaible this BI base inside a public metabase you can check [**here**](http://35.202.47.4/public/dashboard/72bc0b33-f4cb-420b-8eb7-1062430ba861)


# Challenge 1 - Pipeline Architecture Design

For this challenge I have assumed that each asset avaible in BASE A could be a kafka topic and then designed a data ingestion pipeline to handle the messages untill the data is avaible for analysts inside the BI tool, metabase in this case

![image](Data-Lake-Architecture.png)

## Batch Pipeline

All the messages transitioning inside Kafka can be easily persisted in a bucket (cloud storage, s3 or HDFS) using Kafka Connect, an open-source confluent application thought to compose the Kafka ecosystem.

After the data is persisted in its raw-format, if it's size cant fit in memory or some real complex transformation is needed, a spark based solution can be used to do the hard work. I indicated some cloud solutions (EMR, or Data Proc) that could easily make an ad-hoc cluster available to that matter. You can also see the Data Proc exchanging messages with the catalog - and that's important to make data governance possible.

Once all the hard work is done by the spark cluster - if needed - and wrote back to the storage, the data should be ready to be ingested to a database like BigQuery or Athena. In GCP I used a cloud function to apply the schema, held by the catalog. AWS has a solution to that problem - glue - a severless catalog application that can not only hold and apply the schema for each asset but perform several data-quality checks.

Getting into the relational database takes us almost there in the road to the data marts, but we know that our analytics needs are a little more complex than just some asset tables, we commonly need to transform the data, change names and types, aggregate and pivot the data so it can be in a ready to be queried format.

Thats were the catalog comes in again, orchestrating the data from the called staging area (were the ingested data comes in) to the data marts - and thats done with several SQL transformations and aggregations.

One quick example: Let's say that our leads data just got from the bucket to bigquery on the dataset `Leads_staging`, when that happens the catalog is triggered applying some SQL aggregation with our contracts data and write the query results to the `Funnel_data_mart` dataset. The `funnel_data_mart` is the one that our analysts will have access to!

Nice! Now our events are in a relational database ready to be queried by our analysts and shown to the rest of the company by some BI solution, like Metabase. Depending on the required transformations this pipeline can perform ELT near real-time, but you should note that there is a cost increase related to the frequency of the extraction runs.

#### Data Governance

You can apply data governance on this architecture in many levels, I would suggest that only data engineers and scientists should have access to raw data and staging area, as BigQuery and Athena let you set permission on datasets level a data scientists of the origination squad may only have access to origination data. Other permission can be done directly on metabase, assigning people to groups and groups to datasets all you governance needs should be covered!

## Streaming Pipeline

If Passei Direto had Streaming needs for some of this assets:

As we need this data to be available quickly we treat if a little bit differently.

All the transformation is done with Kafka Streams, a framework design to this principle - transform data in streaming pipelines. As you should have thought, this certainly reduce the range of transformations we can perform on the data before it reaches the data marts.

With this design the data should be in a ready to be ingested and consume format after the transformation, so if you want an aggregated metric of sales per region - the schema of the messages after the Kafka Streams pipeline should be really close to the presented to the company, and once it is, we use Kafka Connect to insert this metric directly inside of the database making it ready to be consumed in just a few seconds after the event occurred!