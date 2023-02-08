# Debezium - Change Data Capture
Change Data Capture con Debezium, Postgres y Kafka. 

## Componentes
En este proyecto se usan diferentes conectores de Debezium para sincronizar datos entre dos Bases de Datos SQL (Postgres) usando el patrón Change Data Capture. Los componentes usados se explican a continuación.
![](/resource/Debezium-CDC.png)
### Origin DB
Base de Datos Postgres de origen.
### Debezium - Postgres Connector
Debezium que usa Postgres Connector para estar a la escucha de los cambios en la base de datos de origen.
### Kafka
Broker Kafka donde se registran los cambios realizados en la BD de origen.
### Debezium - JDBC Sink
Debezium con JDBC Sink Connector que se encarga de registrar y actualizar los datos en la BD de destino.
### Target DB
Base de Datos Postgres de destino.
## Pasos para probar
Iniciar los contenedores en docker
```bash
docker compose up
```
