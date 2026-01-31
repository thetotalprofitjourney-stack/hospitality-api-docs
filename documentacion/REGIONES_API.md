# REGIONES - Configuración Multi-Región Opera Cloud

## Objetivo
Documentar cómo trabajar con cadenas hoteleras que tienen propiedades desplegadas en diferentes regiones geográficas de Oracle Cloud.

---

## El Problema

Una cadena hotelera global puede tener hoteles en diferentes regiones de Oracle Cloud:

```
CADENA HOTELERA "MYCHAIN"
│
├── Región US (us-ashburn-1)
│   ├── Hotel Nueva York
│   ├── Hotel Miami
│   └── Hotel Los Angeles
│
├── Región EU (eu-frankfurt-1)
│   ├── Hotel Madrid
│   ├── Hotel París
│   └── Hotel Londres
│
└── Región APAC (ap-tokyo-1)
    ├── Hotel Tokio
    ├── Hotel Singapur
    └── Hotel Sydney
```

**Cada región tiene su propia URL base y potencialmente credenciales diferentes.**

---

## Regiones Disponibles de Oracle Cloud

| Región | Código | URL Base |
|--------|--------|----------|
| US East (Ashburn) | `us-ashburn-1` | `https://{tenant}.hospitality-api.us-ashburn-1.ocs.oraclecloud.com` |
| EU Central (Frankfurt) | `eu-frankfurt-1` | `https://{tenant}.hospitality-api.eu-frankfurt-1.ocs.oraclecloud.com` |
| APAC (Tokyo) | `ap-tokyo-1` | `https://{tenant}.hospitality-api.ap-tokyo-1.ocs.oraclecloud.com` |
| APAC (Sydney) | `ap-sydney-1` | `https://{tenant}.hospitality-api.ap-sydney-1.ocs.oraclecloud.com` |
| UK (London) | `uk-london-1` | `https://{tenant}.hospitality-api.uk-london-1.ocs.oraclecloud.com` |
| Canada (Toronto) | `ca-toronto-1` | `https://{tenant}.hospitality-api.ca-toronto-1.ocs.oraclecloud.com` |

**Nota:** Consultar con Oracle las regiones exactas disponibles para tu cuenta, ya que pueden variar.

---

## Información Necesaria por Región

Para cada región donde tengas hoteles, necesitas:

| Dato | Descripción |
|------|-------------|
| `region` | Código de región (ej: `us-ashburn-1`) |
| `tenant` | Código de tenant/cadena en esa región |
| `client_id` | Client ID para esa región |
| `client_secret` | Client Secret para esa región |
| `app_key` | Application Key (puede ser la misma o diferente) |
| `username` | Usuario de integración en esa región |
| `password` | Contraseña del usuario |

**Importante:** Las credenciales pueden ser las mismas o diferentes por región, dependiendo de cómo Oracle haya configurado tu cuenta.

---

## Flujo de Extracción Multi-Región

```
┌─────────────────────────────────────────────────────────────────────┐
│  PROCESO DE EXTRACCIÓN MULTI-REGIÓN                                 │
│                                                                      │
│  Para CADA región configurada:                                      │
│    1. AUTENTICACIÓN → Obtener token para esa región                │
│    2. HOTELES → Obtener lista de hotelId en esa región             │
│    3. Para CADA hotelId:                                            │
│       - DISPONIBILIDAD                                              │
│       - RESERVAS                                                    │
│       - GRUPOS                                                      │
│       - PRODUCCIONES                                                │
│       - PERFILES                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Configuración de Regiones

### Archivo de Configuración

```python
# config.py

REGIONES = {
    "us-ashburn-1": {
        "nombre": "US East (Ashburn)",
        "host": "https://mychain.hospitality-api.us-ashburn-1.ocs.oraclecloud.com",
        "client_id": "us_client_id",
        "client_secret": "us_client_secret",
        "app_key": "41ecd082-8997-4c69-af34-2f72b83645ff",
        "username": "integration_user_us",
        "password": "password_us"
    },
    "eu-frankfurt-1": {
        "nombre": "EU Central (Frankfurt)",
        "host": "https://mychain.hospitality-api.eu-frankfurt-1.ocs.oraclecloud.com",
        "client_id": "eu_client_id",
        "client_secret": "eu_client_secret",
        "app_key": "41ecd082-8997-4c69-af34-2f72b83645ff",
        "username": "integration_user_eu",
        "password": "password_eu"
    },
    "ap-tokyo-1": {
        "nombre": "APAC (Tokyo)",
        "host": "https://mychain.hospitality-api.ap-tokyo-1.ocs.oraclecloud.com",
        "client_id": "apac_client_id",
        "client_secret": "apac_client_secret",
        "app_key": "41ecd082-8997-4c69-af34-2f72b83645ff",
        "username": "integration_user_apac",
        "password": "password_apac"
    }
}
```

### Usando Variables de Entorno (Recomendado)

```python
import os

