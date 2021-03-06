# Configuracion de replicacion stream DB para Postgresql en Ansible Tower.

> **Nota:** Procedimiento realizado sobre postgres 10 para Ansible Tower 3.8.

## Ambiente de laboratorio.
Se cuenta 5 servidores de un cluster de ansible tower v3.8, de los cuales 3 son Workers y 2 Bases de datos.

La disposicion es la siguiente:

1. tower1.example.com ```(IP=172.160.10.61)```
2. tower2.example.com ```(IP=172.160.10.62)```
3. tower3.example.com ```(IP=172.160.10.63)```
4. db1.example.com  ```(IP=172.160.10.65) ( maestro )```
5. db2.example.com  ```(IP=172.160.10.66) ( esclavo )```

## Configuracion sobre el Maestro

En este paso, configuraremos un servidor maestro para la replicación. Este es el servidor principal, que permite el proceso de lectura y escritura desde aplicaciones que se ejecutan en él. PostgreSQL en el maestro se ejecuta solo en la dirección IP ``` 172.16.10.65``` y realiza la replicación de transmisión al servidor esclavo.

### Crear un rol dedicado a la replicacion
Sobre la consola del maestro ejecutar el siguiente procedimiento para crear un rol de replicacion.

* Ingresar con el user postgres
* Activar el ambiente
* Ingresar a la consola de postgresql
* Crear el rol
* Definir el tipo de encripcion del password
* Definir un password para el rol

```
[root@db1 ~]# su - postgres
Last login: Wed Jan 13 22:27:31 -05 2021 on pts/0
-bash-4.2$
-bash-4.2$ source /opt/rh/rh-postgresql10/enable
-bash-4.2$
-bash-4.2$
-bash-4.2$ psql
psql (10.15)
Type "help" for help.

postgres=#
postgres=#
postgres=# CREATE ROLE replicate WITH REPLICATION LOGIN ; 
CREATE ROLE
postgres=# set password_encryption = 'scram-sha-256';
SET
postgres=# \password replicate 
Enter new password:   *******
Enter it again:       *******
postgres=# \q

```
al finalizar ejecutamos \q para salir de la consola de postgres

### Activar la interface de red
Se procede a activar las interfaces de red por donde se expone el servicio de base de datos, para esto sobre la consola bash del user postgres se edita el fichero ```/var/opt/rh/rh-postgresql10/lib/pgsql/data/postgresql.conf```, ubicando la linea ```listen_adresses```

```
Antes
#listen_addresses = 'localhost'

Despues

Listen_addresses = ‘localhost, 172.16.10.65’
```

### Parametros de transmision de la replicacion
Ajustar el fichero ```/var/opt/rh/rh-postgresql10/lib/pgsql/data/postgresql.conf``` con lo siguiente:

```
wal_level = logical             # minimal, archive, hot_standby, or logical

max_wal_senders = 2             # max number of walsender processes
```

```max_wal_senders``` define el numero de nodos que intervienen en la replicacion


tambien es necesario agregar lo siguiente:
donde la direccion ip pertenece al nodo esclavo.

```
archive_mode = on
archive_command = 'rsync -a %p postgres@172.16.10.66:/var/log/rh/rh0postgresql10/lib/pgsql/archive/%f'
```
El ```archive_command ``` copiará los segmentos wal en un directorio que debe ser accesible por el servidor de reserva. En el ejemplo anterior, uso el comando rsyn  para copiarlos directamente en el modo de espera.

### Permitir la conexion a la base de datos desde el server esclavo
Ahora permita que sus esclavos se conecten al maestro. Edite el fichero ```/var/opt/rh/rh-postgresql10/lib/pgsql/data/pg_hba.conf``` y agregue algo como esto al final de la reglas:

```
# !!! This file managed by Ansible. Any local changes may be overwritten. !!!

# Database administrative login by UNIX sockets
# note: you may wish to restrict this further later
local   all         postgres                                peer

# TYPE  DATABASE    USER        CIDR-ADDRESS          METHOD
local   all             all                              scram-sha-256
host    all             all         0.0.0.0/0            scram-sha-256
host    all             all         ::/0                 scram-sha-256
host   replication     replicate   172.16.10.66/32       scram-sha-256 <<<
```

### Reinicio de servicio postgres sobre el maestro
Una vez realizado los ajustes anteriores se procede con un reinicio de servicios de postgres para garantizar que todo se encuentra en correcto estado.

