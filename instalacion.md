# Instalación y configuración SIEM (ELK) + Agente (nginx-filebeat)

**Actividad:** 2.b.02 — Implementación y Configuración de SIEM  
**Objetivo:** Dejar montado un stack ELK con un agente nginx+Filebeat que recoja logs y los deje listos en Kibana para explotar el caso de uso de fuerza bruta SSH.

## 1. Escenario y arquitectura

Este laboratorio se basa en dos contenedores Docker conectados a una red bridge propia.

Contenedor SOC (SIEM): `sebp/elk:7.16.3`  
Servicios expuestos que vamos a tener: Kibana (5601), Elasticsearch (9200) y Logstash con input Beats (5044).

Contenedor endpoint/agente: `nginx-filebeat` (nginx + Filebeat)  
Nginx escucha en 8080 y Filebeat envía los logs a Logstash sobre el puerto 5044.

Red Docker utilizada: `elk-red` con la subred `172.20.0.0/24`.  
Al contenedor ELK se le fija la IP `172.20.0.10` para evitar problemas de resolución y cambios de IP al reiniciar.

Tenemos que poder provocar alertas en elk por medio de pings y posteriormente usando hydra.

## 2. Instalación del SIEM (contenedor ELK)

### 2.1 Creación de la red Docker

```bash
docker network create -d bridge --subnet 172.20.0.0/24 elk-red
```

### 2.2 Descarga de la imagen

Se trabaja con la versión 7.16.3 para mayor compatibilidad entre agente y servidor.

```bash
docker pull sebp/elk:7.16.3
```

### 2.3 Puesta en marcha del contenedor

Se levanta el contenedor dentro de la red `elk-red` con IP fija para que el resto de servicios puedan referenciarlo sin sorpresas.

```bash
docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 \
  -it --name elk --net elk-red --ip 172.20.0.10 -d sebp/elk:7.16.3
```



### 2.4 Comprobaciones básicas

Kibana:

```bash
curl -i http://localhost:5601/
```

Elasticsearch:

```bash
curl -s http://localhost:9200/
```



## 3. Configuración del agente (nginx-filebeat)


### 3.2 Build y arranque del agente

Desde el directorio `nginx-filebeat/`:

```bash
# Construir imagen personalizada
docker build -t nginx-filebeat .

# Arrancar el agente
docker run -d --name filebeat --net elk-red -p 8080:80 nginx-filebeat
```

### 3.3 Configuración de `filebeat.yml`

El agente se deja en modo sencillo: recoge logs de Nginx y los envía a Logstash sin TLS (en esta fase nos interesa funcionalidad, no endurecer aún el canal).



## 4. Validación de la ingesta

### 4.1 Pruebas de salida

Para verificar conectividad y configuración hacia ELK:

```bash
filebeat test output -e
```



Y para comprobar que el YAML no tiene errores de sintaxis:

```bash
filebeat test config -e
```



### 4.2 Índices en Elasticsearch

Se valida que se estén generando índices (por ejemplo los de Filebeat) en Elasticsearch:

```bash
curl -s http://localhost:9200/_cat/indices?v
```



## Errores de TLS y compatibilidad de versiones

### Problema detectado

En Logstash aparecían errores de handshake TLS (`bad_certificate`) y Filebeat no terminaba de enviar.  
Además, en `filebeat test output -e` aparecieron mensajes del tipo `x509: certificate has expired or is not yet valid` al intentar negociar TLS.

La combinación que fallaba era: contenedor `nginx-filebeat` con Filebeat 8.14.3 hablando con un ELK 7.16.3. A esto se suma que la configuración TLS no estaba cerrada (CA, certificados, etc.), por lo que cualquier verificación extra rompía el canal.

### Enfoque y solución

Primero se alinean versiones: Filebeat pasa a 7.16.3 para ir en paralelo con ELK 7.16.3.  
Segundo, se simplifica: se desactiva SSL en el output hacia Logstash (`ssl.enabled: false`) mientras sea solo una PoC.  
Por último, se ajusta el `filebeat.yml` a la sintaxis correcta para la rama 7.x (`filebeat.inputs`, `output.logstash`, etc.).

