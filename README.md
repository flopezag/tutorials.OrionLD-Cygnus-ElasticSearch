# tutorials.OrionLD-Cygnus-ElasticSearch
[![FIWARE Banner](https://fiware.github.io/tutorials.Historic-Context-Flume/img/fiware.png)](https://www.fiware.org/developers)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/)

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Historic-Context-Flume.svg)](https://opensource.org/licenses/MIT)

[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware-cygnus)
<br/> [![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

<!-- prettier-ignore -->
FIWARE tutorial to check how to use [FIWARE OrionLD]() with [FIWARE Cygnus](https://fiware-cygnus.readthedocs.io/en/latest),
through NGSIv2 subscription and persist the data into [ElasticSearch DB](https://www.elastic.co).


## Contents

<details>
<summary><strong>Details</strong></summary>

-   [Data Persistence](#data-persistence-using-apache-flume)
-   [Architecture](#architecture)
-   [Prerequisites](#prerequisites)
    -   [Docker and Docker Compose](#docker-and-docker-compose)
    -   [Cygwin for Windows](#cygwin-for-windows)
-   [Start Up](#start-up)
-   [ElasticSearch - Persisting Context Data into a Database](#elasticsearch---persisting-context-data-into-a-database)
    -   [ElasticSearch - Database Server Configuration](#elasticsearch---database-server-configuration)
    -   [ElasticSearch - Cygnus Configuration](#elasticsearch---cygnus-configuration)
    -   [ElasticSearch - Start up](#elasticsearch---start-up)
        -   [Checking the Cygnus Service Health](#checking-the-cygnus-service-health-2)
        -   [Generating Context Data](#generating-context-data-2)
        -   [Subscribing to Context Changes](#subscribing-to-context-changes-2)
    -   [ElasticSearch - Reading Data from a database](#elasticsearch---reading-data-from-a-database)
        -   [Show Available Databases on the ElasticSearch server](#show-available-databases-on-the-elasticsearch-server)
        -   [Read Historical Context from the ElasticSearch server](#read-historical-context-from-the-elasticsearch-server)
-   [Next Steps](#next-steps)

</details>

# Data Persistence using Apache Flume

Previous tutorials have introduced a set of IoT Sensors (providing measurements of the state of the real world), and two
FIWARE Components - the **Orion Context Broker** and an **IoT Agent**. This tutorial will introduce a new data
persistence component - FIWARE **Cygnus**.

The system so far has been built up to handle the current context, in other words it holds the data entities defining
the state of the real-world objects at a given moment in time.

From this definition you can see - context is only interested in the **current** state of the system It is not the
responsibility of the existing components to report on the historical state of the system, the context is based
on the last measurement each sensor has sent to the context broker.

In order to do this, we will need to extend the existing architecture to persist changes of state into a database
whenever the context is updated.

Persisting historical context data is useful for big data analysis - it can be used to discover trends, or data can be
sampled and aggregated to remove the influence of outlying data measurements. However, within each Smart Solution, the
significance of each entity type will differ and entities and attributes may need to be sampled at different rates.

Since the business requirements for using context data differ from application to application, there is no one standard
use case for historical data persistence - each situation is unique - it is not the case that one size fits all.
Therefore rather than overloading the context broker with the job of historical context data persistence, this role has
been separated out into a separate, highly configurable component - **Cygnus**.

As you would expect, **Cygnus**, as part of an Open Source platform, is technology agnostic regarding the database to be
used for data persistence. The database you choose to use will depend upon your own business needs.

However, there is a cost to offering this flexibility - each part of the system must be separately configured and
notifications must be set up to only pass the minimal data required as necessary.

# Architecture

This application makes use of two FIWARE components: 
- The [OrionLD Context Broker](https://fiware-orion.readthedocs.io/en/latest/), 
- The [Cygnus Generic Enabler](https://fiware-cygnus.readthedocs.io/en/latest/) for persisting context data to a
  database.
  
Additional databases are now involved - the OrionLD Context Broker relies on [MongoDB](https://www.mongodb.com/)
technology to keep persistence of the information it hold, and we will be persisting our historical context data
another database, **ElasticSearch**.

Therefore, the overall architecture will consist of the following elements:

-   Two **FIWARE Generic Enablers**:
    -   The FIWARE [OrionLD Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests
        using [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/gitlab/NGSI-LD/NGSI-LD/raw/master/spec/updated/full_api.json)
        and [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2).
    -   FIWARE [Cygnus](https://fiware-cygnus.readthedocs.io/en/latest/) which will subscribe to context changes and
        persist them into a database (**MySQL** , **PostgreSQL** or **MongoDB**)
-   One or two of the following **Databases**:
    -   The underlying [MongoDB](https://www.mongodb.com/) database:
        -   Used by the **Orion Context Broker** to hold context data information such as data entities, subscriptions
            and registrations
    -   An additional [ElasticSearch](https://www.elastic.co) database:
        -   Potentially used as a data sink to hold historical context data.

Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run
from exposed ports.

# Prerequisites

## Docker and Docker Compose

To keep things simple all components will be run using [Docker](https://www.docker.com). **Docker** is a container
technology which allows to different components isolated into their respective environments.

-   To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
-   To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
-   To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A series of
[YAML files](https://github.com/FIWARE/tutorials.Historic-Context-Flume/tree/master/docker-compose) are used configure
the required services for the application. This means all container services can be brought up in a single command.
Docker Compose is installed by default as part of Docker for Windows and Docker for Mac, however Linux users will need
to follow the instructions found [here](https://docs.docker.com/compose/install/)

You can check your current **Docker** and **Docker Compose** versions using the following commands:

```console
docker-compose -v
docker version
```

Please ensure that you are using Docker version 18.03 or higher and Docker Compose 1.21 or higher and upgrade if
necessary.

## Cygwin for Windows

We will start up our services using a simple Bash script. Windows users should download [cygwin](http://www.cygwin.com/)
to provide a command-line functionality similar to a Linux distribution on Windows.

### http

**http** is a command line HTTP client, similar to curl or wget, with JSON support, syntax highlighting, persistent
sessions, and wget-like downloads with an expressive and intuitive syntax. `http` can be installed on each
operating system. Follow the instructions described [here](https://httpie.io/docs#installation).

# Start Up

Before you start you should ensure that you have obtained or built the necessary Docker images locally. Please clone the
repository and create the necessary images by running the commands as shown:

```console
git clone https://github.com/flopezag/tutorials.OrionLD-Cygnus-ElasticSearch.git
cd tutorials.OrionLD-Cygnus-ElasticSearch

./services create
```

> **Note** The initial creation of Docker images can take up to three minutes

Thereafter, all services can be initialized from the command-line by running the
[services](https://github.com/flopezag/tutorials.OrionLD-Cygnus-ElasticSearch/blob/main/services)
Bash script provided within the repository:

```console
./services <command>
```

Where `<command>` will be help, start, stop or create.

> :information_source: **Note:** If you want to clean up and start over again you can do so with the following command:
>
> ```console
> ./services stop
> ```

# ElasticSearch - Persisting Context Data into a Database

To persist historic context data into an alternative database such as **ElasticSearch**, we will need an additional
container which hosts the ElasticSearch server - the default Docker image for this data can be used. It is important
to mention that the ElasticSearch Sink has been tested with the versions 6.3 and 7.6 of Elasticsearch. The ElasticSearch
instance is listening on the standard `9200` port, and the overall architecture can be seen below:

![](https://fiware.github.io/tutorials.Historic-Context-Flume/img/cygnus-elasticsearch.png)

We now have a system with two databases, since the MongoDB container is still required to hold data related to the Orion
Context Broker.

## ElasticSearch - Database Server Configuration

```yaml
elasticsearch-db:
    image: elasticsearch:${ELASTICSEARCH_VERSION}
    hostname: elasticsearch
    container_name: db-elasticsearch
    expose:
      - "${ELASTICSEARCH_PORT}"
    ports:
      - "${ELASTICSEARCH_PORT}:${ELASTICSEARCH_PORT}"
    networks:
      - default
    volumes:
      - es-db:/usr/share/elasticsearch/data
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: changeme
      # Use single node discovery in order to disable production mode and avoid bootstrap checks.
      # see: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
      discovery.type: single-node
    healthcheck:
      test: curl http://localhost:${ELASTICSEARCH_PORT} >/dev/null; if [[ $$? == 52 ]]; then echo 0; else echo 1; fi
      interval: 30s
      timeout: 10s
      retries: 5
```

The `elasticsearch-db` container is listening on only one port:

-   Port `9200` (ELASTICSEARCH_PORT) is the default port for a ElasticSearch server.

> Note: If you want to deploy multiplie ElasticSearch nodes, it is needed to specify the corresponding port, by default
> it should be `9300` (ELASTICSEARCH_NODES_COMMUNICATION_PORT) for a ElasticSearch for nodes communication.

The `elasticsearch-db` container is driven by environment variables as shown:

| Key              | Value.              | Description                                                             |
| ---------------- | ------------------- | ----------------------------------------------------------------------- |
| ES_JAVA_OPTS     | `-Xmx256m -Xms256m` | Setting JVM heap size. It is not recommended in production environment. |
| ELASTIC_PASSWORD | `changeme`          | Password for the PostgreSQL database user.                              |

> :information_source: **Note:** Passing the Password in plain text environment variables like this is a security risk.
> Whereas this is acceptable practice in a tutorial, for a production environment, you can avoid this risk by applying
> [Docker Secrets](https://blog.docker.com/2017/02/docker-secrets-management/)

## ElasticSearch - Cygnus Configuration

```yaml
cygnus:
    image: fiware/cygnus-ngsi:${CYGNUS_VERSION}
    hostname: cygnus
    container_name: fiware-cygnus
    depends_on:
      - elasticsearch-db
    networks:
      - default
    expose:
      - "${CYGNUS_API_PORT}"
      - "${CYGNUS_ELASTICSEARCH_SERVICE_PORT}"
    ports:
      - "${CYGNUS_ELASTICSEARCH_SERVICE_PORT}:${CYGNUS_ELASTICSEARCH_SERVICE_PORT}" # localhost:5058
      - "${CYGNUS_API_PORT}:${CYGNUS_API_PORT}" # localhost:5088
    environment:
      - "CYGNUS_ELASTICSEARCH_HOST=elasticsearch-db:${ELASTICSEARCH_PORT}"
      - "CYGNUS_ELASTICSEARCH_PORT=${CYGNUS_ELASTICSEARCH_SERVICE_PORT}"
      - "CYGNUS_ELASTICSEARCH_SSL=false"
      - "CYGNUS_API_PORT=${CYGNUS_API_PORT}" # Port that Cygnus listens on for operational reasons
      - "CYGNUS_LOG_LEVEL=DEBUG" # The logging level for Cygnus
    healthcheck:
      test: curl --fail -s http://localhost:${CYGNUS_API_ADMIN_PORT}/v1/version || exit 1
```

The `cygnus` container is listening on two ports:

-   The Subscription Port for Cygnus, CYGNUS_ELASTICSEARCH_SERVICE_PORT, `5058` is where the service will be listening
    for notifications from the Orion context broker.
-   The Management Port for Cygnus, CYGNUS_API_PORT, - `5080` is exposed purely for tutorial access - so that cUrl or
    Postman can make provisioning commands without being part of the same network.

The `cygnus` container is driven by environment variables as shown:

| Key                            | Value         | Description                                                                    |
| ------------------------------ | ------------- | ------------------------------------------------------------------------------ |
| CYGNUS_ELASTICSEARCH_HOST      | `elasticsearch-db:${ELASTICSEARCH_PORT}` | Hostname and port of the ElasticSearch server used to persist historical context data. |
| CYGNUS_ELASTICSEARCH_PORT      | `${CYGNUS_ELASTICSEARCH_SERVICE_PORT}`   | Port, by default `5058`, that Cygnus used to persist historical context data.          |
| CYGNUS_ELASTICSEARCH_SSL       | `false`                                  | SSL is not configured for communication.                                               |
| CYGNUS_API_PORT                | `${CYGNUS_API_PORT}`                     | Port, by default `5080` that Cygnus listens on for operational reasons.                |
| CYGNUS_LOG_LEVEL               | `DEBUG`                                  | The logging level for Cygnus                                                           |

## ElasticSearch - Start up

To start the system, run the following command:

```console
./services start
```

### Checking the Cygnus Service Health

Once Cygnus is running, you can check the status by making an HTTP request to the exposed `CYGNUS_API_PORT` port.
If the response is blank, this is usually because Cygnus is not running or is listening on another port.

#### Request:

```console
curl -X GET \
  'http://localhost:5080/v1/version'
```

#### Response:

The response will look similar to the following:

```json
{
    "success": "true",
    "version": "1.18.0_SNAPSHOT.etc"
}
```

> **Troubleshooting:** What if the response is blank ?
>
> -   To check that a docker container is running try
>
> ```bash
> docker ps
> ```
>
> You should see several containers running. If `cygnus` is not running, you can restart the containers as necessary.

### Subscribing to Context Changes

Once a dynamic context system is up and running, we need to inform **Cygnus** of changes in context. This is done by
making a subscription to the Orion Context Broker. Context brokers may offer additional custom payload formats
(typically prefixed with an `x-`). The Orion-LD broker offers a backward compatible **NGSI-v2** 
(`/ngsi-ld/v1/subscriptions`) payload option for legacy systems. This subscription type will fire when the `filling`
level is below 0.4. The `format` attribute has been altered to inform the subscriber using NGSI-v2 normalized format.

```bash
curl -L -X POST 'http://localhost:1026/ngsi-ld/v1/subscriptions/' \
-H 'Content-Type: application/ld+json' \
--data-raw '{
  "description": "Notify me of low feedstock on Farm:001",
  "type": "Subscription",
  "entities": [{"type": "FillingLevelSensor"}],
  "watchedAttributes": ["filling"],
  "q": "filling<0.4;controlledAsset==urn:ngsi-ld:Building:farm001",
  "notification": {
    "attributes": ["filling", "controlledAsset"],
    "format": "x-ngsiv2-normalized",
    "endpoint": {
      "uri": "http://cygnus:5058/notify",
      "accept": "application/json"
    }
  },
   "@context": "http://context-provider:3000/data-models/ngsi-context.jsonld"
}'
```

As you can see, the database used to persist context data has no impact on the details of the subscription. It is the
same for each database. The response will be **201 - Created**

#### Subscription Payload:

When a `low-stock-farm001-ngsiv2` event is fired, the response is a normalized NGSI-v2 payload as shown:

```json
{
    "subscriptionId": "urn:ngsi-ld:Subscription:5fd1f31e8b9b83697b855a5d",
    "data": [
        {
            "id": "urn:ngsi-ld:Device:filling001",
            "type": "https://uri.etsi.org/ngsi-ld/default-context/FillingLevelSensor",
            "https://w3id.org/saref#fillingLevel": {
                "type": "Property",
                "value": 0.33,
                "metadata": {
                    "unitCode": "C62",
                    "accuracy": {
                        "type": "Property",
                        "value": 0.05
                    },
                    "observedAt": "2020-12-10T10:11:57.000Z"
                }
            },
            "https://uri.etsi.org/ngsi-ld/default-context/controlledAsset": {
                "type": "Relationship",
                "value": "urn:ngsi-ld:Building:farm001",
                "metadata": {
                    "observedAt": "2020-12-10T10:11:57.000Z"
                }
            }
        }
    ]
}
```

As can be seen, by default the attributes are returned using URN long names. It is also possible to request that the
Orion-LD context broker pre-applies a compaction operation to the payload.

-   `x-nsgiv2-keyValues` - Key-value pairs with URN attribute names
-   `x-nsgiv2-keyValues-compacted` - Key-value pairs with short name attribute aliases
-   `x-ngsiv2-normalized` - NGSI-v2 normalized payload with URN attribute names
-   `x-ngsiv2-normalized-compacted`- NGSI-v2 normalized payload pairs with short name attribute aliases

The set of available custom formats will vary between Context Brokers.

### Generating Context Data

For the purpose of this tutorial, we must be monitoring a system where the context is periodically being updated. The
dummy IoT Sensors can be used to do this. Open the device monitor page at `http://localhost:$TUTORIAL_APP_PORT/device/monitor`
and unlock a **Smart Door** and switch on a **Smart Lamp**. Remember that the variable `TUTORIAL_APP_PORT` is defined
in the `.env` file. This can be done by selecting an appropriate the command from the drop down list and pressing the
`send` button. The stream of measurements coming from the devices can then be seen on the same page:

## ElasticSearch - Reading Data from a database

To read ElasticSearch data from the command-line, we will execute a set of HTTP request to get the data. If you want
to know how create specific queries to access the data, you can take a look to the [Elastic Search API](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/search-your-data.html)
for the current version. 

### Show Available Databases on the ElasticSearch server

To show the list of available databases, run the statement as shown:

### Query:

```console
curl -XGET 'localhost:9200/_cat/indices?v&pretty'
```

### Result:

```
health status index                                        uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   cygnus-openiot--motion-003-motion-2021.04.16 lh3Y2NT8SJaqNrkLDlmFOg   1   1        288            0    119.4kb        119.4kb
yellow open   cygnus-openiot--lamp-004-lamp-2021.04.16     XZ4zor2rTkeXBtO-uvJ0KA   1   1        451            0     92.8kb         92.8kb
yellow open   cygnus-openiot--motion-001-motion-2021.04.16 TSMdVhXhSeOygOGAhgqHtw   1   1         16            0      267kb          267kb
yellow open   cygnus-openiot--motion-004-motion-2021.04.16 K_RN-eVIQGGDlRVNpmiDow   1   1        400            0     95.4kb         95.4kb
yellow open   cygnus-openiot--lamp-002-lamp-2021.04.16     Nveh6Ka1TQ2mDcxqh_YvXg   1   1        522          102    181.9kb        181.9kb
yellow open   cygnus-openiot--lamp-003-lamp-2021.04.16     Mo5paVjeQwabf6PjGuNzWw   1   1        627            0    109.1kb        109.1kb
yellow open   cygnus-openiot--door-001-door-2021.04.16     jWZccKX3Tjayd2nUrjuxSw   1   1         32            4    380.8kb        380.8kb
yellow open   cygnus-openiot--lamp-001-lamp-2021.04.16     9L-1ZbX_TxmgyQXpQN_6lA   1   1        957            0    170.5kb        170.5kb
yellow open   cygnus-openiot--motion-002-motion-2021.04.16 mvvS-IGvR2C4cy7KryJy1g   1   1        440            0    108.6kb        108.6kb
```

The result includes the complete list of indexes as well the number of registries and the store size of each of the
table. Once we have this information, we can request the information of each of the sensor:

### Read Historical Context from the ElasticSearch server

Once running a docker container within the network, it is possible to obtain information about the running database.
In our case we access to the `motion003` entity, and we limit the results only to 2.

### Query:

```console
curl -XGET 'localhost:9200/_sql?format=json' -H 'Content-Type: application/json' -d'
{
  "query": " select * from \"cygnus-openiot--motion-003-motion-2021.04.16\" limit 2 "
}'
```

### Result:

```json
{
  "columns": [
    {
      "name": "attrMetadata.name",
      "type": "text"
    },
    {
      "name": "attrMetadata.type",
      "type": "text"
    },
    {
      "name": "attrMetadata.value",
      "type": "datetime"
    },
    {
      "name": "attrName",
      "type": "text"
    },
    {
      "name": "attrType",
      "type": "text"
    },
    {
      "name": "attrValue",
      "type": "text"
    },
    {
      "name": "entityId",
      "type": "text"
    },
    {
      "name": "entityType",
      "type": "text"
    },
    {
      "name": "recvTime",
      "type": "datetime"
    }
  ],
  "rows": [
    [
      "TimeInstant",
      "DateTime",
      "2021-04-16T09:23:38.418Z",
      "supportedProtocol",
      "Text",
      "[\"ul20\"]",
      "Motion:003",
      "Motion",
      "2021-04-16T09:23:38.418Z"
    ],
    [
      "TimeInstant",
      "DateTime",
      "2021-04-16T09:23:38.418Z",
      "function",
      "Text",
      "[\"sensing\"]",
      "Motion:003",
      "Motion",
      "2021-04-16T09:23:38.418Z"
    ]
  ]
}
```

# Next Steps

Want to learn how to add more complexity to your application by adding advanced features? You can find out by reading
the other [tutorials in this series](https://fiware-tutorials.rtfd.io)

---

## License

[MIT](LICENSE) © 2018-2021 FIWARE Foundation e.V.