```
[root@db1 ~]# systemctl restart rh-postgresql10-postgresql.service
```
el servicio debe reinicior sin inconvenientes, de generarse algun error rectifique las modificaciones realizadas anteriormente.

## Configuraciones sobre el Esclavo
A continuacion se muestran las configuraciones realizadas sobre el servidor esclavo donde se replicara la base de datos.

### Detener los servicios de postgres
Para iniciar las configuraciones es necesario detener lo servicios de postgres para esto ejecutar lo siguiente:

```
[root@db2 ~]# systemctl stop rh-postgresql10-postgresql.service
```
### Generar un respaldo de la data actual
Cambiese al directorio de su PGDATA, con usuario postgres
``` 
[root@db2 ~]# su - postgres
bash-4.2$ cd /var/opt/rh/rh-postgresql10/lib/pgsql
bash-4.2$ mv data bck-data
```

### Generar un respaldo de los datos desde el maestro
Ahora copiaremos todos los datos del maestro con el comando ```pg_basebackup```

```
[root@db2 ~]# su - postgres
bash-4.2$ source /opt/rh/rh-postgresql10/enable
bash-4.2$ pg_basebackup -h 172.16.10.65 -D /var/opt/rh/rh-postgresql10/lib/pgsql/data -P -U replicate --wal-method=stream
Password:  *******
23908/23908 kB (100%), 1/1 tablespace
```
> **Nota:** El password definido se utilizara mas edelante en el proceso de configuracion del servidor esclavo

### Ajustar el fichero postgres.conf
Ahora con el user postgres se edita el fichero ```/var/opt/rh/rh-postgresql10/lib/pgsql/data/postgresql.conf``` ajustando los siguientes parametros:

```
... omitida
listen_addresses = 'localhost,172.16.10.66'

... omitida
hot_standby = on


... omitida
#archive_mode = on
#archive_command = 'rsync -a %p postgres@172.16.10.66:/var/log/rh/rh0postgresql10/lib/pgsql/arch
ive/%f'
```

### Crear un fichero nuevo recovery.conf
Es necesario crear el fichero ```/var/opt/rh/rh-postgresql10/lib/pgsql/data/recovery.conf```, con el siguiente contenido.

```
standby_mode          = 'on'
primary_conninfo      = 'host=172.16.10.65 port=5432 user=replicate password=XXXXXXXX'
trigger_file = '/tmp/MasterNow'
```
donde el **XXXXX** es el password del rol de replicacion definido con anterioridad.

Finalmente es necesario iniciar el servicio de postgresql, el servicio debe subir sin problemas

```
[root@2 ~]# systemctl start rh-postgresql10-postgresql
```

## Validacion del correcto funcionamiento
## Queries sobre la base de datos en maestro

1. Validar que la sincronizacion este operativa
```
[root@db1 ~]# su - postgres
-bash-4.2$ source /opt/rh/rh-postgresql10/enable
-bash-4.2$
-bash-4.2$ psql -c "select application_name, state, sync_priority, sync_state from pg_stat_replication;"
 application_name |   state   | sync_priority | sync_state
------------------+-----------+---------------+------------
 walreceiver      | streaming |             0 | async
(1 row)

-bash-4.2$
-bash-4.2$ psql -x -c "select * from pg_stat_replication;"
-[ RECORD 1 ]----+------------------------------
pid              | 116205
usesysid         | 23055
usename          | replicate
application_name | walreceiver
client_addr      | 172.16.10.66
client_hostname  |
client_port      | 41568
backend_start    | 2021-02-01 22:52:09.444968+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/8305B28
write_lsn        | 0/8305B28
flush_lsn        | 0/8305B28
replay_lsn       | 0/8305B28
write_lag        | 00:00:00.000219
flush_lag        | 00:00:00.000823
replay_lag       | 00:00:00.000934
sync_priority    | 0
sync_state       | async

```
2. Verificacion del funcionamiento de la replica, para esto se crea una tabla **replica_test** sobre el maestro y se confirmar sobre el esclavo
2.1 Revisar que la tabla no exista en ninguno de los nodos

* Maestro
```
postgres=# select * from replica_test;
ERROR:  relation "replica_test" does not exist
LINE 1: select * from replica_test;
```

* Esclavo
```
postgres=# select * from replica_test;
ERROR:  relation "replica_test" does not exist
LINE 1: select * from replica_test;
```
2.2 Crear la tabla sobre el Maestro
```
postgres=# CREATE TABLE replica_test (test varchar(100));
CREATE TABLE
postgres=#
```

