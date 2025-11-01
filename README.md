
# FactuMarket — Sistema de Microservicios para Facturación Electrónica

> Repositorio entregable (skeleton) con implementación parcial en .NET (Clientes) y Ruby on Rails (Facturas y Auditoría).  
> Incluye diseño arquitectónico (teórico), código de ejemplo, scripts de base de datos, docker-compose y pipelines CI/CD de ejemplo.

---

## Índice

1. [Contexto y objetivo](#contexto-y-objetivo)  
2. [Parte 1 — Diseño de arquitectura (teórico-práctico)](#parte-1---diseño-de-arquitectura-te%C3%B3rico-pr%C3%A1ctico)  
   1. Definición de microservicios principales  
   2. Responsabilidades e interacción entre microservicios  
   3. Comunicación: síncrona vs asíncrona y cómo garantizar consistencia  
   4. Estrategia de persistencia (Oracle / NoSQL)  
   5. Principios aplicados: Microservicios, Clean Architecture, MVC  
     
3. [Lista de software requerido y versiones recomendadas](#lista-de-software-requerido-y-versiones-recomendadas)  
4. [Instalación completa en Windows 11 vía WSL (Ubuntu 22.04) — pasos sin omisiones](#instalaci%C3%B3n-completa-en-windows-11-v%C3%ADa-wsl-ubuntu-2204---pasos-sin-omisiones)  
5. [Arrancar el proyecto en local (comandos)](#arrancar-el-proyecto-en-local-comandos)  
6. [Parte 2 — Implementación (endpoints, validaciones, persistencia y pruebas)](#parte-2---implementaci%C3%B3n-endpoints-validaciones-persistencia-y-pruebas)  
7. [Scripts de base de datos y inicialización](#scripts-de-base-de-datos-y-inicializaci%C3%B3n)  
8. [CI/CD y pruebas automáticas](#cicd-y-pruebas-autom%C3%A1ticas)  
9. [Consideraciones operacionales y de seguridad](#consideraciones-operacionales-y-de-seguridad)  
10. [Mejoras futuras sugeridas](#mejoras-futuras-sugeridas)  


---

## Contexto y objetivo

**FactuMarket S.A.** necesita modernizar su facturación electrónica. El objetivo es diseñar e implementar un sistema basado en microservicios que permita: registro/gestión de clientes, emisión de facturas electrónicas con validaciones, almacenamiento transaccional en Oracle, registro de eventos de auditoría en NoSQL, y futura integración con la entidad tributaria (DIAN).

---

## Parte 1 - Diseño de arquitectura (teórico-práctico)

A continuación se reproducen y amplían los seis puntos solicitados en la Parte 1. Cada sección incluye la explicación y la justificación técnica.

### 1) Definición de microservicios principales

Se proponen los siguientes microservicios mínimos:

- **Clientes Service**  
  - Responsabilidad: CRUD de clientes (nombre/razón social, identificación, correo, dirección, estado fiscal).
  - Endpoints (mínimos): `POST /api/v1/clientes`, `GET /api/v1/clientes/:id`, `GET /api/v1/clientes`.
  - Persistencia: Oracle (datos transaccionales).
  - Publica eventos de negocio (`cliente.creado`, `cliente.actualizado`) a la cola de mensajes.

- **Facturas Service**  
  - Responsabilidad: creación, validación y consulta de facturas electrónicas; aplicar reglas fiscales; persistir facturas y líneas.
  - Endpoints (mínimos): `POST /api/v1/facturas`, `GET /api/v1/facturas/:id`, `GET /api/v1/facturas?fechaInicio=&fechaFin=`.
  - Persistencia: Oracle (facturas y líneas).
  - Publica eventos (`factura.emitida`) y envía mensajes a Auditoría.

- **Auditoría Service**  
  - Responsabilidad: registrar eventos inmutables del sistema (creación, consultas, errores).
  - Endpoints (mínimos): `GET /api/v1/auditoria/:facturaId`.
  - Persistencia: NoSQL (MongoDB).

**Justificación:** separación por responsabilidad facilita escalabilidad, despliegue independiente y cumplimiento de requisitos (auditoría en NoSQL, transaccionalidad en Oracle).

---

### 2) Responsabilidad de cada servicio y cómo interactúan

**Clientes Service**  
- Provee API REST para gestión de clientes.  
- Al crear/actualizar cliente: persiste en Oracle y publica evento a la cola.  
- Puede exponer búsquedas y consultas por campos fiscales.

**Facturas Service**  
- Orquesta el proceso de emisión:
  1. Valida request y reglas de negocio (cliente, montos, fecha).
  2. Verifica existencia de cliente (consulta síncrona a Clientes Service o cache local).
  3. Persiste factura y líneas en Oracle (transacción local).
  4. Publica evento `factura.emitida` a la cola y registra evento de auditoría.
- Mantiene su propia persistencia para facturas y trabaja con repositorios/adapter.

**Auditoría Service**  
- Suscriptor de la cola que recibe eventos y los inserta en MongoDB con metadata (`correlation_id`, timestamp, servicio origen).
- Ofrece endpoint para consultar trazabilidad por `facturaId` o `correlation_id`.

**Justificación:** Desacoplamiento mediante mensajes reduce acoplamiento temporal y permite escalado diferenciado.

---

### 3) Flujo de comunicación: síncrona vs asíncrona y consistencia

**Comunicación mixta (recomendado):**
- **Síncrona (REST)**: usar para validaciones puntuales y lecturas que requieren valor actual (ej., Facturas valida existencia del cliente mediante `GET /clientes/:id`).
- **Asíncrona (mensajería con RabbitMQ o Kafka)**: publicar eventos de negocio (`cliente.creado`, `factura.emitida`) que consumirá Auditoría y otros servicios (analytics, DIAN integration).  
  - RabbitMQ: fácil de operar, buenas garantías de entrega y patterns de routing.
  - Kafka: útil si se requiere retención prolongada y alto throughput.

**Garantizar consistencia entre servicios:**
- **Consistencia eventual**: aceptada como modelo.  
- **Idempotencia**: endpoints y consumidores deben ser idempotentes (usar `correlation_id`).
- **Sagas / Orquestación**: para procesos que involucren múltiples servicios, usar patrón Saga (coreografía o orquestador) para manejar commit/compensaciones.
- **Publicar eventos sólo después de commit local**: evita inconsistencias si el servicio falla antes de persistir.
- **Retries y DLQ**: configurar reintentos y cola de mensajes muertos para mensajes que fallen repetidamente.

**Justificación:** patrón mixto aprovecha lo mejor de ambos mundos: coherencia puntual y desacoplamiento para side-effects.

---

### 4) Estrategia de persistencia

**Oracle (transaccional, ACID)** — almacenar:
- `clientes` (datos fiscales, identificación, dirección).
- `facturas` (cabecera: número, cliente_id, fecha, totales, estado).
- `lineas_factura` (detalle de cada ítem).
- Otros catálogos con integridad referencial.

**MongoDB (NoSQL) — auditoría / eventos**:
- Almacenar eventos inmutables: `{ correlation_id, servicio_origen, evento, payload, timestamp }`.
- Indexar por `data.factura_id`, `correlation_id` y `timestamp` para consultas rápidas.

**Justificación:** Oracle para integridad y reportes financieros; Mongo para escritura intensiva y flexibilidad del esquema.

---

### 5) Aplicación de los principios

**Microservicios**
- Independencia y despliegue autónomo: cada servicio con su repositorio, pipeline y base de datos.
- Escalabilidad por servicio: Facturas y Auditoría se escalan según demanda.

**Clean Architecture**
- **Dominio**: entidades puras (PORO en Ruby, POCO en .NET) con reglas de negocio.
- **Use Cases (Application)**: orquestan acciones del dominio y definen flujos (ej. `EmitirFactura`).
- **Adapters/Ports (Infra)**: repositorios, clientes HTTP, publicadores de mensajería implementan interfaces.
- **Controllers (UI/Exposición)**: traducen HTTP -> DTO -> Use Case.

Regla: dependencias apuntan hacia adentro; dominio no depende de frameworks.

**MVC**
- Controllers delgados (Rails / ASP.NET Controllers) para manejo de requests.
- Models de dominio separados de modelos de persistencia (ActiveRecord actúa como adapter).
- Views: JSON serializers (ActiveModel::Serializer / JsonResult).

**Justificación:** mejora testabilidad y mantenibilidad.

---



---

## Lista de software requerido y versiones recomendadas

A continuación un listado exhaustivo del software necesario para reproducir el entorno de desarrollo/producción mínimo.

> Las versiones indicadas son recomendadas y estables al momento de esta documentación (elige las más recientes compatibles si es necesario).

- **Sistema operativo (WSL)**  
  - Windows 11 (actualizado) con WSL2 y distribución: **Ubuntu 22.04 LTS**.

- **Lenguajes / Runtimes / Frameworks**  
  - **Ruby**: 3.2.x (recomendado 3.2.2 o 3.2.4).  
  - **Ruby on Rails**: 7.0.x (ej. 7.0.4).  
  - **Node.js**: 18.x (para assets & webpack/yarn compat).  
  - **Yarn**: 1.22.x (o npm si prefieres).  
  - **.NET SDK**: **7.0.x** (para el microservicio Clientes en C#).
  - **Bundler**: 2.4.x (según versión de Ruby).

- **Bases de datos / Mensajería / Otros**  
  - **Oracle Database XE**: 21c XE (imagen `gvenzl/oracle-xe:21-slim` usada en docker-compose).  
  - **MongoDB**: 6.x.  
  - **RabbitMQ**: 3.9+ con plugin management (`rabbitmq:3-management`).  
  - **Docker**: 24.x (Docker Engine).  
  - **Docker Compose**: v2 (integrado en Docker Desktop) o `docker-compose` v1.29+ compatible.  
  - **Oracle Instant Client**: versión compatible con Oracle XE (ej. 21.x) — requerido solo si quieres conectar desde WSL sin usar contenedor.  
  - **git**: 2.34+.

- **Gemas / Paquetes / NuGet (ejemplos usados en el skeleton)**  
  - Ruby gems: `activerecord-oracle_enhanced-adapter` (para Oracle), `ruby-oci8`, `bunny` (RabbitMQ), `mongo` (Ruby driver), `active_model_serializers`, `rspec-rails`.  
  - .NET packages: `Dapper`, `Oracle.ManagedDataAccess.Core`, `Swashbuckle.AspNetCore` (Swagger), `xUnit` para tests.

- **Herramientas opcionales**  
  - **Postman** o **HTTPie** para probar APIs.  
  - **Compass / Studio 3T** para MongoDB.  
  - **Oracle SQL Developer** para administrar Oracle (opcional).  
  - **OpenTelemetry** (para tracing).

---

## Instalación completa en Windows 11 vía WSL (Ubuntu 22.04) — pasos sin omisiones

A continuación el procedimiento **detallado y paso a paso** para instalar todo el software necesario **en Windows 11**, asumiendo que el usuario instalará WSL2 y trabajará dentro de la terminal de Ubuntu 22.04. Cada comando va en una línea nueva y debe ejecutarse en la terminal WSL (Ubuntu).

> Requisitos previos en Windows: cuenta con permisos de administrador para activar características del sistema (Hyper-V y WSL). Ejecuta PowerShell como administrador cuando se indique.

### 0) Preparativos en Windows (PowerShell como administrador)

1. Habilitar WSL y Virtual Machine Platform:
```powershell
wsl --install -d Ubuntu-22.04
```
> Si ya tienes WSL instalado y quieres actualizar a WSL2:
```powershell
wsl --update
wsl --set-default-version 2
```

2. Reinicia Windows si el instalador lo solicita. Después abre Ubuntu 22.04 desde el menú inicio (esto te mostrará la shell de WSL y te pedirá crear usuario).

---

### 1) Actualizar Ubuntu y herramientas básicas (en WSL Ubuntu 22.04)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential curl file git unzip ca-certificates gnupg lsb-release apt-transport-https software-properties-common
```

---

### 2) Instalar Docker Engine y Docker Compose (en WSL)

Sigue estos comandos para instalar Docker Engine en Ubuntu (usando repositorio oficial):

```bash
# Añadir repositorio Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
# Añadir tu usuario al grupo docker (cierra sesión o reinicia la shell después)
sudo usermod -aG docker $USER
```

> Nota: En Windows 11 es habitual utilizar Docker Desktop que integra WSL. Si usas Docker Desktop, asegúrate de activar integración con la distro Ubuntu-22.04 en la configuración de Docker Desktop en Windows. Si usas Docker Desktop, no necesitas instalar Docker Engine dentro de WSL, pero estos pasos son para instalarlo nativo en la distro si prefieres.

Instalar Docker Compose (plugin incluido con Docker Desktop / Docker Engine moderno). Si necesitas `docker-compose` CLI:

```bash
sudo apt install -y docker-compose
```

---

### 3) Instalar .NET 7 SDK (en WSL)

Agregar repositorio Microsoft y el SDK:

```bash
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt update
sudo apt install -y dotnet-sdk-7.0
```

Verificar:
```bash
dotnet --version
# debe mostrar 7.0.x
```

---

### 4) Instalar rbenv, Ruby y Rails (en WSL)

Instalaremos `rbenv` para manejar versiones ruby:

```bash
# Instalar dependencias para compilar Ruby
sudo apt install -y libssl-dev libreadline-dev zlib1g-dev libsqlite3-dev libxml2-dev libcurl4-openssl-dev build-essential

# Instalar rbenv y ruby-build
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
export PATH="$HOME/.rbenv/bin:$PATH"

# Instalar la versión recomendada de Ruby (3.2.x)
rbenv install 3.2.2
rbenv global 3.2.2

# Verificar
ruby -v
```

Instalar Bundler y Rails:

```bash
gem install bundler -v 2.4.13
gem install rails -v 7.0.4
rbenv rehash
rails -v
```

Instalar Node.js 18.x y Yarn (para assets):

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
npm install --global yarn
node -v
yarn -v
```

---

### 5) Instalar dependencias para Oracle (opcional pero necesario si las apps Rails se conectan directamente a Oracle desde WSL)

> Si todo se ejecutará mediante contenedores y las aplicaciones no requieren conectar desde WSL directamente a Oracle, puedes saltar esta sección. Sin embargo, si deseas que Ruby (gem `ruby-oci8`) o herramientas locales se conecten a Oracle desde WSL, deberás instalar **Oracle Instant Client**.

#### Instalación (resumen):
1. Descarga desde Oracle (sitio oficial) los paquetes **Oracle Instant Client Basic** y **SDK** para Linux x86_64 (versión 21.x).
2. Copia los `.zip` o `.deb` al WSL (por ejemplo `~/downloads`).
3. Si son ZIP:
```bash
sudo mkdir -p /opt/oracle
sudo unzip instantclient-basic-linux.x64-21*.zip -d /opt/oracle
sudo unzip instantclient-sdk-linux.x64-21*.zip -d /opt/oracle
# Crear enlaces
sudo ln -s /opt/oracle/instantclient_21_*/libclntsh.so /usr/lib/libclntsh.so
export LD_LIBRARY_PATH=/opt/oracle/instantclient_21_*/:$LD_LIBRARY_PATH
```
4. Si tienes `.deb`, instala con `sudo dpkg -i oracle-instantclient-basic*.deb` y `sudo dpkg -i oracle-instantclient-devel*.deb`.

5. Verifica con `ldconfig -p | grep libclntsh` o probando `ruby -r oci8 -e 'puts OCI8::VERSION'` después de instalar `gem install ruby-oci8`.

> **Nota**: Los paquetes de Oracle requieren aceptar términos del proveedor; no se pueden descargar automáticamente sin cuenta en algunos casos. Usar Docker Oracle XE simplifica la necesidad de instalar instant client para desarrollo.

---

### 6) Instalar MongoDB y RabbitMQ en Docker (se recomienda usar docker-compose)

No necesitas instalar Mongo ni RabbitMQ en WSL; los levantamos con Docker Compose (siguiente sección).

---

### 7) Clonar el repositorio y preparar entorno (WSL)

```bash
cd ~
git clone <TU_REPO_URL> factumarket_repo
cd factumarket_repo/infra
```

---

## Arrancar el proyecto en local (comandos)

En este repositorio el objetivo es ejecutar todo con `docker-compose up`. A continuación los pasos exactos y comandos para dejar el sistema funcionando desde WSL.

### 1) Construir y levantar contenedores

```bash
# Desde la carpeta infra (donde está docker-compose.yml)
docker-compose up --build -d
```

Esto levantará:
- rabbitmq (management en 15672)
- mongo (27017)
- oracle (1521) — imagen `gvenzl/oracle-xe:21-slim`
- clientes_dotnet (puerto 3001)
- facturas_service (puerto 3002)
- auditoria_service (puerto 3003)

### 2) Inicializar bases

**Oracle** (ejecutar script SQL dentro del contenedor Oracle):
```bash
# Copia el script al contenedor y ejecútalo (ejemplo)
docker cp ../db-scripts/oracle/init.sql oracle:/init.sql
# Ejecutar SQL: si la imagen tiene sqlplus o herramienta, usarla; por ejemplo:
docker exec -it <oracle_container_name> bash -c "echo 'run init.sql' && /opt/oracle/product/21c/dbhome_1/bin/sqlplus sys/Oracle@//localhost:1521/ORCLCDB as sysdba @/init.sql"
```
> Nota: Los comandos exactos para ejecutar scripts en Oracle dependen de la imagen; revisa la documentación de la imagen `gvenzl/oracle-xe` para la forma correcta de ejecutar scripts al arranque (puede aceptar `ORACLE_PASSWORD` y montar `docker-entrypoint-initdb.d`).

**MongoDB**:
```bash
# Ejecutar script de índices
docker exec -i $(docker ps -qf "name=mongo") mongo /docker-entrypoint-initdb.d/init.js
# Alternativamente copiar y ejecutar:
docker cp ../db-scripts/mongo/init.js mongo:/init.js
docker exec -it mongo bash -c "mongo /init.js"
```

### 3) Migraciones y seeds (Rails) — si implementaste migraciones

Si tus servicios Rails incluyen migrations (app completa), ejecútalas dentro del contenedor de cada servicio:

```bash
# Facturas service (si está basado en Rails y tiene migraciones)
docker exec -it facturas_service bash -c "bundle install && rails db:create db:migrate db:seed RAILS_ENV=production"
# Auditoria service (si aplica)
docker exec -it auditoria_service bash -c "bundle install && rails db:create db:migrate db:seed RAILS_ENV=production"
```

> Si los contenedores no tienen bundler o ruby instalados (skeleton), adapta comandos según Dockerfile del servicio.

### 4) Ejecutar tests

**.NET tests (clientes):**
```bash
# Ejecutar tests en el contenedor o localmente si tienes .NET SDK instalado
docker exec -it clientes_dotnet bash -c "dotnet test /app/tests || true"
# O local:
cd clientes_dotnet/tests
dotnet test
```

**RSpec (Ruby):**
```bash
# Dentro del contenedor de facturas_service
docker exec -it facturas_service bash -c "bundle install && bundle exec rspec"
```

### 5) Probar endpoints con curl

Registrar cliente (Clientes .NET):
```bash
curl -X POST http://localhost:3001/api/v1/clientes -H "Content-Type: application/json" -d '{"nombre":"ACME S.A.","identificacion":"800123456-7","correo":"facturacion@acme.com","direccion":"Calle 1"}'
```

Crear factura (Facturas):
```bash
curl -X POST http://localhost:3002/api/v1/facturas -H "Content-Type: application/json" -d '{"cliente_id":1,"fecha_emision":"2025-10-31","lineas":[{"descripcion":"Prod A","cantidad":1,"precio_unitario":100}] }'
```

Consultar auditoría:
```bash
curl http://localhost:3003/api/v1/auditoria/123
```

---

## Parte 2 — Implementación (endpoints, validaciones, persistencia, pruebas)

En esta sección se resume lo que ya está implementado en el skeleton y las decisiones técnicas (extraído y ampliado de lo entregado en la Parte 2).

### 1) Servicio de Clientes (.NET)

- **Responsabilidad**: gestión de información de clientes.
- **Endpoints mínimos**:
  - `POST /api/v1/clientes` → registrar cliente (nombre, identificación, correo, dirección).
  - `GET /api/v1/clientes/{id}`
  - `GET /api/v1/clientes`
- **Persistencia**: Oracle (usando `Oracle.ManagedDataAccess.Core` y Dapper en el ejemplo).
- **Auditoría**: cada operación publica evento a RabbitMQ (topic `auditoria.events`).
- **Tests**: ejemplo xUnit en `clientes_dotnet/tests`.

Estructura principal:
```
Clientes (C#)
├─ Controllers/
├─ Models/
├─ Repositories/ (IClienteRepository, OracleClienteRepository)
```

### 2) Servicio de Facturas (Ruby on Rails skeleton)

- **Responsabilidad**: creación y gestión de facturas electrónicas.
- **Endpoints mínimos**:
  - `POST /api/v1/facturas` — validar cliente y emitir factura.
  - `GET /api/v1/facturas/{id}`
  - `GET /api/v1/facturas?fechaInicio=&fechaFin=`
- **Validaciones**:
  - Cliente válido (existe en Clientes Service).
  - Monto > 0.
  - Fecha de emisión válida (formato y valor).
- **Persistencia**: Oracle (ActiveRecord adapter `activerecord-oracle_enhanced-adapter`).
- **Auditoría**: publica evento `factura.emitida` a RabbitMQ; Auditoría Service guarda el evento en MongoDB.
- **Clean Architecture**:
  - `app/domain` — entidades (PORO).
  - `app/use_cases/emitir_factura.rb` — caso de uso.
  - `app/infrastructure` — adaptadores (p. ej. `active_record_factura` y `rabbit_publisher`).
- **Tests**: RSpec en `facturas_service/spec`.

### 3) Servicio de Auditoría (Ruby)

- **Responsabilidad**: almacenar eventos en MongoDB (colección `events`).
- **Endpoint mínimo**:
  - `GET /api/v1/auditoria/:facturaId` — devuelve eventos relacionados.
- **Persistencia**: MongoDB (driver `mongo` gem).
- **Operación**:
  - Un consumer (worker) suscribe a `auditoria.events` en RabbitMQ y persiste cada evento.
  - Indexes por `data.factura_id` y `correlation_id`.

---

## Scripts de base de datos y inicialización

Los scripts están en `db-scripts/`:

- `db-scripts/oracle/init.sql` — crea tablas `clientes`, `facturas`, `lineas_factura`.
- `db-scripts/mongo/init.js` — crea índices en MongoDB para `events`.

Ejecución:
- Oracle: ejecutar `init.sql` dentro del contenedor Oracle (consulta la documentación de la imagen `gvenzl/oracle-xe` para mounts `/docker-entrypoint-initdb.d` o ejecución post-arranque).
- Mongo: se puede montar `init.js` en `/docker-entrypoint-initdb.d` o ejecutar `docker exec mongo mongo /init.js`.

---

## CI/CD y pruebas automáticas

- **Pipeline**: `.github/workflows/ci.yml` incluido — construye .NET y ejecuta tests; intenta ejecutar RSpec si está configurado.
- **Tests**:
  - Unit tests en dominio (xUnit y RSpec).
  - Recomiendo añadir contract tests (Pact) entre Facturas y Clientes.

---

## Consideraciones operacionales y de seguridad

- Autenticación: centralizar con API Gateway y JWT/OAuth2.
- Secret management: usar Vault o secretos de la nube; no incluir credenciales en el repo.
- Observabilidad: instrumentar tracing distribuido y métricas.
- Backups: configurar backups periódicos de Oracle y retención / archivado de auditoría.

---

## Mejoras futuras sugeridas

1. Completar migraciones y modelos ActiveRecord para Facturas.
2. Implementar DLQ y reintentos para consumidor de Auditoría.
3. Añadir contract testing y e2e.
4. Integración con DIAN via adapter con circuit breaker y reintentos.
5. Despliegue en Kubernetes con helm charts y pipelines automatizados.

---


