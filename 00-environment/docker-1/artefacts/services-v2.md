# nosql-platform-1 - List of Services

| Service | Links | External<br>Port | Internal<br>Port | Description
|--------------|------|------|------|------------
|[better-chatbot](./documentation/services/better-chatbot )|[Web UI](http://192.168.1.112:28405)|28405<br>|3000<br>|Modern AI chatbot with multi-LLM support
|[cassandra-1](./documentation/services/cassandra )||29042<br>7199<br>9160<br>|9042<br>7199<br>9160<br>|wide-column NoSQL database
|[dbgate](./documentation/services/dbgate )|[Web UI](http://192.168.1.112:28120)|28120<br>|3000<br>|Open Source DB Management Tool
|[elasticsearch-1](./documentation/services/elasticsearch )|[Rest API](http://192.168.1.112:9200)|9200<br>9300<br>|9200<br>9300<br>|Search-engine NoSQL store
|[elasticvue](./documentation/services/elasticvue )|[Web UI](http://192.168.1.112:28275)|28275<br>|8080<br>|UI for Elasticsearch
|[jupyter](./documentation/services/jupyter )|[Web UI](http://192.168.1.112:28888)|28888<br>|8888<br>|Web-based interactive development environment for notebooks, code, and data
|[kibana](./documentation/services/kibana )|[Web UI](http://192.168.1.112:5601)|5601<br>|5601<br>|Visualization for Elasticsearch
|[markdown-viewer](./documentation/services/markdown-viewer )|[Web UI](http://192.168.1.112:80)|80<br>|3000<br>|Platys Platform homepage viewer
|[mcp-inspector](./documentation/services/mcp-inspector )|[Web UI](http://192.168.1.112:6274)|6274<br>6277<br>|6274<br>6277<br>|MCP Inspector
|[mongo-1](./documentation/services/mongodb )||27017<br>|27017<br>|Document NoSQL database
|[mongo-express](./documentation/services/mongo-express )|[Web UI](http://192.168.1.112:28123)|28123<br>|8081<br>|MongoDB UI
|[mongo-mcp](./documentation/services/mongo-mcp )|[Rest API](http://192.168.1.112:28403/sse)|28403<br>|8000<br>|
|[open-webui](./documentation/services/open-webui )|[Web UI](http://192.168.1.112:28338)|28338<br>|8080<br>|User-friendly WebUI for LLMs
|[postgresql](./documentation/services/postgresql )||5432<br>|5432<br>|Open-Source object-relational database system
|[postgresql-mcp](./documentation/services/postgresql-mcp )|[Rest API](http://192.168.1.112:28404/sse)|28404<br>|8000<br>|
|[redis-1](./documentation/services/redis )||6379<br>|6379<br>|Key-value store
|[redis-commander](./documentation/services/redis-commander )|[Web UI](http://192.168.1.112:28119)|28119<br>|8081<br>|Graphical interface for Redis
|[redis-insight](./documentation/services/redis-insight )|[Web UI](http://192.168.1.112:28174)|28174<br>|5540<br>|Graphical interface for Redis
|[redis-mcp](./documentation/services/redis-mcp )|[Rest API](http://192.168.1.112:28225/sse)|28225<br>|8000<br>|
|[wetty](./documentation/services/wetty )|[Web UI](http://192.168.1.112:3001)|3001<br>|3000<br>|A terminal window in Web-Browser|

**Note:** init container ("init: true") are not shown