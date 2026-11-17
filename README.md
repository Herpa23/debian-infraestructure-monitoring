# Laboratorio de Monitorización y Observabilidad - Stack Docker

Este repositorio documenta 00.READMEla implementación de un entorno de observabilidad centralizado utilizando contenedores Docker, diseñado para recolectar métricas de rendimiento y registros (_logs_) de sistemas Linux de forma autónoma.

## 📁 Estructura del Directorio y Obtención de Ficheros

Para mantener el entorno organizado y reproducible, todo el stack se aloja en el directorio `/opt/monitoring` en el nodo central (`debian-monitor`)

```plaintext 
/opt/monitoring/
├── docker-compose.yml
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasource.yml
├── loki-config.yaml
├── loki-data/
└── prometheus.yml
```

### Configuración de Prometheus (`prometheus.yml`)

- **Cómo se obtuvo:** Creado de forma manual para definir los intervalos de sondeo globales y la integración con el ecosistema.
    
- **Función:** Establece las reglas de recolección de métricas y prepara el motor para recibir datos mediante _remote-write_.
    

### 2. Configuración de Loki (`loki-config.yaml`)

- **Cómo se obtuvo:** Se descargó directamente desde el repositorio oficial de Loki utilizando la herramienta `wget` para asegurar una estructura limpia y compatible con el almacenamiento de registros:
    
    Bash
    
    ```
    sudo wget https://raw.githubusercontent.com/grafana/loki/main/cmd/loki/loki-local-config.yaml -O loki-config.yaml
    ```
    
- **Función:** Define los parámetros internos del motor de logs, la retención temporal y las rutas de almacenamiento para las consultas en Grafana.
    

### 3. Orquestación con Docker Compose (`docker-compose.yml`)

- **Cómo se obtuvo:** Diseñado para unificar el despliegue concurrente de Prometheus, Grafana y Loki.
    
- **Función:** Configura redes virtuales aisladas (`monitoring_net`), volúmenes persistentes y políticas de reinicio automático (`restart: unless-stopped`) para garantizar la alta disponibilidad del stack ante reinicios del sistema operativo.

## 🛠️ Arquitectura y Componentes del Stack

El stack de monitoreo se encuentra desplegado utilizando **Docker Compose**, centralizando la recepción de datos provenientes de los nodos de infraestructura:

|**Componente**|**Puerto**|**Función Principal**|
|---|---|---|
|**Prometheus**|`9090`|Motor central de almacenamiento de series de tiempo y evaluación de métricas.|
|**Grafana**|`3000`|Interfaz web para la visualización de datos, dashboards y paneles de control.|
|**Loki**|`3100`|Sistema de agregación y almacenamiento escalable de registros (_logs_).|
|**Grafana Alloy**|Agente local|Agente unificado instalado en los nodos remotos encargado de recolectar y enviar métricas y logs mediante _push_.|

## 🚀 Pasos de Implementación

### 1. Despliegue del Stack de Monitoreo (`debian-monitor`)

Se configuró un archivo `docker-compose.yml` para levantar los servicios principales asegurando persistencia mediante volúmenes locales y comunicación interna en una red dedicada (`monitoring_net`).

YAML

```
version: '3.8'

networks:
  monitoring_net:
    driver: bridge

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-remote-write-receiver'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - monitoring_net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - monitoring_net

  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/tmp/loki
    networks:
      - monitoring_net

volumes:
  prometheus_data:
  grafana_data:
```


## A. Deslgose del archivo compse

Para garantizar que nuestro stack de observabilidad recupere su estado operativo de forma autónoma ante un reinicio del sistema operativo o un fallo imprevisto, se implementaron las siguientes directivas clave en cada contenedor:

- **`restart: unless-stopped`**: Configura una política de autorrecuperación. Si el contenedor se encontraba ejecutándose al apagarse el host, Docker lo volverá a iniciar automáticamente al arrancar el demonio (`systemctl enable docker`).
    
