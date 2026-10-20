# nosql-platform-2 - List of Services

| Service | Web UI | Rest API 
|-------------- |------|------------
|[cockroachdb-1](./documentation/services/cockroachdb )|<http://${PUBLIC_IP}:28406>|
|[dbgate](./documentation/services/dbgate )|<http://192.168.1.112:28120>|
|[grafana](./documentation/services/grafana )|<http://192.168.1.112:3000>|<http://192.168.1.112:3000/api/org>
|[graphdb-1](./documentation/services/graphdb )|<http://192.168.1.112:17200>|
|[influxdb3](./documentation/services/influxdb3 )||<http://192.168.1.112:28263/api/v3>
|[influxdb3-explorer](./documentation/services/influxdb3-explorer )|<http://192.168.1.112:28264>|
|[jupyter](./documentation/services/jupyter )|<http://192.168.1.112:28888>|
|[markdown-viewer](./documentation/services/markdown-viewer )|<http://192.168.1.112:80>|
|[mcp-inspector](./documentation/services/mcp-inspector )|<http://192.168.1.112:6274>|
|[mosquitto-1](./documentation/services/mosquitto )||
|[mqtt-ui](./documentation/services/hivemq-ui )|<http://192.168.1.112:28136>|
|[mqttx-cli](./documentation/services/mqttx )||
|[neo4j-1](./documentation/services/neo4j )|<http://192.168.1.112:7474>|
|[neo4j-mcp](./documentation/services/neo4j-mcp )||<http://192.168.1.112:28271>
|[neodash](./documentation/services/neodash )|<http://192.168.1.112:5005>|
|[qdrant](./documentation/services/qdrant )|<http://192.168.1.112:6333/dashboard>|<http://192.168.1.112:6333>
|[telegraf](./documentation/services/telegraf )||
|[wetty](./documentation/services/wetty )|<http://192.168.1.112:3001>||

**Note:** init container ("init: true") are not shown