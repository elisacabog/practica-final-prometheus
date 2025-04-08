
# Practica Final - Prometheus y Grafana

Este proyecto consiste en la configuración de Prometheus y Grafana para monitorizar contenedores Docker que ejecutan una aplicación de ejemplo, así como el uso del sistema de archivos de la máquina. El proyecto incluye la creación de paneles en Grafana para visualizar diferentes métricas como el número de CPUs, el uso del sistema de archivos, y la duración de las solicitudes HTTP.

## Estructura del Proyecto

El repositorio contiene los siguientes archivos:

- **docker-compose.yml**: Archivo de configuración de Docker Compose que inicia los contenedores de Prometheus, Grafana, Node Exporter y una aplicación de ejemplo (prometheus-demo-service).
- **JSON de Dashboards de Grafana**: Definiciones de los dashboards exportadas desde Grafana.

## Pasos para la Implementación

### 1. **Preparación del Entorno**

Antes de comenzar, asegúrate de tener instalados los siguientes prerrequisitos:

- **Docker**: Instalar Docker y Docker Compose.
- **Git**: Para clonar el repositorio y subir los cambios a GitHub.

### 2. **Configuración de Docker Compose**

El archivo `docker-compose.yml` define los siguientes contenedores:

- **Prometheus**: Se configura para escuchar en el puerto 9090 y recolectar métricas de los contenedores de la aplicación.
- **Grafana**: Se configura para escuchar en el puerto 3000 y visualizar los datos de Prometheus.
- **Node Exporter**: Se utiliza para recolectar métricas del sistema de archivos y recursos de la máquina.

```yaml
version: "3.8"
services:
  prometheus-service-demo-0:
    image: docker.io/julius/prometheus-demo-service:latest
    ports:
      - "10000:8080"
    networks:
      - monitoring

  prometheus-service-demo-1:
    image: docker.io/julius/prometheus-demo-service:latest
    ports:
      - "20000:8080"
    networks:
      - monitoring

  prometheus-service-demo-2:
    image: docker.io/julius/prometheus-demo-service:latest
    ports:
      - "30000:8080"
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    depends_on:
      - prometheus
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host/root:ro
    command:
      - "--collector.filesystem"
      - "--collector.filesystem.mount-points-exclude=\"^/(dev|proc|run|sys|media|mnt|var/lib/docker/.+)($|/)\""
    networks:
      - monitoring
  
volumes:
  grafana-storage:

networks:
  monitoring:
    driver: bridge

```

### 3. **Configuración de Prometheus**

El archivo `prometheus.yml` le dice a Prometheus qué métricas recolectar. Este archivo tiene configurado un scrape para Prometheus mismo, los servicios de la aplicación de ejemplo y el Node Exporter.

```yaml
global:
  scrape_interval: 10s
  evaluation_interval: 10s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files: []

scrape_configs:
  # Recolecta métricas de Prometheus mismo
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Recolecta métricas de los servicios de Prometheus demo
  - job_name: 'prometheus-service-demo'
    metrics_path: /metrics
    static_configs:
      - targets:
          - prometheus-service-demo-0:8080
          - prometheus-service-demo-1:8080
          - prometheus-service-demo-2:8080

  # Recolecta métricas de Node Exporter para monitorizar el sistema
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']  # Solo dejamos este trabajo

```

### 4. **Consultas en Prometheus y Dashboards en Grafana**

Las siguientes consultas fueron creadas en Prometheus y mostradas en paneles de Grafana:

- **P90** de la duración de solicitudes HTTP:
  ```prometheus
histogram_quantile(0.90, sum(rate(demo_api_request_duration_seconds_bucket[5m])) by (le, path, method, status))
  ```
  
- **P95** de la duración de solicitudes HTTP:
  ```prometheus
histogram_quantile(0.95, sum(rate(demo_api_request_duration_seconds_bucket[5m])) by (le, path, method))
  ```

- Promedio de duración de solicitudes HTTP:
  ```prometheus
sum(rate(demo_api_request_duration_seconds_sum[5m])) by (path, method) /
sum(rate(demo_api_request_duration_seconds_count[5m])) by (path, method) ```

### 5. **Dashboard en Grafana - Recursos de Cómputo**

En este punto, se ha creado un dashboard en Grafana que incluye una fila (row) dedicada a los recursos de cómputo del sistema. Los paneles creados son los siguientes:

#### 5.1) **Panel de tipo "Stat" - Número de CPUs**

Este panel muestra el número de CPUs disponibles en la máquina. La consulta utilizada para obtener este valor es:

```prometheus
count(count(node_cpu_seconds_total) by (cpu))
```

#### 5.1) **Panel de tipo "Gauge" - Porcentaje de Uso del Sistema de Archivos Raíz**

Este panel muestra el porcentaje de uso del sistema de archivos raíz. La consulta utilizada para obtener el porcentaje de uso es:

```prometheus
100 * (1 - (node_filesystem_free_bytes{mountpoint="/etc/hosts"} / node_filesystem_size_bytes{mountpoint="/etc/hosts"}))
```

Este panel muestra el porcentaje de uso del sistema de archivos raíz, con umbrales de color naranja para el 80% y rojo para el 90%. Para este panel que indica el porcentaje de uso del sistema de archivos, utilicé el directorio `/etc/hosts` como `mountpoint` debido a que en mi entorno de Docker, ese es el directorio raíz desde el cual se pueden obtener las métricas de uso del sistema de archivos. 

### 6. **Subir los Archivos a GitHub**

Una vez que todo estuvo configurado y los dashboards de Grafana estén exportados a formato JSON, los archivos se subieron al repositorio en GitHub, incluyendo:

- `docker-compose.yml`
- `prometheus.yml`
- Archivos JSON con los dashboards de Grafana.

### Conclusión

Este proyecto permite monitorear métricas de un servicio Dockerizado utilizando Prometheus para la recolección de métricas y Grafana para la visualización de datos. He configurado un contenedor de Prometheus que recolecta métricas de la aplicación de ejemplo y de Node Exporter para supervisar los recursos del sistema. Además, he configurado dashboards en Grafana para visualizar el uso del sistema de archivos y las métricas de rendimiento de la aplicación.
