## Hi there 👋  
Crafting a next-gen **multi-vendor e-commerce platform** with a scalable **microservice architecture**! 🚀  
Built with **Spring Boot** & **Spring Cloud**, this system enables seamless vendor management, ensuring **high availability** and **fault tolerance**. 🏗️  

🧩 **Event-driven architecture** using **Kafka** allows real-time communication between services, while **distributed locks** ensure data consistency in concurrent transactions.  

⚙️ **CockroachDB** powers the **distributed SQL database layer**, offering horizontal scalability and strong consistency across global regions. Shard-aware key design and geo-partitioning enable optimized performance and data locality. 🌍  

🔍 **Elasticsearch** enhances the search experience with full-text indexing, enabling lightning-fast product and vendor discovery. Supports autocomplete, filtering, and real-time analytics.  

🛡️ Designed with robust design patterns (**Saga**, **CQRS**, **Circuit Breaker**) to enhance modularity, fault isolation, and maintainability.  

⚡ Powered by **WebFlux** & **Project Reactor** for reactive, non-blocking performance — perfect for **ultra-fast order processing** and backpressure-aware streams.  

🌐 **Distributed user tracking** captures **IP address** and **geolocation** (region, city, country) for personalized recommendations, fraud detection, and region-specific optimization.  

A future-proof solution for **high-demand e-commerce ecosystems**! 🛒🔥  


<a href="https://dbdiagram.io/d/csdlpt-67f93b174f7afba18445cf3b">Diagram</a>


How to install ?

1. Clone code
2. build docker

version: '3.8'
services:
  # CockroachDB Node 1
  cockroachdb-1:
    image: cockroachdb/cockroach:latest
    container_name: cockroachdb-1
    command: start --insecure --join=cockroachdb-1,cockroachdb-2,cockroachdb-3
    ports:
      - "26257:26257"
      - "8081:8080"
    volumes:
      - cockroach-data-1:/cockroach/cockroach-data
    networks:
      - app-net

  # CockroachDB Node 2
  cockroachdb-2:
    image: cockroachdb/cockroach:latest
    container_name: cockroachdb-2
    command: start --insecure --join=cockroachdb-1,cockroachdb-2,cockroachdb-3
    ports:
      - "26258:26257"
      - "8082:8080"
    volumes:
      - cockroach-data-2:/cockroach/cockroach-data
    networks:
      - app-net

  # CockroachDB Node 3
  cockroachdb-3:
    image: cockroachdb/cockroach:latest
    container_name: cockroachdb-3
    command: start --insecure --join=cockroachdb-1,cockroachdb-2,cockroachdb-3
    ports:
      - "26259:26257"
      - "8083:8080"
    volumes:
      - cockroach-data-3:/cockroach/cockroach-data
    networks:
      - app-net

  # CockroachDB Cluster Initializer
  cockroachdb-init:
    image: cockroachdb/cockroach:latest
    container_name: cockroachdb-init
    command: init --insecure --host=cockroachdb-1
    depends_on:
      - cockroachdb-1
      - cockroachdb-2
      - cockroachdb-3
    networks:
      - app-net

  # CockroachDB SQL Runner
  cockroachdb-sql:
    image: cockroachdb/cockroach:latest
    container_name: cockroachdb-sql
    entrypoint: /bin/sh
    command: -c "sleep 5 && /cockroach/cockroach sql --insecure --host=cockroachdb-1 -f /init.sql"
    depends_on:
      - cockroachdb-init
    volumes:
      - ./init.sql:/init.sql
    networks:
      - app-net

  # Eureka Service Registry
  eureka-service:
    build:
      context: ./discovery
      dockerfile: Dockerfile
    container_name: discovery-service
    ports:
      - "8761:8761"
    networks:
      - app-net
    depends_on:
      - cockroachdb-1
      - cockroachdb-2
      - cockroachdb-3
    environment:
      - S3_ENDPOINT=http://minio:9000 # Point to local MinIO
      - AWS_ACCESS_KEY_ID=minioadmin # Default MinIO credentials
      - AWS_SECRET_ACCESS_KEY=minioadmin
      - AWS_REGION=us-east-1

  # MinIO (Local S3-compatible Storage)
  minio:
    image: minio/minio:latest
    container_name: minio
    ports:
      - "9000:9000" # S3 API port
      - "9001:9001" # Console port
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio-data:/data
    networks:
      - app-net

  # RabbitMQ Message Broker - Primary Instance
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq-primary
    ports:
      - "5672:5672" # AMQP port
      - "15672:15672" # Management UI port
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    networks:
      - app-net

  # Elasticsearch
  elasticsearch:
    image: elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200" # HTTP port
      - "9300:9300" # Transport port
    volumes:
      - es-data:/usr/share/elasticsearch/data
    networks:
      - elk-net

  # Kibana
  kibana:
    image: kibana:8.12.0
    container_name: kibana
    ports:
      - "5601:5601" # Kibana UI port
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - elk-net

  # Redis In-Memory Cache
  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379" # Redis port
    volumes:
      - redis-data:/data
    command: redis-server --requirepass mypassword
    networks:
      - app-net
  # Web UI for CockroachDB (pgweb)
  cockroachdb-ui:
    image: sosedoff/pgweb
    container_name: cockroachdb-ui
    ports:
      - "8085:8081" # Web UI will be available at http://localhost:8085
    networks:
      - app-net
    depends_on:
      - cockroachdb-1
    environment:
      - DATABASE_URL=postgres://root@cockroachdb-1:26257/defaultdb?sslmode=disable
    entrypoint: sh -c "sleep 10 && pgweb --bind=0.0.0.0 --listen=8081 --url=postgres://root@cockroachdb-1:26257/defaultdb?sslmode=disable"

  # Redis Insight - Web UI for Redis
  redis-insight:
    image: redislabs/redisinsight:latest
    container_name: redis-insight
    ports:
      - "8002:8001" # Web UI will be available at http://localhost:8002
    volumes:
      - redisinsight:/db
    networks:
      - app-net
    restart: always
    depends_on:
      - redis

networks:
  app-net:
    driver: bridge
  elk-net:
    driver: bridge

volumes:
  cockroach-data-1:
  cockroach-data-2:
  cockroach-data-3:
  rabbitmq-data:
  redis-data:
  es-data:
  minio-data:
  redisinsight:

3. Run each service
4. Post man <a href="[https://dbdiagram.io/d/csdlpt-67f93b174f7afba18445cf3b](https://interstellar-zodiac-219622.postman.co/workspace/My-Workspace~163666f7-e5d3-41f3-84fe-d0342eba4356/collection/31743421-52a8b138-ef98-455d-8c73-a08011dc0a14?action=share&creator=31743421)">Postman</a>
