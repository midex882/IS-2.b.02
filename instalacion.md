# Instalación y configuración SIEM (ELK) con Nginx-filebeat

**Actividad:** 2.b.02 — Implementación y Configuración de SIEM  
**Objetivo:** Dejar montado un stack ELK con un agente nginx+Filebeat que recoja logs y los deje listos en Kibana para explotar el caso de uso de fuerza bruta SSH.

## 1. Escenario y arquitectura

Este laboratorio se basa en dos contenedores Docker conectados a una red bridge propia, más un docker Kali para hacer de atacante.

Contenedor SOC (SIEM): `sebp/elk:7.16.3`  
Servicios expuestos que vamos a tener: Kibana (5601), Elasticsearch (9200) y Logstash con input Beats (5044).

Contenedor endpoint/agente: `nginx-filebeat` (nginx + Filebeat)  
Nginx escucha en 8080 y Filebeat envía los logs a Logstash sobre el puerto 5044.

Red Docker utilizada: `elk-red` con la subred `172.20.0.0/24`.  
Al contenedor ELK se le fija la IP `172.20.0.10` para evitar problemas de resolución y cambios de IP al reiniciar.

Tenemos que poder provocar alertas en elk por medio de pings y posteriormente usando hydra.

## 2. Instalación del SIEM (contenedor ELK)

### 2.1 Creación de la red Docker

```
docker network create -d bridge --subnet 172.20.0.0/24 elk-red
```
<img width="824" height="91" alt="Captura de pantalla 2026-02-18 095026" src="https://github.com/user-attachments/assets/613bef4a-16ff-4737-9892-2986f70ee706" />

### 2.2 Descarga de la imagen


Se trabaja con la versión 7.16.3 para mayor compatibilidad entre agente y servidor.

```
docker pull sebp/elk:7.16.3
```
<img width="945" height="238" alt="Captura de pantalla 2026-02-18 091127" src="https://github.com/user-attachments/assets/40e1ed43-3064-4d71-bc93-5b055fb3713b" />


### 2.3 Puesta en marcha del contenedor

Se levanta el contenedor dentro de la red `elk-red` con IP fija para que el resto de servicios puedan referenciarlo sin sorpresas.

```
docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 \
  -it --name elk --net elk-red --ip 172.20.0.10 -d sebp/elk:7.16.3
```
<img width="877" height="182" alt="2026-02-19 20_01_49-Parsec" src="https://github.com/user-attachments/assets/bc5d1338-634b-45b4-9b02-586455e25062" />



### 2.4 Comprobaciones básicas

Kibana:

```
curl -i http://localhost:5601/
```
<img width="871" height="324" alt="2026-02-20 23_30_02-Parsec" src="https://github.com/user-attachments/assets/c2f8fb02-040a-4044-a05b-7d5f8036a19a" />

Elasticsearch:

```
curl -s http://localhost:9200/
```

## 3. Configuración del agente

### 3.2 Build y arranque del agente

Desde el directorio `nginx-filebeat/`:
<img width="947" height="536" alt="Captura de pantalla 2026-02-18 121211" src="https://github.com/user-attachments/assets/03a3f74a-9ebf-4b0d-9185-3f7d2fada70c" />

```
docker build -t nginx-filebeat .
docker run -d --name filebeat --net elk-red -p 8080:80 nginx-filebeat
```

### 3.3 Configuración de `filebeat.yml`

Queremos que el agente recoga los logs, sin más.

<img width="898" height="687" alt="2026-02-19 20_32_25-Parsec" src="https://github.com/user-attachments/assets/698b411c-a7ff-47c1-af3e-403546ba7b22" />


## 4. Validación de la ingesta

### 4.1 Pruebas de salida

Para verificar conectividad y configuración hacia ELK:

```
filebeat test output -e
```
<img width="947" height="536" alt="Captura de pantalla 2026-02-18 121211" src="https://github.com/user-attachments/assets/d6b8e777-382d-4df2-8f7c-abebffcd0688" />
Este error se debía a que ELK no estaba arrancado. Se eencendió y pasó a OK



Y para comprobar que el YAML no tiene errores de sintaxis:

```
filebeat test config -e
```

<img width="872" height="237" alt="2026-02-19 20_34_36-Parsec" src="https://github.com/user-attachments/assets/4a0008a6-d3dc-4235-ac7c-418a164de075" />

Con la configuración en OK también, seguimos


### 4.2 Índices en Elasticsearch

