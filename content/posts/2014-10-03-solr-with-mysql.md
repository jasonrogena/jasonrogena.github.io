---
title: Solr with MySQL
date: 2014-10-03
tags: [Solr, MySQL]
---

I work in a Research Institute. My job there involves making Software Systems that work with biological sample data. About six months ago, we started developing systems that involved doing high level searches on the biological sample data we have stored in numerous MySQL databases. These high level searches were being done directly on MySQL with the assistance of wildcards and a bit of cleaning. To be honest, the searches weren't really working like we wanted them to. Some were super slow, others didn't return hits (even the more obvious ones). The cracks really started to show when the size of the datasets on MySQL databases started growing. Something had to be done.

In comes [Solr](lucene.apache.org/solr)... Well Solr really wasn't the first thing we tried. I played with [Elasticsearch](www.elasticsearch.org) and I loved it:

 1. Elasticsearch is dead simple to deploy. It took me about an hour to come up with something tangible enough to show to my boss.
 2. It has great plugins. For instance, [Kibana](http://www.elasticsearch.org/overview/kibana) for visualizations.
 3. Elasticsearch is easily scalable over multiple hosts. Scaling was as simple as deploying Elasticsearch in multiple hosts on the same subnet. The hosts automatically discovered each other (what could possibly go wrong).
 4. Indexing MySQL data is not hard. Well, importing MySQL data might not have been as easy as deploying the Elasticsearch instance but it was close enough.

I had one major problem with Elasticsearch though. Updating the Elasticsearch index when using MySQL as the source seemed like one big hack. I had to create a bash script that was being called by crontab every 5 minutes or so. The bash scripts first checked for the number of records in the Elasticsearch index, then it checked for the number of rows in an indexed MySQL database and finally, it fetched any new MySQL rows. There were simply too many moving parts. Crontab, files in tmp storing timestamps, updating rivers and streams in [River JDBC](https://github.com/jprante/elasticsearch-river-jdbc).... It simply wasn't working for me.

So a few weeks ago I decided to switch to Solr and I must say I'm loving it so far:

 1. No need to install third-party libraries to make it work with MySQL.
 2. Extremely powerful when it comes to defining the structure of the MySQL datasets.
 3. Somewhat easy to deploy.

However, be warned. Solr is not as easy to configure as Elasticsearch. It might take you a day to wrap your head around how it works.

> **Note:**
> I will not explain how to install Solr. Mainly because [somebody else](http://mjanja.co.ke) did it for me. There are numerouse posts out there you can use for this. [Here's](https://www.digitalocean.com/community/tutorials/how-to-install-solr-on-ubuntu-14-04) one for example. I will also not go through step-by-step configuration of Solr with MySQL. Solr's [official wiki page](http://wiki.apache.org/solr/DataImportHandler) for the DataImportHandler is pretty comprehensive on this topic. I will mainly consentate on the odd behaviours in Solr that made configuration a bit time consuming.

 
### My Set Up

My setup looks like this:

 -  The server runs Ubuntu 14.04
 -  I'm using Tomcat7 as the servlet container
 -  Solr is installed in /usr/share/tomcat7/solr. I'll be calling this **SOLR_HOME**
 -  I have a Solr core called samples in /usr/share/tomcat7/solr/cores/samples
 -  The samples core's config directory is /usr/share/tomcat7/solr/cores/samples/config

After installation and configuration, my Solr core home had the following files:

 -  solrconfig.xml. This file is the main configuration file for you core
 -  schema.xml. This file basically handles datatypes and associates fields to datatypes
 -  data-config.xml. This file contains the MySQL queries responsible for importing data from MySQL into Solr.

#### 1. The MySQL Connector

Java needs the MySQL [Connector Library](http://dev.mysql.com/downloads/connector/j/5.1.html) for it to connect to MySQL. There are two ways you can import the Connector Library into solr.

 1.  Dumping the lib into Tomcat's common class loader directory
 2.  Manually importing the library in your solrconfig.xml file

If you plan on having more than one core that will import data from MySQL, I'd prefer you use the first method. Manually importing the library in each of your cores might lead to Java throwing [PermGen Space Errors](http://www.integratingstuff.com/2011/07/24/understanding-and-avoiding-the-java-permgen-space-error/).


#### 2. Importing the Necessary Solr Libs

Solr will need the dataimporthandler library for it to be able to import data from MySQL. I created a directory called **dist** in my SOLR_HOME, copied the solr-dataimporthandler.jar and solr-dataimporthandler-extras.jar from Solr example provided in the installation zip/tarball file into this **dist** directory then pointed my solrconfig.xml files to these libraries:

```xml
<lib dir="../../dist/" regex="solr-dataimporthandler-\d.*\.jar" />
<lib dir="../../dist/" regex="solr-dataimporthandler-extras-\d.*\.jar" />
```

To be safe, I added the lines after the last lib tag in the file.


#### 3. MySQL Authentication and Datasources

The [dataimporthandler wiki](https://wiki.apache.org/solr/DataImportHandler) recommends you define your MySQL datasources in your data-config.xml file. However, this did not work for me. Solr was not able to read the data-config.xml file whenever I did this. I therefore had to move the datasources to the solrconfig.xml file. Here's how the DataImportHandler looks in my solrconfig.xml file:

```xml
<requestHandler name="/handlers" class="org.apache.solr.handler.dataimport.DataImportHandler"><!-- make sure the is there in the name otherwise this wont work -->
  <lst name="defaults">
     <str name="config">data-config.xml</str><!-- here is where you specify where the schema is  -->
     <lst name="datasource">
        <str name="name">data_source_0</str>
        <str name="driver">com.mysql.jdbc.Driver</str>
        <str name="url">jdbc:mysql://db_host0/db0?zeroDateTimeBehavior=convertToNull</str>
        <str name="user">user0</str>
        <str name="password">pw0</str>
     </lst>
     <lst name="datasource">
        <str name="name">data_source_1</str>
        <str name="driver">com.mysql.jdbc.Driver</str>
        <str naqme="url">jdbc:mysql://db_host1/db1?zeroDateTimeBehavior=convertToNull</str>
        <str name="user">user1</str>
        <str name="password">pass1</str>
     </lst>
  </lst>
</requestHandler>
```

You'll also want to add the **zeroDateTimeBehavior** variable in your MySQL connection URL to cater for MySQL **datetime** columns that have the '00:00:0000' value. The Java datatype is not able to handle such a value.


#### 4. My data-config.xml File

Here's a simplified version of my data-config.xml file:

```xml
<dataConfig>
 <document name='samples'>
   <entity name="sample"
           dataSource="data_source_0"
           pk="sample_id"
           query="select a.id as sample_id, a.name, a.box_id, b.name as loc
                 from samples as a
                 left join locations as b on a.loc_id = b.id"
           preImportDeleteQuery="*:*">
     <field column="SAMPLE_ID" name="sample_id" />
     <field column="NAME" name="sample_name" />
     <field column="LOC" name="sample_loc" />
     <field column="BOX_ID" name="box_id" />
     <entity name="storage_box"
             dataSource="data_source_1"
             pk="box_id"
             onError="continue"
             query="select id as box_id, a.name as box_name
                   from boxes
                   where id = '${sample.box_id}'"
             preImportDeleteQuery="*:*">
       <field column="NAME" name="box_name" />
     </entity>
   </entity>
 </document>
<dataConfig>
```

Pay special attention on how joined the root entity (sample) with it's child entity (storage_box). I recommend adding the onError attribute to your child entity. Refer to the dataimport WiKi for possible value of this entity. Make sure your parent entity's primary key is surrounded in quotes in the child entity's MySQL query. Do this even if the primary key is an integer. I did this as a form of a failsafe in cases where the primary key is empty.

Also note that I used joins instead of entities whever I could. Using sub-entities would have improve the performance during importation and indexing. On the other hand, it's simpler to use joins.

Solr was unable to import the data-config.xml file whenever I put comments in it. This is a bit odd, and maybe I'm the one who doesn't know how to comment on xml. My data-config.xml file is therefore free of comments. 

Another issue you should watch out for is Solr will not populate fields that have names that confilct with words reserved by Solr. In my case, Solr was unable to import data for fields named: **'feature'** and **'size'**. You should therefore check which of your fields are blank after importing data from MySQL. 

It sucks Solr in most cases keeps quiet whenever it's unable to import the data-config.xml file. One way, however, you can tell if your data-config.xml file has an issue is by checking if you can see the root entity your core's Dataimport page:

![image showing loaded entities from a successfully imported data-config.xml file](/images/2014-10-03-solr-with-mysql_1.png)

#### 5. Defining My Datatypes

The last major thing I needed to do was to define my datatypes in the schema.xml file. Note that the default schema.xml has most if not all the datatypes you'll need. Here are the lines I needed to add to make it work with the data-config.xml file above:

```xml
<uniqueKey>sample_id</uniqueKey>
<field name="sample_id" type="string" indexed="true" required="true" stored="true" />
<field name="sample_name" type="text_general" indexed="true" stored="true" default="NULL" />
<field name="sample_loc" type="text_general" indexed="true" stored="true" default="" />
<field name="box_id" type="string" indexed="true" stored="true" default="" />
<field name="box_name" type="text_general" indexed="true" stored="true" default="" />
```

To be on the safe side, you might want to initialize your integer fields with the **'string'** datatype. This is because Solr will not be able to import null values in fields that are of type **'int'**.


### Conclusion

Solr has some undocumented weird behaviours when connected to MySQL. Maybe someone working on Solr will someday read this and clarify on these. However, for now, it's important for me to iterate that **if you have a relational database that is normalized, you're better off going with Solr**. I will, in another post, blog on the optimizations I did (especially in the schema.xml file) to make Solr work well in my setup. Feel free to comment on any mistakes (misinformation or grammatical) that I might have made in this post.
