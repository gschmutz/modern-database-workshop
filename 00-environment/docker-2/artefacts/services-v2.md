# nosql-platform-2 - List of Services

| Service | Links | External<br>Port | Internal<br>Port | Description
|--------------|------|------|------|------------
|[cockroachdb-1](./documentation/services/cockroachdb )|[Web UI](http://${PUBLIC_IP}:28406)|26257<br>28406<br>|26257<br>8080<br>|Cloud-native distributed SQL database
|[dbgate](./documentation/services/dbgate )|[Web UI](http://192.168.1.112:28120)|28120<br>|3000<br>|Open Source DB Management Tool
|[grafana](./documentation/services/grafana )|[Web UI](http://192.168.1.112:3000) - [Rest API](http://192.168.1.112:3000/api/org)|3000<br>|3000<br>|Data visualization and dashboarding
|[graphdb-1](./documentation/services/graphdb )|[Web UI](http://192.168.1.112:17200)|17200<br>|7200<br>|Triple/RDF Store
|[influxdb3](./documentation/services/influxdb3 )|[Rest API](http://192.168.1.112:28263/api/v3)|28263<br>|8181<br>|Timeseries Database
|[influxdb3-explorer](./documentation/services/influxdb3-explorer )|[Web UI](http://192.168.1.112:28264)|28264<br>|8080<br>|Timeseries Database UI
|[jupyter](./documentation/services/jupyter )|[Web UI](http://192.168.1.112:28888)|28888<br>|8888<br>|Web-based interactive development environment for notebooks, code, and data
|[markdown-viewer](./documentation/services/markdown-viewer )|[Web UI](http://192.168.1.112:80)|80<br>|3000<br>|Platys Platform homepage viewer
|[mcp-inspector](./documentation/services/mcp-inspector )|[Web UI](http://192.168.1.112:6274)|6274<br>6277<br>|6274<br>6277<br>|MCP Inspector
|[mosquitto-1](./documentation/services/mosquitto )||1883<br>9101<br>|1883<br>9001<br>|MQTT Broker
|[mqtt-ui](./documentation/services/hivemq-ui )|[Web UI](http://192.168.1.112:28136)|28136<br>|80<br>|MQTT UI
|[mqttx-cli](./documentation/services/mqttx )||||a CLI for MQTT
|[neo4j-1](./documentation/services/neo4j )|[Web UI](http://192.168.1.112:7474)|7474<br>7687<br>|7474<br>7687<br>|Native graph database management system
|[neo4j-mcp](./documentation/services/neo4j-mcp )|[Rest API](http://192.168.1.112:28271)|28271<br>|8000<br>|MCP Server for Neo4J
|[neodash](./documentation/services/neodash )|[Web UI](http://192.168.1.112:5005)|5005<br>|5005<br>|Visualize you Neo4J data
|[qdrant](./documentation/services/qdrant )|[Web UI](http://192.168.1.112:6333/dashboard) - [Rest API](http://192.168.1.112:6333)|6333<br>6334<br>|6333<br>6334<br>|AI-native vector database
|[telegraf](./documentation/services/telegraf )||||Agent for collecting, processing, aggregating, and writing metrics
|[wetty](./documentation/services/wetty )|[Web UI](http://192.168.1.112:3001)|3001<br>|3000<br>|A terminal window in Web-Browser|

**Note:** init container ("init: true") are not shown