def cargar_configuracion_regiones():
    """
    Carga la configuración de regiones desde variables de entorno.
    """
    regiones = {}

    # Lista de regiones activas (separadas por coma)
    regiones_activas = os.environ.get("OPERA_REGIONES", "").split(",")

    for region in regiones_activas:
        region = region.strip()
        if not region:
            continue

        prefix = f"OPERA_{region.upper().replace('-', '_')}"

        regiones[region] = {
            "nombre": region,
            "host": os.environ.get(f"{prefix}_HOST"),
            "client_id": os.environ.get(f"{prefix}_CLIENT_ID"),
            "client_secret": os.environ.get(f"{prefix}_CLIENT_SECRET"),
            "app_key": os.environ.get(f"{prefix}_APP_KEY"),
            "username": os.environ.get(f"{prefix}_USERNAME"),
            "password": os.environ.get(f"{prefix}_PASSWORD")
        }

    return regiones

# Variables de entorno ejemplo:
# OPERA_REGIONES=us-ashburn-1,eu-frankfurt-1,ap-tokyo-1
# OPERA_US_ASHBURN_1_HOST=https://mychain.hospitality-api.us-ashburn-1.ocs.oraclecloud.com
# OPERA_US_ASHBURN_1_CLIENT_ID=...
# etc.
```

---

## Código de Extracción Multi-Región

```python
import requests
import base64
import time
from datetime import datetime

class OperaCloudClient:
    """Cliente para una región específica de Opera Cloud."""

    def __init__(self, config, region_name):
        self.region = region_name
        self.host = config["host"]
        self.client_id = config["client_id"]
        self.client_secret = config["client_secret"]
        self.app_key = config["app_key"]
        self.username = config["username"]
        self.password = config["password"]
        self.token = None
        self.token_expiry = 0

    def authenticate(self):
        """Obtiene token para esta región."""
        if self.token and time.time() < (self.token_expiry - 300):
            return self.token

        credentials = f"{self.client_id}:{self.client_secret}"
        credentials_b64 = base64.b64encode(credentials.encode()).decode()

        response = requests.post(
            f"{self.host}/oauth/v1/tokens",
            headers={
                "Authorization": f"Basic {credentials_b64}",
                "x-app-key": self.app_key,
                "Content-Type": "application/x-www-form-urlencoded"
            },
            data={
                "grant_type": "password",
                "username": self.username,
                "password": self.password
            }
        )

        if response.status_code != 200:
            raise Exception(f"Auth failed for region {self.region}: {response.status_code}")

        data = response.json()
        self.token = data["access_token"]
        self.token_expiry = time.time() + data.get("expires_in", 3600)
        return self.token

    def get_hoteles_activos(self):
        """Obtiene hoteles activos en esta región."""
        response = requests.get(
            f"{self.host}/ent/config/v1/hotels",
            headers={
                "Authorization": f"Bearer {self.authenticate()}",
                "x-app-key": self.app_key,
                "Content-Type": "application/json"
            }
        )

        if response.status_code == 204:
            return []

        if response.status_code != 200:
            raise Exception(f"Error obteniendo hoteles en {self.region}: {response.status_code}")

        data = response.json()
        hoteles = data.get('hotelSummaryInfoList', [])

        hoy = datetime.now().date().isoformat()
        activos = [
            h['hotelId'] for h in hoteles
            if h.get('inactiveDate') is None or h['inactiveDate'] > hoy
        ]

        return activos


