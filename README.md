# 🐳 Spark + Kafka Streaming Demo

Este repositorio contiene un entorno Docker Compose para probar **Spark Structured Streaming con Kafka**.  
Se incluye un notebook productor de datos de ejemplo y un notebook que consume mensajes en tiempo real.

---

## 🚀 Despliegue de contenedores

1. Clona el repositorio y entra en el directorio:
   ```bash
   git clone https://github.com/huamank/spark-kafka-streaming-demo.git
   cd spark-kafka-streaming-demo
   ```

2.	Levanta los servicios con Docker Compose:
    ```bash
    docker compose up -d --build
    ```

3.	Verifica que los contenedores estén corriendo:
    ```bash
    docker ps
    ```
    Deberías ver servicios como zookeeper, kafka, spark-master, spark-worker-1, etc.

## 🔎 Probar Kafka dentro del contenedor
1.	Ingresa al contenedor de Kafka:
    ```bash
    docker exec -it kafka bash
    ```

2.	Listar tópicos existentes:
    ```bash
    kafka-topics --bootstrap-server kafka:9092 --list
    ```

3.	Crear un nuevo tópico (ejemplo: orders):
    ```bash
    kafka-topics --bootstrap-server kafka:9092 \
    --create --topic orders --partitions 1 --replication-factor 1
    ```

4.	Describir el tópico:
    ```bash
    kafka-topics --bootstrap-server kafka:9092 --describe --topic orders
    ```

5.	Enviar datos (productor):
    ```bash
    kafka-console-producer --bootstrap-server kafka:9092 --topic orders
    ```

    Escribe algunas líneas (cada Enter = un mensaje).<br>
    Para salir ctrl+c

6.	Leer datos (consumidor):
    ```bash
    kafka-console-consumer --bootstrap-server kafka:9092 \
    --topic orders --from-beginning --max-messages 5
    ```
    El flag --from-beginning lee desde el inicio, y --max-messages limita la cantidad.

## Ejecutar los notebooks en Jupyter
Abre http://localhost:8200 y carga:

* python/producers/producer_orders.ipynb → envía eventos al tópico orders.
* python/consumers/consumer_orders.ipynb → lee del tópico orders con PySpark (Structured Streaming).

Importante: valor de kafka.bootstrap.servers
* Notebook dentro de contenedor (Jupyter en Docker): usa kafka:9092.
* Notebook en tu host (fuera de Docker): usa localhost:29092 (puerto publicado).

Ejemplo de lectura en el consumer:
    
```Python

from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DoubleType

spark = (SparkSession.builder
        .appName("orders-consumer")
        .config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.2")
        .getOrCreate())

schema = StructType([
    StructField("order_id", IntegerType()),
    StructField("customer_id", IntegerType()),
    StructField("amount", DoubleType()),
    StructField("status", StringType()),
    StructField("event_time", StringType()),
])

# Si Jupyter corre en contenedor: "kafka:9092"
# Si Jupyter corre en tu host:   "localhost:29092"
BOOTSTRAP = "kafka:9092"

raw = (spark.readStream.format("kafka")
    .option("kafka.bootstrap.servers", BOOTSTRAP)
    .option("subscribe", "orders")
    .option("startingOffsets", "latest")
    .load())

df = (raw.selectExpr("CAST(value AS STRING) AS v")
        .select(from_json(col("v"), schema).alias("data"))
        .select("data.*"))

query = (df.writeStream.format("console")
        .outputMode("append")
        .option("truncate", False)
        .start())

spark.streams.awaitAnyTermination()
```
Y un productor simple en el notebook de producers:

```Python
import json, time, random, os
from kafka import KafkaProducer

# Si Jupyter en contenedor: "kafka:9092"
# Si Jupyter en host:       "localhost:29092"
BOOTSTRAP = "kafka:9092"
TOPIC = "orders"

producer = KafkaProducer(
    bootstrap_servers=BOOTSTRAP,
    value_serializer=lambda v: json.dumps(v).encode("utf-8")
)

i = 1
while i <= 20:
    evt = {
        "order_id": i,
        "customer_id": random.randint(1000, 1010),
        "amount": round(random.uniform(10, 300), 2),
        "status": random.choice(["new","paid","shipped","cancelled"]),
        "event_time": time.strftime("%Y-%m-%dT%H:%M:%S")
    }
    producer.send(TOPIC, evt)
    i += 1
    time.sleep(0.5)

producer.flush()
print("Enviados 20 mensajes a", TOPIC)
```
