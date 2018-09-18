# Scaling a Node.js server based webapp

# 1. - Take advantage of machine cores
  - Node.js instances run in a single thread. Node.js can launch a cluster of processes (fork the original process) to handle the load and use each core of a multicore processor (round robin or creating a socket and workers accepting incoming connections).
  - Use PM2, the process manager. Maybe add a message queue (RabbitMQ, Java based; Kafka, Scala based) to feed the workers.

  
# 2. - Proxy server
  - (scaling out): proxy server (nginx, haproxy) in front of different Node servers. It is recommended HAProxy as load balancer (3 techniques: round-robin, IP hash, least connections).

# 3. - Database
  - vertical (scaling up): adding resources, more memory, more CPUs, same server
  - horizontal (scaling out): more servers, sharding (distribute data across multiple machines)

# 4. - In-memory storage
  - memory-cache: caching content on server side but not shared between multiple node.js process
  - redis: distributed cache service

# 5. - Caching static content (CDN): for example React.js apps
