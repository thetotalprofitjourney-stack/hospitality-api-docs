# Opera Cloud Data Extractor

Extracción automatizada de datos desde Oracle Hospitality Opera Cloud hacia nuestra base de datos.

## Objetivo

Sincronizar diariamente los datos de nuestros hoteles desde Opera Cloud:
- Disponibilidad de habitaciones
- Reservas
- Grupos/Bloques
- Producciones (ingresos por outlet)
- Perfiles de huéspedes

## Estructura del Proyecto

```
opera-extractor/
│
├── documentacion/           # Especificaciones de la API Opera Cloud
│   ├── AUTENTICACION_API.md
│   ├── REGIONES_API.md
│   ├── HOTELES_API.md
│   ├── DISPONIBILIDAD_API.md
│   ├── RESERVAS_API.md
│   ├── PERFILES_HUESPED_API.md
│   ├── GRUPOS_API.md
│   └── PRODUCCIONES_API.md
│
├── src/                     # Código de implementación
│   ├── config/              # Configuración y credenciales
│   ├── extractors/          # Lógica de extracción por entidad
│   ├── database/            # Modelos y conexión a BD
│   └── main.py              # Orquestador principal
│
├── sql/                     # Scripts de creación de tablas
│
├── tests/                   # Tests unitarios
│
└── README.md
```

## Flujo de Extracción

```
1. Autenticación (OAuth2)
2. Obtener lista de hoteles activos
3. Para cada hotel:
   ├── Disponibilidad (3 días atrás → 31 dic próximo año)
   ├── Reservas (4 días atrás)
   ├── Grupos (activos y futuros)
   ├── Producciones (4 días atrás)
   └── Perfiles (enriquecer reservas sin fecha nacimiento)
```

## Configuración

### Variables de Entorno

```bash
# Región única
OPERA_HOST=https://tenant.hospitality-api.eu-frankfurt-1.ocs.oraclecloud.com
OPERA_CLIENT_ID=your_client_id
OPERA_CLIENT_SECRET=your_client_secret
OPERA_APP_KEY=your_app_key
OPERA_USERNAME=integration_user
OPERA_PASSWORD=integration_password

# Base de datos
DB_HOST=localhost
DB_PORT=5432
DB_NAME=opera_data
DB_USER=postgres
DB_PASSWORD=password
```

### Multi-Región (si aplica)

Ver `documentacion/REGIONES_API.md` para configuración con múltiples regiones.

## Instalación

```bash
# Clonar repositorio
git clone https://github.com/tu-usuario/opera-extractor.git
cd opera-extractor

# Crear entorno virtual
python -m venv venv
source venv/bin/activate  # Linux/Mac
# o: venv\Scripts\activate  # Windows

# Instalar dependencias
pip install -r requirements.txt

# Configurar variables de entorno
cp .env.example .env
# Editar .env con tus credenciales
```

## Uso

```bash
# Ejecutar extracción completa
python src/main.py

# Ejecutar solo un proceso
python src/main.py --proceso disponibilidad
python src/main.py --proceso reservas

# Carga histórica inicial
python src/main.py --historico
```

## Documentación

| Documento | Descripción |
|-----------|-------------|
| [AUTENTICACION_API.md](documentacion/AUTENTICACION_API.md) | OAuth2, tokens, renovación |
| [REGIONES_API.md](documentacion/REGIONES_API.md) | Multi-región para cadenas globales |
| [HOTELES_API.md](documentacion/HOTELES_API.md) | Obtener lista de hoteles |
| [DISPONIBILIDAD_API.md](documentacion/DISPONIBILIDAD_API.md) | Inventario de habitaciones |
| [RESERVAS_API.md](documentacion/RESERVAS_API.md) | Reservas con resumen diario |
| [PERFILES_HUESPED_API.md](documentacion/PERFILES_HUESPED_API.md) | Datos de huéspedes |
| [GRUPOS_API.md](documentacion/GRUPOS_API.md) | Bloques y grupos |
| [PRODUCCIONES_API.md](documentacion/PRODUCCIONES_API.md) | Ingresos por outlet |

## Estado del Proyecto

- [x] Documentación de APIs
- [ ] Configuración de base de datos
- [ ] Implementación de extractores
- [ ] Tests
- [ ] Despliegue

## Requisitos

- Python 3.9+
- PostgreSQL 13+ (o tu BD preferida)
- Credenciales de Opera Cloud API
