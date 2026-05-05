# nosql-platform-2 - List of Services

| Service | Web UI | Rest API 
|-------------- |------|------------
|[cockroachdb-1](./documentation/services/cockroachdb )|<http://${PUBLIC_IP}:28406>|
|[cockroachdb-2](./documentation/services/cockroachdb )|<http://${PUBLIC_IP}:28407>|
|[dbgate](./documentation/services/dbgate )|<http://127.0.0.1:28120>|
|[grafana](./documentation/services/grafana )|<http://127.0.0.1:3000>|<http://127.0.0.1:3000/api/org>
|[graphdb-1](./documentation/services/graphdb )|<http://127.0.0.1:17200>|
|[influxdb3](./documentation/services/influxdb3 )||<http://127.0.0.1:28263/api/v3>
|[influxdb3-explorer](./documentation/services/influxdb3-explorer )|<http://127.0.0.1:28264>|
|[jupyter](./documentation/services/jupyter )|<http://127.0.0.1:28888>|
|[markdown-viewer](./documentation/services/markdown-viewer )|<http://127.0.0.1:80>|
|[mcp-inspector](./documentation/services/mcp-inspector )|<http://127.0.0.1:6274>|
|[mosquitto-1](./documentation/services/mosquitto )||
|[mqtt-ui](./documentation/services/hivemq-ui )|<http://127.0.0.1:28136>|
|[mqttx-cli](./documentation/services/mqttx )||
|[neo4j-1](./documentation/services/neo4j )|<http://127.0.0.1:7474>|
|[neo4j-mcp](./documentation/services/neo4j-mcp )||<http://127.0.0.1:28271>
|[neodash](./documentation/services/neodash )|<http://127.0.0.1:5005>|
|[qdrant](./documentation/services/qdrant )|<http://127.0.0.1:6333/dashboard>|<http://127.0.0.1:6333>
|[telegraf](./documentation/services/telegraf )||
|[wetty](./documentation/services/wetty )|<http://127.0.0.1:3001>||

**Note:** init container ("init: true") are not shown