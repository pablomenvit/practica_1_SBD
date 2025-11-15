# practica_1_SBD
Repositoria de la practica 1: Pila MING

# Pila MING (Mosquitto, Node-RED, InfluxDB, Grafana) - Ingesta de Criptomonedas

Este repositorio contiene la configuración para desplegar una pila de observación/IoT ligera (MING) utilizando Docker Compose. El objetivo principal es la ingesta robusta del precio de Bitcoin (BTC) desde la API pública de CoinGecko, para su posterior visualización en un cuadro de mando en Grafana.

## 1. Despliegue del Stack

El stack está diseñado para ser levantado utilizando Docker Compose, con una red interna (`red-ejemplo`) y volúmenes persistentes para asegurar que los datos de InfluxDB, Grafana, Mosquitto y Node-RED se conserven entre reinicios.

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
| **mosquitto** | `sbd-mosquitto:latest` | 1883, 8883, 9001 | `red-ejemplo` | N/A |
| **node-red** | `nodered/node-red:latest` | 1880 | `red-ejemplo` | `mosquitto` |
| **influxdb** | `influxdb:2.7.12-alpine` | 8086 | `red-ejemplo` | `node-red` |
| **coingecko-mqtt** | (Build local) | N/A | `red-ejemplo` | `mosquitto` |
| **grafana** | `grafana/grafana:latest` | 3000 | `red-ejemplo` | `influxdb` |

---

## 2. Endpoints y Credenciales por Defecto

Los siguientes *endpoints* están disponibles en su máquina *host* (asumiendo `localhost` o la IP de su *host* Docker):

| Servicio | Endpoint de Acceso | Usuario por Defecto | Contraseña por Defecto |
| :--- | :--- | :--- | :--- |
| **Node-RED** | `http://localhost:1880` | N/A | N/A |
| **InfluxDB** | `http://localhost:8086` | **`admin`** | **`Alandalus2526`** |
| **Grafana** | `http://localhost:3000` | **`admin`** | (No especificada, usualmente `admin` o se configura al inicio) |
| **Mosquitto** | (Acceso MQTT) | **`admin`** | **`Alandalus2526`** |

### Configuración de InfluxDB y Tokens

El *stack* está configurado para inicializar InfluxDB v2 utilizando el modo `setup` y variables de entorno.

| Parámetro | Valor | Fuente |
| :--- | :--- | :--- |
| **Organización (ORG)** | `sbd` | `DOCKER_INFLUXDB_INIT_ORG` |
| **Bucket Inicial** | `aula` | `DOCKER_INFLUXDB_INIT_BUCKET` |
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
| `DOCKER_INFLUXDB_INIT_BUCKET` | Bucket inicial | `aula` | |

### Variables de Node-RED

| Variable | Descripción | Valor | Fuente |
| :--- | :--- | :--- | :--- |
| `TZ` | Zona horaria | `Europe/Madrid` | |

### Variables de `coingecko-mqtt` (Servicio Adicional)

Este servicio ingesta datos de criptomonedas y los publica en Mosquitto.

| Variable | Descripción | Valor | Fuente |
| :--- | :--- | :--- | :--- |
| `BROKER_HOST` | Host MQTT | `mosquitto` | |
| `BROKER_PORT` | Puerto MQTT | `1883` | |
| `MQTT_USERNAME` | Usuario de conexión MQTT | `admin` | |
| `MQTT_PASSWORD` | Contraseña de conexión MQTT | `Alandalus2526` | |
| `COINS` | Monedas solicitadas | `bitcoin,ethereum,solana` | |
| `INTERVAL_SECONDS` | Cadencia de ingesta MQTT | `30` | |
| `TOPIC_TEMPLATE` | Tema MQTT | `coingecko/{symbol}` | |

---

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
