# Debezium - Change Data Capture
Change Data Capture con Debezium, Postgres y Kafka usando Docker. 

## Componentes
En este proyecto se usan diferentes conectores de Debezium para sincronizar datos entre dos Bases de Datos SQL (Postgres) usando el patrón Change Data Capture. Los componentes usados se explican a continuación.
![](/resource/Debezium-CDC.png)
* ### Origin DB
Base de Datos Postgres de origen.
* ### Debezium - Postgres Connector
Debezium que usa Postgres Connector para estar a la escucha de los cambios en la base de datos de origen.
* ### Kafka
Broker Kafka donde se registran los cambios realizados en la BD de origen.
* ### Debezium - JDBC Sink
Debezium con JDBC Sink Connector que se encarga de registrar y actualizar los datos en la BD de destino.
* ### Target DB
Base de Datos Postgres de destino.
## Configuración del ambiente
Nos situamos en la raíz del proyecto y seguimos los siguientes pasos.

Iniciar los contenedores en docker
```bash
docker compose up -d
```
Instalación de JDBC Sink en debezium.
```bash
docker exec -it debezium-jdbc-sink sh -c \"cd /kafka/libs && curl -sO https://jdbc.postgresql.org/download/postgresql-42.4.1.jar"
```
```bash
docker exec -it debezium-jdbc-sink sh -c \"mkdir /kafka/connect/kafka-connect-jdbc"
```
```bash
docker exec -it debezium-jdbc-sink sh -c \"cd /kafka/connect/kafka-connect-jdbc && curl -sO https://packages.confluent.io/maven/io/confluent/kafka-connect-jdbc/5.5.3/kafka-connect-jdbc-5.5.3.jar"
```
```bash
docker restart debezium-jdbc-sink
```
Creación de una tabla el la Base de Datos origen.
```bash
docker exec -it postgres psql -U postgres -c "CREATE TABLE students(id int primary key, name varchar(30), description varchar(30));"
```
Creación de Base de Datos de destino.
```bash
docker exec -it postgres psql -U postgres -c "CREATE DATABASE targetpostgres;"
```
Creación de conector Postgres en Debezium. En este caso es el Debezium que capturará los cambios en la Base de datos de origen, puerto de Debezium: 8083. Especificamos el nombre de la tabla a la que nos conectaremos *public.students* y las columnas que informaremos a kafka *id* y *description*
```bash
curl --location --request POST 'localhost:8083/connectors/' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "connector-postgres",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "plugin.name": "pgoutput",
        "database.hostname": "postgres",
        "database.port": "5432",
        "database.user": "postgres",
        "database.password": "postgres",
        "database.dbname": "postgres",
        "database.server.name": "postgres",
        "table.include.list": "public.students",
        "column.include.list": "public.students.id,public.students.description"
    }
}'
```
Creación de conector JDBC Sink en Debezium. En este caso es el Debezium que se encargará de actualizar los datos en la Base de Datos de destino. Puerto de Debezium: 28083. En este caso la tabla donde se mapearán los datos se llama *public.student*.
```bash
curl --location --request POST 'localhost:28083/connectors/' \
--header 'Content-Type: application/json' \
--data-raw '{
   "name":"jdbc-sink",
   "config":{
      "connector.class":"io.confluent.connect.jdbc.JdbcSinkConnector",
      "connection.url":"jdbc:postgresql://postgres:5432/targetpostgres",
      "tasks.max":"1",
      "topics":"postgres.public.students",
      "connection.user":"postgres",
      "connection.password":"postgres",
      "auto.create":true,
      "auto.evolve":true,
      "insert.mode":"upsert",
      "table.name.format":"targetpostgres.public.student",
      "pk.mode":"record_key",
      "pk.fields":"id",
      "transforms":"unwrap",
      "transforms.unwrap.type":"io.debezium.transforms.ExtractNewRecordState"
   }
}'
```
## Prueba
Insertamos un nuevo registro en la Base de Datos de origen.
```bash
docker exec -it postgres psql -U postgres -c "INSERT INTO public.students VALUES(random() * 100, 'asd', 'description');"
```
Verificamos la inserción en la Base de Datos de origen.
```bash
docker exec -it postgres psql -U postgres -c "SELECT * FROM  postgres.public.students;"
```
Deberíamos observar lo siguiente:
```bash
 id | name | description
----+------+-------------
 72 | asd  | description
(1 row)
```
El registro también debería mostrarse en la Base de Datos de destino, en este caso como en el conector Postgres en Debezium especificamos que solo informe las columnas "id" y "description" son las únicas que deberían estar en la BD destino.
```bash
docker exec -it postgres psql -U postgres -d targetpostgres -c "SELECT * FROM public.student;"
```
Deberíamos observar lo siguiente:
```bash
 id | description
----+-------------
 72 | description
(1 row)
```
## Otros recursos
En el proyecto también se incluyen:
* **kafka-ui**: Es una interfaz gráfica que permite ver los tópicos de kafka. URL: http://localhost:9000
* **dozzle**: Permite ver los logs de los contenedores que levantamos en docker. URL: http://localhost:9090