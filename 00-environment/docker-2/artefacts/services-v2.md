# nosql-platform - List of Services

| Service | Links | External<br>Port | Internal<br>Port | Description
|--------------|------|------|------|------------
|[cassandra-1](./documentation/services/cassandra )||29042<br>7199<br>9160<br>|9042<br>7199<br>9160<br>|wide-column NoSQL database
|[dbgate](./documentation/services/dbgate )|[Web UI](http://127.0.0.1:28120)|28120<br>|3000<br>|Open Source DB Management Tool
|[elasticsearch-1](./documentation/services/elasticsearch )|[Rest API](http://127.0.0.1:9200)|9200<br>9300<br>|9200<br>9300<br>|Search-engine NoSQL store
|[elasticvue](./documentation/services/elasticvue )|[Web UI](http://127.0.0.1:28275)|28275<br>|8080<br>|UI for Elasticsearch
|[graphdb-1](./documentation/services/graphdb )|[Web UI](http://127.0.0.1:17200)|17200<br>|7200<br>|Triple/RDF Store
|[influxdb3](./documentation/services/influxdb3 )|[Rest API](http://127.0.0.1:28263/api/v3)|28263<br>|8181<br>|Timeseries Database
|[influxdb3-explorer](./documentation/services/influxdb3-explorer )|[Web UI](http://127.0.0.1:28264)|28264<br>|80<br>|Timeseries Database UI
|[jupyter](./documentation/services/jupyter )|[Web UI](http://127.0.0.1:28888)|28888<br>|8888<br>|Web-based interactive development environment for notebooks, code, and data
|[kibana](./documentation/services/kibana )|[Web UI](http://127.0.0.1:5601)|5601<br>|5601<br>|Visualization for Elasticsearch
|[markdown-viewer](./documentation/services/markdown-viewer )|[Web UI](http://127.0.0.1:80)|80<br>|3000<br>|Platys Platform homepage viewer
|[mcp-inspector](./documentation/services/mcp-inspector )|[Web UI](http://127.0.0.1:6274)|6274<br>6277<br>|6274<br>6277<br>|MCP Inspector
|[mongo-1](./documentation/services/mongodb )||27017<br>|27017<br>|Document NoSQL database
|[mongo-express](./documentation/services/mongo-express )|[Web UI](http://127.0.0.1:28123)|28123<br>|8081<br>|MongoDB UI
|[mongo-mcp](./documentation/services/mongo-mcp )|[Rest API](http://127.0.0.1:28403/sse)|28403<br>|8000<br>|
|[mosquitto-1](./documentation/services/mosquitto )||1883<br>9101<br>|1883<br>9001<br>|MQTT Broker
|[mqtt-ui](./documentation/services/hivemq-ui )|[Web UI](http://127.0.0.1:28136)|28136<br>|80<br>|MQTT UI
|[mqttx-cli](./documentation/services/mqttx )||||a CLI for MQTT
|[neo4j-1](./documentation/services/neo4j )|[Web UI](http://127.0.0.1:7474)|7474<br>7687<br>|7474<br>7687<br>|Native graph database management system
|[neo4j-mcp-cypher](./documentation/services/neo4j-mcp )|[Rest API](http://127.0.0.1:28271)|28271<br>|8000<br>|MCP Server for Neo4J (Cypher)
|[neo4j-mcp-memory](./documentation/services/neo4j-mcp )|[Rest API](http://127.0.0.1:28272)|28272<br>|8000<br>|MCP Server for Neo4J (Memory)
|[neodash](./documentation/services/neodash )|[Web UI](http://127.0.0.1:5005)|5005<br>|5005<br>|Visualize you Neo4J data
|[postgresql](./documentation/services/postgresql )||5432<br>|5432<br>|Open-Source object-relational database system
|[qdrant](./documentation/services/qdrant )|[Web UI](http://127.0.0.1:6333/dashboard) - [Rest API](http://127.0.0.1:6333)|6333<br>6334<br>|6333<br>6334<br>|AI-native vector database
|[redis-1](./documentation/services/redis )||6379<br>|6379<br>|Key-value store
|[redis-commander](./documentation/services/redis-commander )|[Web UI](http://127.0.0.1:28119)|28119<br>|8081<br>|Graphical interface for Redis
|[redis-insight](./documentation/services/redis-insight )|[Web UI](http://127.0.0.1:28174)|28174<br>|5540<br>|Graphical interface for Redis
|[redis-mcp](./documentation/services/redis-mcp )|[Rest API](http://127.0.0.1:28225/sse)|28225<br>|8000<br>|
|[telegraf](./documentation/services/telegraf )||||Agent for collecting, processing, aggregating, and writing metrics
|[wetty](./documentation/services/wetty )|[Web UI](http://127.0.0.1:3001)|3001<br>|3000<br>|A terminal window in Web-Browser|

**Note:** init container ("init: true") are not shown