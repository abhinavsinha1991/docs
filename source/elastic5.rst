Kafka Connect Elastic 5
=======================

A Connector and Sink to write events from Kafka to Elastic Search using `Elastic4s <https://github.com/sksamuel/elastic4s>`__ client.
The connector converts the value from the Kafka Connect SinkRecords to Json and uses Elastic4s's JSON insert functionality to index.

The Sink creates an Index and Type corresponding to the topic name and uses the JSON insert functionality from Elastic4s.

The Sink supports:

1. Auto index creation at start up.
2. :ref:`The KCQL routing querying <kcql>` - Topic to index mapping and Field selection.
3. Auto mapping of the Kafka topic schema to the index.

Prerequisites
-------------

- Confluent 3.2
- Elastic Search 5.4
- Java 1.8
- Scala 2.11

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Elastic Setup
~~~~~~~~~~~~~

Download and start Elastic search.

.. sourcecode:: bash

    curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.4.0.tar.gz
    tar -xvf elasticsearch-5.4.0.tar.gz
    cd elasticsearch-5.4.0/bin
    ./elasticsearch --cluster.name elasticsearch


Sink Connector QuickStart
-------------------------

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.


Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for Elastic.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create elastic-sink < conf/elastic-sink.properties

    #Connector name=`elastic-sink`
    name=elastic-sink
    connector.class=com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    connect.elastic.url=localhost:9300
    connect.elastic.cluster.name=elasticsearch
    tasks.max=1
    topics=TOPIC1
    connect.elastic.sink.kcql=INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1
    #task ids: 0

The ``elastic-sink.properties`` file defines:

1. The name of the connector.
2. The class containing the connector.
3. The name of the cluster on the Elastic Search server to connect to.
4. The max number of task allowed for this connector.
5. The Source topic to get records from.
6. :ref:`The KCQL routing querying. <kcql>`

If you switch back to the terminal you started the Connector in you should see the Elastic Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    elastic-sink

.. sourcecode:: bash

    [2016-05-08 20:56:52,241] INFO

        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           ________           __  _      _____ _       __
          / ____/ /___ ______/ /_(_)____/ ___/(_)___  / /__
         / __/ / / __ `/ ___/ __/ / ___/\__ \/ / __ \/ //_/
        / /___/ / /_/ (__  ) /_/ / /__ ___/ / / / / / ,<
       /_____/_/\__,_/____/\__/_/\___//____/_/_/ /_/_/|_|


    by Andrew Stevenson
           (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:33)

    [2016-05-08 20:56:52,327] INFO [Hebe] loaded [], sites [] (org.elasticsearch.plugins:149)
    [2016-05-08 20:56:52,765] INFO Initialising Elastic Json writer (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:31)
    [2016-05-08 20:56:52,777] INFO Assigned List(test_table) topics. (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:33)
    [2016-05-08 20:56:52,836] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@69b6b39 finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)

Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``id`` field of type int
and a ``random_field`` of type string.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic TOPIC1 \
     --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},
    {"name":"random_field", "type": "string"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"id": 999, "random_field": "foo"}
    {"id": 888, "random_field": "bar"}


Check for records in Elastic Search
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now if we check the logs of the connector we should see 2 records being inserted to Elastic Search:

.. sourcecode:: bash

    [2016-05-08 21:02:52,095] INFO Flushing Elastic Sink (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:73)
    [2016-05-08 21:03:52,097] INFO No records received. (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:63)
    [2016-05-08 21:03:52,097] INFO org.apache.kafka.connect.runtime.WorkerSinkTask@69b6b39 Committing offsets (org.apache.kafka.connect.runtime.WorkerSinkTask:187)
    [2016-05-08 21:03:52,097] INFO Flushing Elastic Sink (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:73)
    [2016-05-08 21:04:20,613] INFO Elastic write successful for 2 records! (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:77)

If we query Elastic Search for ``id`` 999:

.. sourcecode:: bash

    curl -XGET 'http://localhost:9200/INDEX_1/_search?q=id:999'

    {
        "took": 45,
        "timed_out": false,
        "_shards": {
            "total": 5,
            "successful": 5,
            "failed": 0
        },
        "hits": {
            "total": 1,
            "max_score": 1.2231436,
            "hits": [{
                "_index": "test_table",
                "_type": "test_table",
                "_id": "AVMY4eZXFguf2uMZyxjU",
                "_score": 1.2231436,
                "_source": {
                    "id": 999,
                    "random_field": "foo"
                }
            }]
        }
    }

Features
--------

1. Auto index creation at start up.
2. Topic to index mapping.
3. Auto mapping of the Kafka topic schema to the index.
4. Field selection

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`__
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Elastic Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <index> SELECT <fields> FROM <source topic> [WITHDOCTYPE=<your_document_type>] [WITHINDEXSUFFIX=<your_suffix>]