def proceso_extraccion_multiregion(regiones_config):
    """
    Proceso principal que extrae datos de todas las regiones.

    Args:
        regiones_config: Diccionario con configuración por región
    """
    resultados = {}

    for region_code, config in regiones_config.items():
        print(f"\n{'='*60}")
        print(f"PROCESANDO REGIÓN: {region_code}")
        print(f"{'='*60}")

        try:
            # Crear cliente para esta región
            client = OperaCloudClient(config, region_code)

            # Autenticar
            client.authenticate()
            print(f"✓ Autenticación exitosa")

            # Obtener hoteles de esta región
            hotel_ids = client.get_hoteles_activos()
            print(f"✓ Hoteles activos en {region_code}: {hotel_ids}")

            # Procesar cada hotel
            for hotel_id in hotel_ids:
                print(f"\n--- {region_code} / {hotel_id} ---")

                extraer_disponibilidad(client, hotel_id)
                extraer_reservas(client, hotel_id)
                extraer_grupos(client, hotel_id)
                extraer_producciones(client, hotel_id)
                enriquecer_perfiles(client, hotel_id)

            resultados[region_code] = {
                "status": "OK",
                "hoteles": hotel_ids
            }

        except Exception as e:
            print(f"✗ Error en región {region_code}: {str(e)}")
            resultados[region_code] = {
                "status": "ERROR",
                "error": str(e)
            }

    # Resumen final
    print(f"\n{'='*60}")
    print("RESUMEN DE EXTRACCIÓN")
    print(f"{'='*60}")
    for region, resultado in resultados.items():
        if resultado["status"] == "OK":
            print(f"✓ {region}: {len(resultado['hoteles'])} hoteles procesados")
        else:
            print(f"✗ {region}: ERROR - {resultado['error']}")

    return resultados


# Ejecutar
if __name__ == "__main__":
    from config import REGIONES
    proceso_extraccion_multiregion(REGIONES)
```

---

## Ejecución en Paralelo por Región

Para cadenas grandes, puedes procesar regiones en paralelo:

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def procesar_region(region_code, config):
    """Procesa una región completa."""
    try:
        client = OperaCloudClient(config, region_code)
        client.authenticate()

        hotel_ids = client.get_hoteles_activos()

        for hotel_id in hotel_ids:
            extraer_disponibilidad(client, hotel_id)
            extraer_reservas(client, hotel_id)
            extraer_grupos(client, hotel_id)
            extraer_producciones(client, hotel_id)
            enriquecer_perfiles(client, hotel_id)

        return {
            "region": region_code,
            "status": "OK",
            "hoteles": len(hotel_ids)
        }

    except Exception as e:
        return {
            "region": region_code,
            "status": "ERROR",
            "error": str(e)
        }


def proceso_extraccion_paralelo(regiones_config):
    """
    Procesa todas las regiones en paralelo.
    Cada región se procesa en un thread separado.
    """
    resultados = []

    # Máximo 3 regiones en paralelo
    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = {
            executor.submit(procesar_region, region, config): region
            for region, config in regiones_config.items()
        }

        for future in as_completed(futures):
            resultado = future.result()
            resultados.append(resultado)
            print(f"Completado: {resultado['region']} - {resultado['status']}")

    return resultados
```

---

## Consideraciones Multi-Región

### 1. Base de Datos

Incluir siempre la región en tus tablas para diferenciar datos:

```sql
-- Opción A: Columna de región
CREATE TABLE disponibilidad (
    id_unico VARCHAR(100) PRIMARY KEY,  -- incluye región
    region VARCHAR(20) NOT NULL,
    hotel_id VARCHAR(20) NOT NULL,
    fecha DATE NOT NULL,
    tipo_hab VARCHAR(20) NOT NULL,
    ...
);

-- id_unico = "eu-frankfurt-1-HOTEL1-2026-01-01-DBL"
```

### 2. Zona Horaria

Cada región tiene diferente zona horaria. Considera:
- Ejecutar el proceso diario en horarios apropiados para cada región
- Almacenar fechas en UTC y convertir para visualización

```python
# Ejemplo: ejecutar después del night audit de cada región
HORARIOS_EJECUCION = {
    "us-ashburn-1": "06:00 EST",      # Night audit ~04:00 EST
    "eu-frankfurt-1": "06:00 CET",    # Night audit ~04:00 CET
    "ap-tokyo-1": "06:00 JST"         # Night audit ~04:00 JST
}
```

### 3. Rate Limiting