- **Volúmenes Nombrados (`volumes`)**: Aseguran que los datos históricos de métricas de Prometheus y los registros almacenados en Loki no se pierdan al destruir o actualizar los contenedores.


### 2. Configuración del Agente de Recolección (`Grafana Alloy`) en Nodos Remotos (`debian-infra`)

En lugar de depender exclusivamente del método tradicional de sondeo (_pull_), se implementó **Grafana Alloy** en los servidores objetivo para transmitir activamente las métricas del sistema y los eventos del _systemd journal_ hacia el servidor central de monitoreo mediante envío directo (_push_). También se agrego un componente para poder revisar los logs manuales del servicio **samba** por separada con el componente `loki.source.file`:

```
prometheus.exporter.unix "local" {
  enable_collectors = ["cpu", "meminfo", "diskstats", "filesystem", "netdev"]
}

prometheus.scrape "metrics_integration" {
    targets    = prometheus.exporter.unix.local.targets
    forward_to = [prometheus.remote_write.endpoint.receiver]
    scrape_interval = "15s"
}

prometheus.remote_write "endpoint" {
    endpoint {
        url = "http://192.168.56.30:9090/api/v1/write"
    }
}

loki.source.journal "journal_logs" {
    path       = "/var/log/journal"
    forward_to = [loki.write.endpoint.receiver]
}

loki.write "endpoint" {
    endpoint {
        url = "http://192.168.56.30:3100/loki/api/v1/push"
    }
}

loki.source.file "samba_logs" {
    targets = [
        {__path__ = "/var/log/samba/*.log", job = "samba"},
    ]
    forward_to = [loki.write.endpoint.receiver]
}
```

### A. Desglose Técnico del Agente de Recolección (`Grafana Alloy`)

El archivo de configuración de Alloy (`/etc/alloy/config.alloy`) opera mediante un grafo modular que procesa y reenvía los datos de forma independiente:

- **`prometheus.exporter.unix "local"`**: Actúa como el recolector de métricas nativas del sistema operativo (recolectando estadísticas de CPU, memoria, estado de discos, sistemas de archivos y rendimiento de red).

- **`prometheus.scrape "metrics_integration"`**: Define la frecuencia temporal (`scrape_interval = "15s"`) con la que Alloy extrae las métricas del exportador local para empaquetarlas.

- **`prometheus.remote_write "endpoint"`**: Establece el canal de comunicación activo (_push_) hacia el servidor central, enviando los bloques de métricas directamente al puerto de Prometheus (`9090`).

- **`loki.source.journal "journal_logs"`**: Captura de forma estructurada los eventos generados por el subsistema `systemd-journald` del sistema operativo (kernel, servicios, fallos de autenticación).

- **`loki.source.file "samba_logs"`**: Realiza un seguimiento en tiempo real (_tailing_) de los archivos de texto plano específicos generados por el servicio de compartición de archivos Samba en `/var/log/samba/*.log`, etiquetándolos bajo el `job = "samba"` para facilitar su filtrado en Loki.

- **`loki.write "endpoint"`**: Reenvía tanto los logs del journal como los registros de Samba hacia el servidor central de Loki mediante el protocolo HTTP POST en el puerto `3100`.

## 📊 Validación y Pruebas

- **Métricas:** Verificación de la recepción de series de tiempo (`node_cpu_seconds_total`) desde Prometheus.
    
- **Logs:** Consulta y filtrado en tiempo real de los registros del sistema operativo en la sección _Explore_ de Grafana utilizando Loki.

## 📚 Referencias y Documentación Oficial 

La arquitectura, el despliegue de los contenedores y la configuración de los componentes de este laboratorio se basaron en las guías técnicas oficiales: 

- [Docker Hub - Prometheus Hardened Images & Guides](https://hub.docker.com/hardened-images/catalog/dhi/prometheus/guides) 
- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/latest/) 
- [Prometheus Configuration Documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) 
- [Grafana Loki Getting Started](https://grafana.com/docs/loki/latest/)