### Configuración final de `filebeat.yml`

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/*.log
    fields:
      app: nginx
    fields_under_root: true

setup.template.enabled: false
setup.ilm.enabled: false

output.logstash:
  hosts: ["172.20.0.10:5044"]
  ssl.enabled: false

setup.kibana:
  host: "elk:5601"

logging.level: info
```

## 5. Visualización en Kibana

Para trabajar los datos en Kibana se siguen estos pasos sobre `http://localhost:5601`:

1. Acceder a Kibana.  
2. Ir a Stack Management.  
3. Entrar en Index Patterns.  
4. Crear un nuevo patrón:  
   - Index pattern: `filebeat-*`  
   - Time field: `@timestamp` (si está presente en los documentos).  
5. Ir a Discover y seleccionar el patrón creado.

  


### Problema: no llegaban eventos nuevos

En Discover no aparecían logs nuevos a pesar de refrescar.  
En el contenedor se comprobó que Nginx sí estaba generando entradas en `/var/log/nginx/access.log` al lanzar peticiones con `curl http://localhost:8080/`, pero esos eventos no terminaban en el índice `filebeat-*`.

El root cause fue sencillo: el servicio de Filebeat estaba parado (`Active: inactive (dead)` en `systemctl status filebeat`), así que no se enviaba nada.

Se solucionó arrancando el servicio:

```bash
systemctl start filebeat
systemctl status filebeat --no-pager
```

## 6. Instalación de Snort 2.9.20 (IDS)

### 6.1 Objetivo

Compilar e instalar Snort 2.9.20 desde fuente dentro del contenedor para poder detectar tráfico con reglas propias (por ejemplo en `local.rules`) y generar alertas.

### 6.2 Paquetes previos

Se instalan herramientas básicas de compilación y dependencias habituales:

```bash
apt-get update
apt-get install -y build-essential bison flex libpcap-dev zlib1g-dev \
  libdumbnet-dev pkg-config
```

El resto de dependencias se van añadiendo conforme `configure` y `make` van mostrando errores.

### 6.3 Descarga del código fuente (DAQ + Snort)

```bash
cd /opt/snort_src
wget https://www.snort.org/downloads/snort/daq-2.0.7.tar.gz
wget https://www.snort.org/downloads/snort/snort-2.9.20.tar.gz
tar -xzf daq-2.0.7.tar.gz
tar -xzf snort-2.9.20.tar.gz
```

### 6.4 Compilación e instalación de DAQ

```bash
cd /opt/snort_src/daq-2.0.7
./configure
make
make install
ldconfig
```

DAQ actúa como capa de captura de paquetes que Snort 2.9.x utiliza por debajo.

### 6.5 Compilación e instalación de Snort

```bash
cd /opt/snort_src/snort-2.9.20
./configure --enable-sourcefire --disable-open-appid
make
make install
ldconfig
```

Se usa `--enable-sourcefire` siguiendo la recomendación oficial, y `--disable-open-appid` para evitar arrastrar dependencias adicionales como LuaJIT en esta PoC.

### 6.6 Verificación

```bash
snort -V
which snort
```



### 6.7 Errores durante el build y su resolución

#### Error: `Libpcre header not found` / `pcre-config: command not found`

En `./configure` aparece:

```text
pcre-config: command not found
checking for pcre.h... no
ERROR! Libpcre header not found.
```

En Debian 13 no está disponible `libpcre3-dev`, así que se opta por compilar PCRE 8.45 desde fuente en `/usr/local/pcre` y revisar la versión con:

```bash
/usr/local/pcre/bin/pcre-config --version  # 8.45
```

Después se exponen las rutas a `configure`:

```bash
export PATH="/usr/local/pcre/bin:$PATH"
export CPPFLAGS="-I/usr/local/pcre/include"
export LDFLAGS="-L/usr/local/pcre/lib"
```

#### Error: `LuaJIT library not found` (OpenAppID)

Durante `./configure` se ve:

```text
checking for luajit... no
ERROR! LuaJIT library not found ...
```

La causa es que se intenta compilar con soporte OpenAppID, que requiere LuaJIT. La solución es recompilar desactivando OpenAppID:

```bash
./configure --enable-sourcefire --disable-open-appid
```

#### Error: `fatal error: rpc/rpc.h: No such file or directory`

Falló el plugin `sp_rpc_check.c` al hacer `make`:

```text
fatal error: rpc/rpc.h: No such file or directory
```

En sistemas recientes se trabaja con TIRPC, por lo que se instala:

```bash
apt-get install -y libtirpc-dev
```

Y se fuerza el include y el enlace:

```bash
export CPPFLAGS="-I/usr/include/tirpc ${CPPFLAGS:-}"
export LIBS="-ltirpc ${LIBS:-}"
```

## Problemas de configuración de Snort (`snort.conf`) y ajustes

La validación se hace con:

```bash
snort -T -c /etc/snort/snort.conf
```

A partir de aquí van apareciendo errores encadenados, porque el `snort.conf` es genérico y este despliegue concreto no tiene todas las rutas y rulesets que el fichero espera.

### `snort.conf` ausente

Primero, Snort no encuentra el archivo:

```text
Unable to open rules file "/etc/snort/snort.conf": No such file or directory
```

Al instalar desde fuente, el binario sí queda en el sistema, pero los ficheros de configuración hay que copiarlos manualmente desde `snort-2.9.20/etc/` a `/etc/snort/`:

```bash
cp /opt/snort_src/snort-2.9.20/etc/*.conf /etc/snort/
cp /opt/snort_src/snort-2.9.20/etc/*.map /etc/snort/
```

## Verificación de Snort en modo consola

### Reglas locales (`local.rules`)

Para probar detección rápida se usan reglas en `/etc/snort/rules/local.rules`:

```bash
# Regla para detectar un ping (ICMP)
alert icmp any any -> $HOME_NET any (msg:"!Trafico ICMP!"; sid:3000001;)

# Regla para detectar SSH (con limitación de eventos)
alert tcp any any -> any 22 (msg:"Acceso SSH"; sid:3000002; flow:established; threshold: type limit, track by_src, count 1, seconds 30;)
```

La primera sirve como smoke test con un simple ping.  
La segunda alerta sobre conexiones SSH, limitando el spam con un threshold de una alerta cada 30 segundos por IP origen.



### Ejecución en modo IDS por consola

```bash
snort -A console -q -c /etc/snort/snort.conf -i eth0
```

Parámetros relevantes: salida por consola, modo quiet, carga de `snort.conf` e interfaz `eth0`.

### Tráfico de prueba

Desde Kali se lanza un ping al contenedor `nginx-filebeat` dentro de la misma red Docker para generar tráfico ICMP y comprobar si se dispara la alerta configurada.

### Resultado

Se observa que Snort genera alertas con el mensaje `!Trafico ICMP!` y el SID configurado (`3000001`).



## Caso de uso: fuerza bruta SSH con Hydra

### Contexto

Para probar el caso de fuerza bruta SSH se utilizan los scripts proporcionados (`crear_claves`, `crear_usuarios` y `lanzar_hydra`) con la idea de automatizar la preparación del entorno y el ataque.

### Preparación con scripts

Orden lógico de ejecución:

- `crear_claves` para generar las credenciales.  
- `crear_usuarios` para dar de alta los usuarios objetivo.  
- `lanzar_hydra` para lanzar el ataque SSH.

El objetivo es que Snort genere alertas y se registren en `/var/log/snort/alert` de forma consumible por ELK.

### Primer problema

Aunque Hydra estaba trabajando, las alertas no se registraban como se esperaba.  
Revisando, se ve que:

- La regla dependía de `flow:established`.  
- Parte del tráfico no se consideraba “establecido” desde el punto de vista de Snort, por lo que la condición no saltaba.

### Ajustes de la regla y ejecución

Como paso intermedio se flexibiliza la condición (por ejemplo quitando el `established` del `flow`) para ver más intentos, y se modifica la forma de lanzar Snort:

```bash
snort -A fast -q -c /etc/snort/snort.conf -i eth0 -k none -l /var/log/snort
```

Puntos clave:

- `-A fast` genera alertas en formato rápido, ideal para ficheros que luego leerá Filebeat.  
- `-l /var/log/snort` fuerza el directorio para los logs.  
- `-k none` ignora el checksum, útil en laboratorios con offloading que puede dar lecturas raras. No es una opción recomendable para producción.

Con estos cambios, Snort empieza a volcar alertas SSH de forma consistente a `/var/log/snort/alert`.

  


### Directorio de reglas dinámicas inexistente

Error:

```text
Could not stat dynamic module path "/usr/local/lib/snort_dynamicrules": No such file or directory.
```

Se soluciona creando el directorio esperado:

```bash
mkdir -p /usr/local/lib/snort_dynamicrules
```

### Problemas con `RULE_PATH` y `local.rules`

Error:

```text
Unable to open rules file "/etc/snort/../rules/local.rules": No such file or directory.
```

El `RULE_PATH` venía como ruta relativa (`../rules`), lo cual en este contexto manda a `/etc/rules`. Se corrige definiendo una ruta absoluta y ajustando el include:

```bash
var RULE_PATH /etc/snort/rules
include $RULE_PATH/local.rules
```

Y se crea la estructura básica:

```bash
mkdir -p /etc/snort/rules
touch /etc/snort/rules/local.rules
```

### Reglas de OpenAppID/AppID

Error al intentar incluir `app-detect.rules` y otras reglas relacionadas, cuando en realidad Snort se compiló con `--disable-open-appid`.

La solución en esta PoC pasa por comentar las líneas `include` relacionadas con AppID/OpenAppID en `snort.conf`, para que no intente cargar reglas que no existen.

### Reglas por defecto ausentes

`snort.conf` referencia múltiples `.rules` (por ejemplo `attack-responses.rules`) que no se han desplegado porque solo se está usando `local.rules`.

Para esta práctica se opta por dejar activas únicamente las reglas locales y comentar el resto de includes que apuntan a ficheros no presentes.

### Resultado de la validación

Tras los ajustes de rutas, directorios y includes, la validación de Snort termina correctamente.



## Configuración de Filebeat para Nginx y Snort

### Ficheros a monitorizar

En el contenedor agente interesan estos paths:

- Nginx: `/var/log/nginx/*.log`  
- Snort (alertas): `/var/log/snort/alert`

Como prerequisito, Snort debe estar corriendo en modo que escriba alertas a fichero:

```bash
snort -A fast -q -c /etc/snort/snort.conf -i eth0 -k none -l /var/log/snort
```

Esto hace que el archivo `/var/log/snort/alert` se vaya alimentando con cada alerta generada.

Verificación rápida:

```bash
ls -lh /var/log/snort/alert
tail -n 5 /var/log/snort/alert
```

### Configuración de Filebeat

Se añaden inputs apuntando a los logs reales (Nginx y Snort) para que Filebeat recoja cualquier línea nueva que aparezca. La idea es simple: si un path no está definido en los inputs, Filebeat no lo va a enviar.



El output se configura hacia Logstash en el puerto Beats habitual (5044):

```yaml
output.logstash:
  hosts: ["elk:5044"]
```

### Ejecución dentro del contenedor

En el contenedor, `service filebeat start` no mantiene el proceso vivo por la ausencia de systemd clásico, así que se opta por ejecutarlo en primer plano.

Comprobaciones:

```bash
filebeat test config -e
filebeat test output -e
```

Ejecución recomendada:

```bash
filebeat -e
```

Con esto, Filebeat permanece corriendo y enviando eventos en tiempo real. Las evidencias finales muestran que las alertas tanto de ICMP como de SSH se están enviando correctamente a ELK.