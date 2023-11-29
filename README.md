# Cluster de 3 nodos con Apache Cassandra

Tutorial adaptado de [esta dirección](https://developer.ibm.com/tutorials/ba-set-up-apache-cassandra-architecture/) para correr sin errores usando Docker y **Windows CMD**.

## Correr contenedor raíz
```cmd
docker run --name cassandra-node-1 cassandra:3.11
```

## Encontrar dirección IP del contenedor y guardar en variable SEED_NODE_ADDR
```cmd
for /f "tokens=* usebackq" %F in (`docker inspect -f {{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}} cassandra-node-1`) do (set SEED_NODE_ADDR=%F)
```

## Correr contenedores adicionales
```cmd
docker run --name cassandra-node-2 -e CASSANDRA_SEEDS=%SEED_NODE_ADDR% cassandra:3.11
```
```cmd
docker run --name cassandra-node-3 -e CASSANDRA_SEEDS=%SEED_NODE_ADDR% cassandra:3.11
```

## Probar correcto número de nodos corriendo en cluster desde contenedor raíz
```cmd
docker exec cassandra-node-1 nodetool status
```

## Probar corriendo línea de comandos `cqlsh`
```cmd
docker exec -it cassandra-node-1 cqlsh
```

## Crear el espacio de llaves de prueba
Desde una terminal corriendo `cqlsh` en el contenedor raíz:

1. Crear esquema `patient`:
    ```cmd
    CREATE KEYSPACE patient WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};
    ```

2. Crear tabla `exam` para datos de exámenes de pacientes:
    ```cmd
    CREATE TABLE patient.exam (patient_id int, id int, date timeuuid, details text, PRIMARY KEY (patient_id, id));
    ```
    La tabla creada tiene como llave primaria los atributos de ID del paciente y el ID del examen en sí.

## Insertar datos de prueba
Desde una terminal corriendo `cqlsh` en el contenedor raíz:

1. Insertar:
    ```cmd
    USE patient;
    INSERT INTO exam (patient_id,id,date,details) values (1,1,now(),'first exam patient 1');
    INSERT INTO exam (patient_id,id,date,details) values (1,2,now(),'second exam patient 1');
    INSERT INTO exam (patient_id,id,date,details) values (2,1,now(),'first exam patient 2');
    INSERT INTO exam (patient_id,id,date,details) values (3,1,now(),'first exam patient 3'); 
    ```

2. Correr consulta de prueba que devuelve los exámenes del paciente 1:
    ```cmd
    SELECT * FROM exam WHERE patient_id=1;
    ```

## Probar replicación
Con factor de replicación = 3 los datos insertados en el nodo 1 deberían estar presentes en los demás nodos. Para probarlo puede seguir estos pasos.

1. Correr `cqlsh` en contenedor de nodo 3:
    ```cmd
    docker exec -it cassandra-node-3 cqlsh
    ```

2. Correr consulta de prueba que devuelve los exámenes del paciente 1:
    ```cmd
    USE patient;
    SELECT * FROM exam WHERE patient_id=1;
    ```

## Probar fallo de nodos
1. Detener nodos 2 y 3 para simular un fallo de estos nodos:
    ```cmd
        docker stop cassandra-node-2
    ```
    ```cmd
        docker stop cassandra-node-3
    ```

2. Correr `cqlsh` en contenedor de nodo 1:
    ```cmd
    docker exec -it cassandra-node-1 cqlsh
    ```

3. Insertar un nuevo examen:
    ```cmd
    USE patient;
    INSERT INTO exam (patient_id,id,date,details) values (10,1,now(),'first exam patient 10');
    ```

4. Volver a correr los nodos 2 y 3:
    ```cmd
    docker start cassandra-node-2
    ```
    ```cmd
    docker start cassandra-node-3
    ```
    Luego esperar un minuto para que se inicien los procesos.

5. Correr `cqlsh` en contenedor de nodo 3:
    ```cmd
    docker exec -it cassandra-node-3 cqlsh
    ```

6. Correr consulta de prueba que devuelve los exámenes del nuevo paciente:
    ```cmd
    USE patient;
    SELECT * FROM exam WHERE patient_id=10;
    ```
    Si se muestra el examen recién ingresado, significa que la replicación continúa funcionando incluso después de un fallo de nodos.

## Probar consistencia de nodos