2.3 Validar sobre el esclavo que la tabla este presente
```
postgres=# select * from replica_test;
 test
------
(0 rows)
```
## Test de escritura
La base de datos Maestra debe permitir crear o insertar data en ella, pero la esclava debe estar en modo de solo lectura

1. Inserta data sobre la tabla replica_test en el Maestro
```
postgres=# INSERT INTO replica_test VALUES ('howtoforge.com');
INSERT 0 1
postgres=# INSERT INTO replica_test VALUES ('This is from Master');
INSERT 0 1
postgres=#
postgres=# select * from replica_test;
        test
---------------------
 howtoforge.com
 This is from Master
(2 rows)

postgres=#
```

Sobre el esclavo intentar hacer el otro insert. El resultado debe ser que no puede escribir, pero el select debe mostrar la data replicada.
```
postgres=# INSERT INTO replica_test VALUES ('pg replication by hakase-labs');
ERROR:  cannot execute INSERT in a read-only transaction
postgres=#
postgres=# select * from replica_test;
        test
---------------------
 howtoforge.com
 This is from Master
(2 rows)

postgres=#
```

## Test de failover controlado

1. Bajar el servicio postgresql en el maestro simulando una falla.
   ```
   [root@db1 ~]# systemctl stop rh-postgresql10-postgresql
   ```
   > **NOTA** Los nodos debe reportar fallas sobre la interface web.

2. Bajar el servicio de tower en los 3 workers
   ``` 
    [root@tower1 ~]# ansible-tower-service stop
    [root@tower2 ~]# ansible-tower-service stop
    [root@tower3 ~]# ansible-tower-service stop
   ```
   
3. Editar el fichero /etc/tower/conf.d/postgres.py  para apuntar al nodo esclavo en los 3 nodos workers
```
# Ansible Tower database settings.

DATABASES = {
   'default': {
       'ATOMIC_REQUESTS': True,
       'ENGINE': 'awx.main.db.profiled_pg',
       'NAME': 'awx',
       'USER': 'awx',
       'PASSWORD': """XXXXXXXX""",
       'HOST': 'db2.example.com',  <<<<<<< parametro a ajustar con el nodo esclavo
       'PORT': '5432',
       'OPTIONS': { 'sslmode': 'prefer',
                    'sslrootcert': '/etc/pki/tls/certs/ca-bundle.crt',
       },
   }
}
```

4. Sobre el server de base de datos esclavo realizar lo siguiente:
   * Renombrar el fichero  ```/var/opt/rh/rh-postgresql10/lib/pgsql/data/recovery.conf``` a  ```/var/opt/rh/rh-postgresql10/lib/pgsql/data/old_recovery.conf_old```  
   *  Comentar los siguientes parametros sobre el fichero  ```/var/opt/rh/rh-postgresql10/lib/pgsql/data/postgresql.conf```  y ajustar la direccion del listen_address con la direccion del nodo esclavo.
   ```
   listen_addresses = 'localhost,172.160.10.66'
   
   ... omited
   #archive_mode = on
   #archiv_command = 'rsync -a %p postgres@172.16.10.66:/var/log/rh/rh0postgresql10/lib/pgsql/archive/%f'
   #hot_standby = on 
   ```
5. Comentar la linea `host   replication     replicate   172.18.47.59/32       scram-sha-256` sobre el fichero `/var/opt/rh/rh-postgresql10/lib/pgsql/data/pg_hba.conf`
   ```
   # !!! This file managed by Ansible. Any local changes may be overwritten. !!!

   # Database administrative login by UNIX sockets
   # note: you may wish to restrict this further later
   local   all         postgres                                peer

   # TYPE  DATABASE    USER        CIDR-ADDRESS          METHOD
   local   all             all                              scram-sha-256
   host    all             all        0.0.0.0/0             scram-sha-256 
   host    all             all        ::/0                  scram-sha-256
   # host   replication     replicate   172.18.47.59/32       scram-sha-256  <<<<< Comentar esta linea.
   ```
   
6. Iniciar el servicio postgresql sobre el esclavo
   ```
    [root@db2 ~]# systemctl start rh-postgresql10-postgresql
   ```
   
7. Por ultimo iniciar servicios de tower en todos los nodos workers
   ```
    [root@tower1 ~]# ansible-tower-service start
    [root@tower2 ~]# ansible-tower-service start
    [root@tower3 ~]# ansible-tower-service start
   ```
   
