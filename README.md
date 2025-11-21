# Pila MING (Mosquitto, Node-RED, InfluxDB, Grafana) - Ingesta de Criptomonedas

Este repositorio contiene la configuración para desplegar una pila de observación/IoT ligera (MING) utilizando Docker Compose. El objetivo principal es la ingesta robusta del precio de Bitcoin (BTC) desde la API pública de CoinGecko, para su posterior visualización en un cuadro de mando en Grafana.

## 1. Despliegue del Stack

El stack está diseñado para ser levantado utilizando Docker Compose, con una red interna (`red-practica1-sbd`) y volúmenes persistentes para asegurar que los datos de InfluxDB, Grafana y Node-RED se conserven entre reinicios.

### Requisitos
Asegúrese de tener Docker y Docker Compose instalados en su sistema.

### Comando de Arranque

Para desplegar y levantar todos los servicios en segundo plano, use el siguiente comando:

```bash
docker compose up -d
```

### Estructura de Servicios

| Servicio | Imagen Base | Puerto Expuesto (Host) | Red Interna | Dependencias |
| :--- | :--- | :--- | :--- | :--- |
| **mosquitto** | `sbd-mosquitto:latest` | 1883, 8883, 9001 | `red-practica1-sbd` | N/A |
| **node-red** | `nodered/node-red:latest` | 1880 | `red-practica1-sbd` | `mosquitto` |
| **influxdb** | `influxdb:2.7.12-alpine` | 8086 | `red-practica1-sbd` | `node-red` |
| **grafana** | `grafana/grafana:latest` | 3000 | `red-practica1-sbd` | `influxdb` |

---

## 2. Endpoints y Credenciales por Defecto

Los siguientes *endpoints* están disponibles en su máquina *host* (asumiendo `localhost` o la IP de su *host* Docker):

| Servicio | Endpoint de Acceso | Usuario por Defecto | Contraseña por Defecto |
| :--- | :--- | :--- | :--- |
| **Node-RED** | `http://localhost:1880` | N/A | N/A |
| **InfluxDB** | `http://localhost:8086` | **`admin`** | **`Alandalus2526`** |
| **Grafana** | `http://localhost:3000` | **`admin`** | **`Alandalus2526`** |

### Configuración de InfluxDB y Tokens

El *stack* está configurado para inicializar InfluxDB v2 utilizando el modo `setup` y variables de entorno.

| Parámetro | Valor | Fuente |
| :--- | :--- | :--- |
| **Organización (ORG)** | `sbd` | `DOCKER_INFLUXDB_INIT_ORG` |
| **Bucket Inicial** | `coingecko` | `DOCKER_INFLUXDB_INIT_BUCKET` |
| **Token de Autorización** | `E26MZMrIxjVdASs6YCPutZ_ps5sM_XQkoyFbxGi2x_qW8ZPTMth-nkb5CNJ_xlvXvMTjhNo_JJqfpPz5rEhr9g==` | Usado en el nodo HTTP Request de Node-RED |

---

## 3. Variables de Entorno Utilizadas

### Variables de InfluxDB (Inicialización)

| Variable | Descripción | Valor | Fuente |
| :--- | :--- | :--- | :--- |
| `DOCKER_INFLUXDB_INIT_MODE` | Modo de inicialización | `setup` | |
| `DOCKER_INFLUXDB_INIT_USERNAME` | Usuario inicial | `admin` | |
| `DOCKER_INFLUXDB_INIT_PASSWORD` | Contraseña inicial | `Alandalus2526` | |
| `DOCKER_INFLUXDB_INIT_ORG` | Organización | `sbd` | |
| `DOCKER_INFLUXDB_INIT_BUCKET` | Bucket inicial | `coingecko` | |


## 4. Flujo de Ingesta (Node-RED)

El flujo de Node-RED se encarga de obtener el precio de Bitcoin (BTC) y persistirlo en InfluxDB.

### Frecuencia y API
El nodo `inject` está configurado para solicitar los datos de CoinGecko cada **10 minutos** (`repeat: "600"`). La API utilizada para la petición HTTP incluye la variación de 24 horas (`include_24hr_change=true`).

### Esquema de Persistencia en InfluxDB

Los datos se formatean siguiendo el esquema requerido para InfluxDB v2:

| Componente | Clave | Valor/Tipo |
| :--- | :--- | :--- |
| **Bucket** | `coingecko` | (Usado en la URL de `http request`) |
| **Measurement** | `crypto_price` | |
| **Tag** | `symbol` | `"BTC"` |
| **Tag** | `currency` | `"EUR"` |
| **Field** | `price` | `float` |
| **Field** | `change_24h` | `float` (Mapeado desde la API) |

### Robustez Implementada
El flujo cumple con la **Robustez mínima** solicitada:
1. **Respeto por la cadencia:** La inyección inicial ocurre exactamente cada 10 minutos.
2. **Manejo de Errores y Reintento:** En caso de fallo HTTP (código ≠ 200) en la petición a CoinGecko, se implementa una lógica de `Catch`, `Log` y `Delay` para reintentar la solicitud antes de declarar un fallo crítico.