Se valida que se estén generando índices (por ejemplo los de Filebeat) en Elasticsearch:

```
curl -s http://localhost:9200/_cat/indices?v
```

<img width="885" height="464" alt="2026-02-19 20_36_22-Parsec" src="https://github.com/user-attachments/assets/db625cc9-1748-45b9-849c-019945b1f589" />


## 5. Kibana

Para trabajar los datos en Kibana seguimos accediendo en nuestro navegador a `http://localhost:5601`:

Y seguimos este orden
<img width="1843" height="913" alt="2026-02-19 20_39_02-Parsec" src="https://github.com/user-attachments/assets/cd68f2b2-7ff0-4640-893d-5a23ae6d6454" />

Kibana >  Stack Management > Index Patterns > Crear un nuevo patrón:  
   - Index pattern: `filebeat-*`  
   - Time field: `@timestamp` (si está presente en los documentos).  
5. Ir a Discover y seleccionar el patrón creado.

  
Pero aquí vemos que no se generan eventos nuevos

En el contenedor se comprobó que sí se estaban rellenando en `/var/log/nginx/access.log` al lanzar peticiones con `curl http://localhost:8080/`, pero no se nos mostraba en `filebeat-*`.

Resultó que, por alguna razón, filebeat estaba detenido, por lo que lo reiniciamos:

Antes, hubo que instalar systemctl, pues el contenedor no lo tenía instalado, y nos hacía falta para arrancar el servicio.

```
apt install systemctl
```
Con systemctl instalado, seguimos

```
systemctl start filebeat
systemctl status filebeat --no-pager
```
<img width="1810" height="973" alt="2026-02-20 18_16_38-" src="https://github.com/user-attachments/assets/54c59e2e-e50f-4c7a-ab6e-b2449585b99a" />

## 6. Instalación de Snort 2.9.20 (IDS)

Para instalar snort en nuestro contenedor, basta con hacer:

```
apt install snort
```
<img width="895" height="516" alt="2026-02-20 18_40_15-Parsec" src="https://github.com/user-attachments/assets/c7dd3814-032e-4ad2-9f41-9c84a5486f7b" />

Snort nos pregunta en qué puerto debería escuchar, así que le decimos: eth0

Y luego nos pide el rango de direcciones:

<img width="900" height="277" alt="2026-02-20 18_40_28-Parsec" src="https://github.com/user-attachments/assets/dbbd4fde-5f7a-4d8a-b851-5a6ccca1a017" />

Después añadimos reglas a snort:

<img width="900" height="289" alt="2026-02-20 18_44_25-Parsec" src="https://github.com/user-attachments/assets/bf32e0f9-f9b8-4188-bdcb-4810dd272270" />

Y hacemos una comprobación de configuración:

<img width="892" height="754" alt="2026-02-20 18_45_07-Parsec" src="https://github.com/user-attachments/assets/3b991a09-6ef7-4eb6-b6f4-c6157890c5b5" />



### Test de conexión

Desde el contenedor de Kali le leanzamos un ping al contenedor de filebeat dentro de la misma red Docker para generar tráfico ICMP y comprobar si snort genera la alerta, y snort nos muestra un: 

```
02/20-19:05:20.123145 [] [1:30000001:0] !Tráfico ICMP! [] [Priority: 0] {ICMP} 172.20.0.1 -> 172.20.0.2

```

Por lo que sabemos que funciona.

## Hydra

Por desgracia, llegados a este punto, no pude continuar con la práctica. Los ataques desde Hydra no se conectan con mis contenedores de ELK y Filebeat. Las soluciones probadas fueron:
 - Cambiar de Kali en WSL a máquina virtual en virtualbox
 - Cambiar de Kali en virtualbox a Kali en docker.

Me aseguré de que Kali podía hacer pink a sus máquinas hermanas:
<img width="890" height="568" alt="2026-02-20 21_03_51-Parsec" src="https://github.com/user-attachments/assets/e66de493-4cc8-47f0-aef2-f44ea018fe52" />

Intenté en numerosas ocasiones lanzar los scripts después de rellenarlos con los comandos proporcionados por el profesor:
<img width="376" height="450" alt="2026-02-20 21_40_17-Parsec" src="https://github.com/user-attachments/assets/83ad3077-ddc7-4229-afb3-3f2182f2a075" />

Sin éxito. Con esto, y dadas mis condiciones de salud en el momento, me temo que no hay más que pueda hacer, pero consultaré con mis compañeros para entender qué puedo haber hecho mal, y resolveré el error.