Cada región tiene sus propios límites. Si recibes 429 (Too Many Requests):

```python
def llamar_api_con_retry(client, endpoint, max_retries=4):
    """Llamada con backoff exponencial."""
    for intento in range(max_retries):
        response = client.get(endpoint)

        if response.status_code == 429:
            wait_time = 2 ** intento  # 1, 2, 4, 8 segundos
            print(f"Rate limit en {client.region}. Esperando {wait_time}s...")
            time.sleep(wait_time)
            continue

        return response

    raise Exception(f"Max retries alcanzado para {endpoint}")
```

### 4. Monitoreo

Implementa monitoreo por región:

```python
def log_metrica(region, hotel_id, proceso, duracion, registros):
    """Registra métricas de extracción."""
    print(f"[METRIC] region={region} hotel={hotel_id} "
          f"proceso={proceso} duracion_ms={duracion} registros={registros}")
```

---

## Preguntas para Oracle

Antes de implementar, confirmar con Oracle:

1. **¿En qué regiones están desplegados tus hoteles?**
   - Solicitar lista exacta de regiones con hoteles

2. **¿Las credenciales son las mismas o diferentes por región?**
   - Algunas cadenas tienen un set de credenciales global
   - Otras tienen credenciales por región

3. **¿El tenant code es el mismo en todas las regiones?**
   - Normalmente sí, pero confirmar

4. **¿Hay algún endpoint central que liste todas las regiones?**
   - Actualmente no existe - debes conocer tus regiones de antemano

---

## Ejemplo de Configuración Real

```python
# Para una cadena con hoteles en España, USA y Japón

REGIONES = {
    "eu-frankfurt-1": {
        "nombre": "Europa (Frankfurt)",
        "host": "https://mychain.hospitality-api.eu-frankfurt-1.ocs.oraclecloud.com",
        "client_id": os.environ["OPERA_EU_CLIENT_ID"],
        "client_secret": os.environ["OPERA_EU_CLIENT_SECRET"],
        "app_key": os.environ["OPERA_APP_KEY"],
        "username": os.environ["OPERA_EU_USERNAME"],
        "password": os.environ["OPERA_EU_PASSWORD"],
        "hoteles_esperados": ["HOTEL_MADRID", "HOTEL_BARCELONA", "HOTEL_SEVILLA"]
    },
    "us-ashburn-1": {
        "nombre": "US East (Ashburn)",
        "host": "https://mychain.hospitality-api.us-ashburn-1.ocs.oraclecloud.com",
        "client_id": os.environ["OPERA_US_CLIENT_ID"],
        "client_secret": os.environ["OPERA_US_CLIENT_SECRET"],
        "app_key": os.environ["OPERA_APP_KEY"],
        "username": os.environ["OPERA_US_USERNAME"],
        "password": os.environ["OPERA_US_PASSWORD"],
        "hoteles_esperados": ["HOTEL_NYC", "HOTEL_MIAMI"]
    },
    "ap-tokyo-1": {
        "nombre": "APAC (Tokyo)",
        "host": "https://mychain.hospitality-api.ap-tokyo-1.ocs.oraclecloud.com",
        "client_id": os.environ["OPERA_APAC_CLIENT_ID"],
        "client_secret": os.environ["OPERA_APAC_CLIENT_SECRET"],
        "app_key": os.environ["OPERA_APP_KEY"],
        "username": os.environ["OPERA_APAC_USERNAME"],
        "password": os.environ["OPERA_APAC_PASSWORD"],
        "hoteles_esperados": ["HOTEL_TOKYO", "HOTEL_OSAKA"]
    }
}
```

---

## Notas Importantes

1. **No hay endpoint global**: No existe un endpoint que devuelva hoteles de todas las regiones. Debes iterar por cada región.

2. **Tokens son por región**: Un token obtenido en `us-ashburn-1` NO sirve para `eu-frankfurt-1`.

3. **Datos aislados**: Los datos de cada región están completamente aislados. Un hotel en Frankfurt no aparece en la API de Ashburn.

4. **Misma cadena, múltiples despliegues**: Aunque sea la misma cadena hotelera, Oracle despliega los datos en la región más cercana al hotel.

5. **Confirmar con Oracle**: Antes de implementar, solicita a Oracle la lista exacta de regiones donde tienes hoteles y las credenciales correspondientes.