`WITHDOCTYPE` allows you to associate a document type to the document inserted.
`WITHINDEXSUFFIX` allows you to specify a suffix to your index and we support date format. All you have to say is '_suffix_{YYYY-MM-dd}'

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to indexA
    INSERT INTO indexA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to indexB
    INSERT INTO indexB SELECT x AS a, y AS b and z AS c FROM topicB PK y

This is set in the ``connect.elastic.sink.kcql`` option.

Auto Index Creation
~~~~~~~~~~~~~~~~~~~

The Sink will automatically create missing indexes at startup. The Sink use elastic4s, more details can be found
`here <https://github.com/sksamuel/elastic4s>`__

Configurations
--------------

``connect.elastic.url``

Url of the Elastic cluster.

* Data Type : string
* Importance: high
* Optional  : no


``connect.elastic.sink.kcql``

Kafka connect query language expression. Allows for expressive table to topic routing, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1

* Data type : string
* Importance: high
* Optional  : no


``connect.elastic.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.cassandra.max.retries``
option. The ``connect.cassandra.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Default: ``throw``

``connect.elastic.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.cassandra.error.policy`` is set to ``retry``.

* Type: string
* Importance: high
* Default: 10

``connect.elastic.xpack.settings``

Enables secure connection. `here <https://www.elastic.co/products/x-pack/security>`__ .By providing a value for the entry the sink will end up creating a secure connection.
The entry is a `;` separated list of key=value sequence

Example:

.. sourcecode:: javascript

    connect.elastic.xpack.settings=xpack.security.user=transport_client_user:changeme;xpack.ssl.key=/path/to/client.key;xpack.ssl.certificate=/path/to/client.crt

* Type: string
* Importance: medium
* Default: null
* Optional: yes

``connect.elastic.xpack.plugins``

Provides the list of plugins to enable.
The entry is a `;` separated list of full class path (The classes need to derive from `org.elasticsearch.plugins.Plugin`)


* Type: string
* Importance: medium
* Default: null
* Optional: yes


``connect.elastic.write.timeout``

Specifies the wait time for pushing the records to ES.

* Data type : long
* Importance: low
* Optional  : yes
* Default   : 300000 (5mins)

``connect.progress.enabled``

Enables the output for how many records have been processed.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

Example
~~~~~~~

.. sourcecode:: bash

    name=elastic-sink
    connector.class=com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    connect.elastic.url=localhost:9300
    connect.elastic.cluster.name=elasticsearch
    tasks.max=1
    topics=test_table
    connect.elastic.sink.kcql=INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

Elastic Search is very flexible about what is inserted. All documents in Elasticsearch are stored in an index. We do not
need to tell Elasticsearch in advance what an index will look like (eg what fields it will contain) as Elasticsearch will
adapt the index dynamically as more documents are added, but we must at least create the index first. The Sink connector
automatically creates the index at start up if it doesn't exist.

The Elastic Search Sink will automatically index if new fields are added to the Source topic, if fields are removed
the Kafka Connect framework will return the default value for this field, dependent of the compatibility settings of the
Schema registry.


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
