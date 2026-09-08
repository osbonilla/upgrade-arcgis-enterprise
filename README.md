# Guía de Implementación: ArcGIS Enterprise 12.1, ArcGIS Data Store (Spatiotemporal) y ArcGIS Velocity en Despliegue Híbrido Físico/Virtual

> **Tipo de documento:** Runbook técnico de infraestructura
> **Alcance:** Instalación, configuración y resolución de incidencias de un despliegue de ArcGIS Enterprise 12.1 con Data Store Spatiotemporal y ArcGIS Velocity, replicado en dos entornos paralelos (usuario principal y una compañera).
> **Idioma:** Español
> **Estado del despliegue al cierre de este documento:** Entorno del usuario principal validado y operativo (Data Stores en verde); federación de ArcGIS Velocity en progreso. Entorno de la compañera con los 3 Data Stores validados tras resolver bloqueos de red/antivirus.

---

# Objetivo de la implementación

El objetivo general de este trabajo fue desplegar y dejar operativo un entorno de **ArcGIS Enterprise 12.1** capaz de soportar **ArcGIS Velocity** (la versión self-hosted, on-premises), lo cual requirió, en orden de aparición durante la conversación:

1. Instalar y verificar Portal for ArcGIS en Windows.
2. Gestionar la instalación/reinstalación del ArcGIS Web Adaptor (Portal y, potencialmente, Server).
3. Autorizar el software mediante archivo de licencia (`.ecp`) de Esri.
4. Diagnosticar y resolver un error de federación entre ArcGIS Server y Portal relacionado con la base de datos administrada del Data Store.
5. Determinar la versión más reciente de ArcGIS Enterprise (12.1) y confirmar que es la mínima requerida para instalar ArcGIS Velocity self-hosted.
6. Instalar ArcGIS Velocity en una máquina virtual (VM) separada.
7. Configurar el **ArcGIS Data Store tipo Spatiotemporal (Big Data Store)**, requerido por Velocity para almacenar salidas de feature layers espacio-temporales.
8. Resolver una serie encadenada de problemas de **conectividad de red, DNS, firewall (Windows y antivirus de terceros)** entre una máquina física (host) y una o más máquinas virtuales (VirtualBox), tanto en el entorno del usuario como en el de su compañera de trabajo.
9. Federar ArcGIS Server con Portal, y posteriormente federar ArcGIS Velocity con Portal.
10. Documentar la totalidad del proceso para que otro ingeniero pueda reproducirlo sin necesidad de leer la conversación original.

> **Nota:** La conversación no especifica un caso de uso de negocio final (por ejemplo, qué tipo de sensores o feeds en tiempo real se conectarán a Velocity). Este dato es: *Información no especificada en la conversación.*

---

# Arquitectura general

## Resumen ejecutivo de la arquitectura

Durante la conversación se trabajó, de forma simultánea, con **dos entornos independientes**:

- **Entorno A — Usuario principal** (el interlocutor de este chat).
- **Entorno B — Compañera de trabajo** (mencionada como "mi amiga" / "mi compañera"), con su propio despliegue paralelo, cuya instalación se guio en paralelo aplicando las mismas soluciones encontradas para el Entorno A.

Ambos entornos comparten el mismo patrón arquitectónico: **una máquina física (host) que aloja Portal + ArcGIS Server**, y **una máquina virtual (VirtualBox) que aloja ArcGIS Data Store** (con los tres tipos de almacén: Relacional, Objeto y Espaciotemporal) **y, en el caso del Entorno A, también ArcGIS Velocity**.

### Diagrama de arquitectura — Entorno A (Usuario principal)

```
┌─────────────────────────────────────────────────┐
│  MÁQUINA FÍSICA (Host)                          │
│  Hostname: obonilla.esrinosa.local              │
│  IP (Wi-Fi): 192.168.100.248                    │
│  IP (VirtualBox Host-Only): 192.168.56.1        │
│                                                 │
│  - Portal for ArcGIS 12.1                       │
│  - ArcGIS Server 12.1 (Hosting Server)          │
│  - ArcGIS Web Adaptor                           │
└───────────────────┬─────────────────────────────┘
                    │ Federación (puerto 6443)
                    │ Comunicación BIDIRECCIONAL
                    ▼
┌─────────────────────────────────────────────────┐
│  MÁQUINA VIRTUAL (VirtualBox)                   │
│  Hostname: WIN-OC2B24K34HS                      │
│  IP: 192.168.100.250                            │
│                                                 │
│  - ArcGIS Data Store 12.1                       │
│      • Relational store                         │
│      • Object store                             │
│      • Spatiotemporal big data store            │
│  - ArcGIS Velocity (puerto 7143)                │
└─────────────────────────────────────────────────┘
```

> **Nota sobre buenas prácticas de arquitectura:** Esri recomienda para producción una topología de 3 máquinas (Web GIS Server, Real-Time Server, Big Data Server) y desaconseja combinar más de un tipo de Data Store en una sola máquina por motivos de rendimiento (advertencia mostrada textualmente en el propio asistente de configuración: *"While more than one type of data store can be configured on a single machine, it is not recommended for production systems due to performance considerations."*). En este despliegue, por tratarse de un entorno de laboratorio/pruebas en VirtualBox, se combinaron intencionalmente los tres tipos de Data Store y ArcGIS Velocity en una sola VM.

### Diagrama de arquitectura — Entorno B (Compañera)

```
┌─────────────────────────────────────────────────┐
│  MÁQUINA FÍSICA (Host de la compañera)          │
│  Hostname/Dominio: geoportal.esri.co            │
│  Antivirus: Kaspersky Endpoint Security         │
│  IP observada: 192.168.100.249 (ver nota IP)    │
│                                                 │
│  - Portal for ArcGIS 12.1                       │
│  - ArcGIS Server 12.1 (Hosting Server)          │
└───────────────────┬─────────────────────────────┘
                    │ Federación (puerto 6443)
                    ▼
┌─────────────────────────────────────────────────┐
│  MÁQUINA VIRTUAL (VirtualBox)                   │
│  Hostname: WIN-KQ3HPTQDHP1                      │
│  IP observada: 192.168.100.249 (ver nota IP)    │
│                                                 │
│  - ArcGIS Data Store 12.1                       │
│      • Relational store                         │
│      • Object store                             │
│      • Spatiotemporal big data store            │
└─────────────────────────────────────────────────┘
```

> ⚠️ **Nota sobre inconsistencia de direcciones IP (Entorno B):** En distintos momentos de la conversación, la dirección `192.168.100.249` aparece asociada tanto a `geoportal.esri.co` (máquina física) como, más adelante, a la VM `WIN-KQ3HPTQDHP1`, mientras que en pruebas posteriores la dirección de origen (physical) aparece como `192.168.100.228`. Esto sugiere una posible **reasignación de IP por DHCP** entre distintos momentos de la sesión de trabajo. Esta discrepancia no fue aclarada explícitamente en la conversación — *información no especificada en la conversación.* Se recomienda, al reproducir este runbook, **fijar IPs estáticas o reservas DHCP** para evitar este tipo de confusión (ver sección de Lecciones Aprendidas).

### Entorno de referencia adicional (mencionado, no parte del despliegue principal)

En un punto de la conversación, el usuario compartió una captura de un tercer entorno (descrito como perteneciente a "un ingeniero"), mostrando:

- Dominio de Portal: `geoportal.esri.co` (mismo dominio que luego se confirmó pertenece a la compañera, aunque en ese momento no se sabía)
- Servidor host: `jquinteros.esrinosa.local:6443`
- Servidor ArcGIS Velocity: `win-hm9i9ncuvv9:7143`

Esta captura se usó únicamente para explicar el significado de los nombres de máquina generados automáticamente por Windows (patrón `WIN-XXXXXXXXXX`) y **no se retomó como parte activa del despliegue documentado en este runbook** más allá de esa explicación conceptual.

---

# Infraestructura

| Elemento | Entorno A (Usuario) | Entorno B (Compañera) |
|---|---|---|
| Sistema operativo | Windows (físico) + Windows (VM VirtualBox) | Windows (físico) + Windows (VM VirtualBox) |
| Virtualización | Oracle VirtualBox | Oracle VirtualBox |
| Hostname máquina física | `obonilla.esrinosa.local` | `geoportal.esri.co` |
| Hostname VM (Data Store [+ Velocity en Entorno A]) | `WIN-OC2B24K34HS` | `WIN-KQ3HPTQDHP1` |
| Dominio interno | `esrinosa.local` | No especificado como dominio interno; parece dominio propio (`esri.co`) |
| IP física (adaptador relevante) | `192.168.100.248` (Wi-Fi) | `192.168.100.249` / `192.168.100.228` (ver nota de inconsistencia) |
| IP VM Data Store | `192.168.100.250` | `192.168.100.249` (ver nota de inconsistencia) |
| Rango de red | `192.168.100.0/24` | `192.168.100.0/24` |
| Antivirus / EDR adicional | No reportado (Información no especificada) | **Kaspersky Endpoint Security for Windows** (causa raíz de bloqueos) |
| DNS | Sin servidor DNS interno funcional para estos hostnames; se resolvió vía archivo `hosts` | Igual: resuelto vía archivo `hosts` |
| Certificados / HTTPS | Certificado autofirmado por defecto de ArcGIS (advertencia "Not secure" observada en navegador) | No se detalla explícitamente; se asume el mismo comportamiento por defecto |
| Web Adaptor | Instalado para Portal (contexto de la primera parte de la conversación); gestión de reinstalación discutida | No mencionado explícitamente para este entorno |
| IIS | Implícito como contenedor del Web Adaptor (no se detallaron pasos de configuración de IIS más allá del reinicio con `iisreset`) | Información no especificada en la conversación |
| Reverse Proxy / Balanceadores | No aplican en este despliegue (arquitectura de 2 máquinas, sin balanceo) | No aplican |

## Puertos utilizados (según documentación oficial de Esri, consultada durante la resolución de incidencias)

> Fuente: documentación oficial "Ports used by ArcGIS Data Store" (enterprise.arcgis.com / doc.esri.com). Esta tabla fue clave para resolver el problema de validación del Data Store Espaciotemporal, ya que inicialmente se probó (por error) un puerto que no correspondía.

| Puerto | Protocolo | Componente | Propósito |
|---|---|---|---|
| 443 / 80 | HTTPS/HTTP | Web Adaptor | Acceso externo estándar |
| 7443 | HTTPS | Portal for ArcGIS | Comunicación de Portal (HTTPS forzado por defecto) |
| 7080 | HTTP | Portal for ArcGIS | Deshabilitado por defecto (solo si se permite HTTP) |
| 6443 | HTTPS | ArcGIS Server | Administración del sitio de Server; Data Store envía solicitudes salientes al hosting server por este puerto |
| 2443 | HTTPS | ArcGIS Data Store (todos los tipos) | Comunicación entre máquinas del Data Store y con el Configuration Wizard / hosting server |
| 9006 | TCP | ArcGIS Data Store | Comunicación interna con servidor web (Tomcat); no requiere apertura en firewall, pero debe estar libre en la máquina |
| **9876** | TCP | **Relational store** | Comunicación interna entre hosting server y el **almacén Relacional** (⚠️ no es el puerto del Spatiotemporal — ver "Problemas encontrados") |
| 9840 | TCP | Relational store | Comunicación con caché en memoria del sistema |
| 9820, 9850 | TCP | Relational store | Comunicación entre máquinas del data store |
| 45671, 45672 / 25672, 44369 | TCP | Relational store | Requeridos si se usan webhooks de servicios |
| 50432 | TCP | Relational store | Requerido al actualizar (upgrade) el almacén relacional |
| **9220** | HTTP/HTTPS | **Spatiotemporal big data store** | **Comunicación entre el hosting server (y servidores federados) y el spatiotemporal big data store** ✅ puerto correcto |
| **9320** | TCP | **Spatiotemporal big data store** | **Comunicación interna entre máquinas del clúster spatiotemporal** ✅ puerto correcto |
| 29878/29879 (u otras variantes según versión) | HTTP/HTTPS | Object store | Comunicación del hosting server con el object store |
| 29080, 29081 | HTTP/HTTPS | Tile cache / Scene tile cache data store | Comunicación del tile cache data store |
| 9829 | TCP | Graph store (ArcGIS Knowledge Server) | Comunicación con el graph store (no usado en este despliegue, incluido por completitud) |
| 9828, 9830, 9831 | TCP | Graph store | Comunicación interna de clúster (no usado en este despliegue) |
| **7143** | HTTPS | **ArcGIS Velocity** | Puerto de administración/servicios de Velocity usado al federarlo con Portal |


---

# Componentes instalados

## Portal for ArcGIS

- **Versión:** 12.1 (confirmada por el título de ventana "ArcGIS Data Store 12.1 Setup" visto durante la instalación del Data Store, y por ser la versión mínima requerida para Velocity self-hosted, que fue el objetivo declarado del despliegue).
- **Propósito:** Interfaz web organizativa de ArcGIS Enterprise; gestión de usuarios, contenido, roles y federación de servidores.
- **Ubicación (instalación por defecto en Windows):** `C:\Program Files\ArcGIS\Portal`
- **Subcarpetas relevantes:** `framework`, `tools`, `webapps`, `etc\config`
- **Logs:** `C:\Program Files\ArcGIS\Portal\logs` y log de instalador en `%TEMP%\ArcGISPortal_Install.log` (nombre exacto puede variar según versión).
- **Puerto por defecto:** 7443 (HTTPS), acceso directo sin Web Adaptor vía `https://localhost:7443/arcgis/home`.
- **Servicio de Windows:** "Portal for ArcGIS" (verificable en `services.msc`).
- **Dependencias:** Ninguna previa; es el primer componente en instalarse en una arquitectura base de ArcGIS Enterprise.
- **Orden de instalación:** 1º (primer componente instalado en la máquina física, según el inicio de la conversación).
- **Licenciamiento:** Autorizado mediante archivo `.ecp` (Software Authorization Wizard), usando la opción *"I have received an authorization file from ESRI and am now ready to finish the authorization process."* El archivo se descarga desde my.esri.com → Licensing → My Organizations' Licenses.

## ArcGIS Server (Hosting Server)

- **Versión:** 12.1 (acorde a Portal).
- **Propósito:** Servidor de hospedaje ("Hosting Server") de Portal; ejecuta y publica servicios, incluidos los feature layers hospedados que usan el Data Store.
- **Rol confirmado en Portal:** "Hosting Server", con estado final "All systems operational" en `obonilla.esrinosa.local:6443`.
- **Puerto de administración:** 6443 (HTTPS) — `https://obonilla.esrinosa.local:6443/arcgis/admin` (Server Manager en `https://obonilla.esrinosa.local:6443/arcgis/manager`).
- **Cuenta administrativa:** Primary Site Administrator (PSA) — **cuenta distinta a la de Portal** (`portaladmin` ≠ `siteadmin`), aunque en algunos despliegues pueden compartir el mismo usuario/contraseña por conveniencia.
- **Dependencias:** Requiere Portal instalado (o instalarse en conjunto) para poder federarse.
- **Orden de instalación:** 2º.
- **Federación con Portal:** Confirmada y validada durante la conversación vía Portal → Organización → Configuración → Servidores → `obonilla.esrinosa.local:6443` (Service URL: `https://obonilla.esrinosa.local/server`; Administration URL: `https://obonilla.esrinosa.local:6443/arcgis`).

## ArcGIS Web Adaptor

- **Propósito:** Proxy inverso que expone Portal (y opcionalmente Server) típicamente en el puerto 443 vía IIS, en lugar de los puertos internos 7443/6443.
- **Nota arquitectónica clave:** Portal for ArcGIS funciona de manera **independiente** del Web Adaptor; el Web Adaptor es solo un reverse-proxy, por lo que **se puede verificar que Portal está correctamente instalado sin necesidad de tener el Web Adaptor instalado**.
- **Instalaciones separadas:** El Web Adaptor de Portal y el Web Adaptor de Server son instalaciones independientes, aunque puedan convivir en el mismo servidor IIS.
- **Dependencias:** Requiere que Portal (y/o Server) ya estén instalados y en ejecución.
- **Dato relevante posterior:** Más adelante en la conversación se detectó un error en `obonilla.esrinosa.local/server/manager`: *"Could not access any server machines. Please contact your system administrator."* — ver sección "Problemas encontrados".

## ArcGIS Data Store — Relational store

- **Versión:** 12.1.
- **Propósito:** Almacén relacional (PostgreSQL embebido) para feature layers hospedados estándar.
- **Ubicación:** VM `WIN-OC2B24K34HS` (Entorno A) / VM `WIN-KQ3HPTQDHP1` (Entorno B).
- **Directorio de contenido:** `C:\arcgisdatastore` (ruta usada consistentemente en los comandos `configuredatastore.bat` y en el asistente gráfico).
- **Puerto principal:** 9876 (TCP), además de 9840, 9820 y 9850.
- **Estado final:** Validado (verde) en ambos entornos.

## ArcGIS Data Store — Object store (Administrado Objeto)

- **Versión:** 12.1.
- **Propósito:** Almacenamiento de objetos binarios grandes usados por ciertos tipos de servicios (p. ej. notebooks, ciertos análisis).
- **Ubicación:** Misma VM que el Relational store en ambos entornos.
- **Estado final:** Validado (verde) en ambos entornos.

## ArcGIS Data Store — Spatiotemporal big data store (Espaciotemporal)

- **Versión:** 12.1.
- **Propósito:** Almacén especializado para **datos observacionales** (objetos en movimiento, sensores estacionarios con atributos cambiantes) requerido específicamente para que **ArcGIS Velocity** pueda escribir salidas de feature layers.
- **Ubicación:** Misma VM que Relational y Object store.
- **Puertos correctos:** 9220 (HTTP/HTTPS) y 9320 (TCP) — ver corrección documentada en "Problemas encontrados", ya que inicialmente se intentó diagnosticar con el puerto 9876 (que en realidad pertenece al Relational store).
- **Comando de configuración:** `configuredatastore.bat` con el flag `--stores spatiotemporal`.
- **Instalación de la característica:** Requiere que el instalador de ArcGIS Data Store tenga seleccionada la característica "Spatiotemporal big data" (no viene seleccionada por defecto; en un primer intento fue necesario usar la opción "Modificar" en Agregar o quitar programas, aunque finalmente se optó por instalarlo desde cero en una nueva VM seleccionándolo desde el inicio — ver "Decisiones técnicas").
- **Estado final:** Validado (verde) en ambos entornos, tras resolver bloqueos de firewall a los puertos 9220/9320.

## ArcGIS Velocity (self-hosted / on-premises)

- **Versión mínima de Enterprise requerida:** ArcGIS Enterprise **12.1** (primera versión que soporta Velocity self-hosted; versiones anteriores, 11.x y 10.x, no pueden instalar Velocity Server y solo contaban con ArcGIS GeoEvent Server como alternativa on-premises de tiempo real).
- **Propósito:** Motor de ingestión, feeds y análisis en tiempo real. Reemplazo planeado de ArcGIS GeoEvent Server (ArcGIS Enterprise 12.3, esperado para 2027, será la última versión que incluya GeoEvent Server).
- **Ubicación:** Instalado en la misma VM que el Data Store (`WIN-OC2B24K34HS`), en el Entorno A.
- **Puerto:** 7143 (HTTPS) — usado al federar con Portal (`https://win-oc2b24k34hs:7143/arcgis`).
- **Licenciamiento:** Requiere licencia propia (Standard, Advanced o Dedicated), separada de la licencia base de Portal/Server.
- **Limitación de la versión self-hosted (primera versión):** No incluye Big Data Analytics (solo Real-Time Analytics); Big Data Analytics permanece exclusivo de la versión en ArcGIS Online.
- **Requisitos de hardware (producción):** Mínimo 4 núcleos físicos / 8 lógicos (recomendado 8+ físicos en producción), mínimo 16 GB RAM, mínimo 20 GB de disco solo para instalación (cada feed/analítica consume espacio adicional vía Kafka), red de alto ancho de banda (1 GB o 10 GB). Se menciona que generaciones más nuevas de RAM (DDR5) pueden mejorar el throughput.
- **Requisito de DNS/hostname:** La máquina de Velocity debe poder resolver el hostname configurado durante la creación del sitio de Portal; si el dominio interno no está registrado en DNS público, debe agregarse manualmente al archivo hosts.
- **Estado final documentado en el chat:** Instalación completada; federación con Portal en progreso al cierre de la conversación (bloqueada por un error de conectividad al puerto 7143, en proceso de diagnóstico con el mismo patrón de solución ya aplicado a otros componentes).

---

# Procedimiento completo de instalación

> Esta sección documenta, en orden cronológico, cada paso realizado durante la conversación, incluyendo comandos exactos, rutas, capturas mencionadas y verificaciones.

## Fase 1 — Verificación de Portal for ArcGIS recién instalado

**Objetivo:** confirmar que Portal quedó correctamente instalado antes de manipular el Web Adaptor.

1. **Verificar el servicio de Windows:**
   ```
   services.msc
   ```
   Buscar el servicio **"Portal for ArcGIS"** → debe estar en estado **Running**.

2. **Verificar acceso directo vía HTTPS (sin Web Adaptor)**, dado que Portal corre por defecto en el puerto 7443:
   ```
   https://localhost:7443/arcgis/home
   ```
   o
   ```
   https://<nombre-maquina>:7443/arcgis/home
   ```
   Si aparece la pantalla de bienvenida/login de Portal, la instalación fue exitosa.

3. **Verificar el estado de la configuración inicial** (creación de la cuenta de administrador primario): si no se ha completado, la URL anterior redirige automáticamente a la página de configuración inicial.

4. **Revisar la carpeta de instalación:**
   ```
   C:\Program Files\ArcGIS\Portal
   ```
   Debe contener las carpetas `framework`, `tools`, `webapps`, `etc\config`.

5. **Revisar logs de instalación:**
   ```
   C:\Program Files\ArcGIS\Portal\logs
   ```
   o el log del instalador en `%TEMP%\ArcGISPortal_Install.log` (nombre exacto puede variar según versión).

6. **Verificar vía REST API (opcional, más técnico), una vez configurado:**
   ```
   https://localhost:7443/arcgis/portaladmin/
   ```

**Resultado:** Portal quedó verificado como correctamente instalado, con la configuración inicial (administrador primario) ya completada por el usuario.

## Fase 2 — Autorización del software (licenciamiento)

Se presentó el **Software Authorization Wizard** con tres opciones. Se seleccionó la tercera:

> *"I have received an authorization file from ESRI and am now ready to finish the authorization process."*

**Pasos ejecutados:**

1. Clic en **Browse...**
2. Localizar el archivo de autorización enviado por Esri (extensión `.ecp`, con un nombre similar a `Authorization_XXXXXXXX.ecp` o `Portal_for_ArcGIS_XXXXX.ecp`).
3. Seleccionarlo y hacer clic en **Abrir**.
4. Clic en **Next >**.

**Fuente del archivo `.ecp` (si no se dispone de él):**

1. Ir a `https://my.esri.com` e iniciar sesión con la cuenta organizacional.
2. Ir a **Licensing → My Organizations' Licenses** (o "Licensing Portal").
3. Buscar la licencia de **Portal for ArcGIS**.
4. Descargar el archivo de autorización (`.ecp`).

**Alternativa mencionada (no utilizada finalmente, ver Decisiones Técnicas):** autorización en línea seleccionando la primera opción del wizard (*"I have installed my software and need to authorize it."*) usando el número de autorización directamente, sin necesidad del archivo `.ecp`, si hay conexión a internet disponible.

## Fase 3 — Gestión del Web Adaptor (reinstalación)

**Contexto:** el usuario quería desinstalar el Web Adaptor actual para instalar uno nuevo, y no tenía claro si debía desinstalar también el Web Adaptor de Server.

**Regla aclarada:** el Web Adaptor de Portal y el de Server son instalaciones independientes. Solo se desinstala el que corresponda al componente que se está actualizando/reinstalando.

**Procedimiento general para reemplazar el Web Adaptor de Portal:**

```
1. Panel de Control → Programas y características
2. Desinstalar "ArcGIS Web Adaptor" (el que apunta a Portal)
3. (Opcional pero recomendado) Reiniciar IIS:
   iisreset
4. Instalar el nuevo Web Adaptor
5. Ejecutar la configuración (Configure with Portal/Server):
   https://localhost/<webadaptorname>/webadaptor
6. Verificar en Portal:
   https://<tu-dominio>/arcgis/home
```

> **Advertencia documentada:** si ArcGIS Server ya está federado con Portal a través del Web Adaptor actual, se debe verificar que la URL pública de Portal no cambie (para no romper la federación). Si cambia el nombre del Web Adaptor o el puerto, es necesario re-federar Server con Portal después.

## Fase 4 — Diagnóstico del error de federación (Data Store no validado)

Se detectó, en la pantalla de servidores federados de Portal (`obonilla.esrinosa.local:6443`), el estado **"Issues found"** con el siguiente mensaje textual:

> *"La dirección URL de administración de ArcGIS Server 'https://obonilla.esrinosa.local:6443/arcgis' está accesible. Validando servidor de alojamiento. La versión 'https://obonilla.esrinosa.local:6443/arcgis' de ArcGIS Server coincide con Portal for ArcGIS. Error: Error al validar la base de datos administrada del servidor '/enterpriseDatabases/AGSDataStore_ds_nmpts56z'."*

**Diagnóstico:** ArcGIS Server se federó correctamente con Portal (conexión base funcional), pero falla la validación de la base de datos administrada del Relational Data Store.

**Pasos de diagnóstico aplicados:**

1. Verificar servicio de Data Store: `services.msc` → **"ArcGIS Data Store"** → debe estar **Running**.
2. Verificar si Data Store fue configurado: `https://<servidor-datastore>:2443/arcgis/datastoreadmin/`.
3. Revisar conectividad entre Server y Data Store: `telnet <nombre-datastore> 2443`.
4. Revisar logs de Server en Portal (Organization → Servers → Logs) o en Server Manager (`https://obonilla.esrinosa.local:6443/arcgis/manager`).

## Fase 5 — Instalación de ArcGIS Velocity (self-hosted)

### Prerrequisitos verificados

1. Base de ArcGIS Enterprise 12.1 ya funcionando (Portal + Server federados).
2. Hardware: ≥4 núcleos físicos / 8 lógicos (8+ físicos recomendado en producción), ≥16 GB RAM, ≥20 GB disco solo para instalación, red de alto ancho de banda.
3. DNS/hostname: la máquina de Velocity debe poder resolver el hostname de Portal (si es dominio interno no público, agregar entrada manual en `hosts`).

### Pasos de instalación

1. **Descargar el instalador** desde My Esri (`my.esri.com`), sección de descargas de la licencia de ArcGIS Enterprise 12.1, buscando el instalador de ArcGIS Velocity.
2. **Preparar la máquina:** idealmente separada de Portal/Server (en este despliegue, por ser entorno de pruebas, se instaló en la misma VM que el Data Store).
3. **Ejecutar el setup:**
   ```
   VelocitySetup.exe
   ```
   El asistente solicita: aceptar el acuerdo de licencia, ruta de instalación, ruta para logs y respaldos de configuración.
4. **Configurar Velocity (post-instalación)** vía navegador:
   ```
   https://<maquina-velocity>:7143/velocity
   ```
   Solicitando: federación con el Portal (URL de Portal, credenciales de administrador), configuración del servidor host de Velocity, autorización de la licencia de Velocity (separada de la licencia base de Enterprise).
5. **Licenciamiento:** asignar la licencia de Velocity (Standard/Advanced/Dedicated) a la organización antes de completar la configuración.

## Fase 6 — Configuración del ArcGIS Data Store tipo Spatiotemporal

### Primer intento (en la VM original — posteriormente abandonado)

1. Instalación de ArcGIS Data Store con el tipo **Relational** únicamente seleccionado (por defecto).
2. Ejecución del comando `configuredatastore.bat` con el flag `--stores spatiotemporal`, que arrojó el error:
   > *"El almacén 'spatiotemporal' no está instalado en la configuración actual. En Windows, use la opción Modificar para seleccionar funciones adicionales de Agregar o quitar programas."*
3. Se inició el proceso de **Modificar** la instalación existente (Panel de Control → Programas y características → ArcGIS Data Store → Modificar), en la pantalla "Select Features", para agregar la característica **"Spatiotemporal big data"** (cambiando su estado de ❌ *"This feature will not be available"* a *"This feature will be installed on local hard drive"*), dejando intactas las características ya instaladas (Relational store, Object store).
4. **Este enfoque fue abandonado** antes de completarse — ver "Decisiones técnicas".

### Segundo intento (nueva VM — enfoque finalmente exitoso)

1. Se creó una **nueva máquina virtual** en VirtualBox.
2. Se instaló ArcGIS Data Store 12.1 desde cero, seleccionando **los tres tipos de almacén** (Relational, Object, Spatiotemporal) desde el instalador inicial, evitando así la necesidad de "Modificar" posteriormente.
3. Durante la instalación, se solicitó la pantalla **"Specify ArcGIS Data Store Account"**, donde se definió:
   - **Username:** `arcgis` (usuario de servicio de Windows local a esa VM, sin relación con `portaladmin` ni `siteadmin`).
   - **Password:** definida por el usuario para esa VM específica.
4. Al finalizar la instalación, se accedió al asistente de configuración vía navegador:
   ```
   https://localhost:2443/arcgis/datastore/
   ```
   (Se detectó inicialmente un problema de renderizado — página sin estilos CSS/JS — resuelto probando otro navegador / limpiando caché / aceptando el certificado autofirmado explícitamente).

5. **Paso "Hosting server details":**
   - **Hosting server:** hostname:puerto del ArcGIS Server que actúa como hosting server (ej. `obonilla.esrinosa.local:6443`), **sin** `https://` ni `/arcgis`.
   - **Username:** cuenta del **Primary Site Administrator** de ese ArcGIS Server (no la de Portal).
   - **Password:** contraseña de esa cuenta.

6. **Resolución de errores de conectividad** (documentados en detalle en "Problemas encontrados"), que incluyeron:
   - Resolución DNS vía archivo `hosts`.
   - Apertura de puertos de firewall en ambas direcciones (física ↔ VM).
   - Identificación y neutralización de un antivirus de terceros (Kaspersky) en el Entorno B.
   - Corrección del puerto usado para diagnosticar el Spatiotemporal store (9876 → 9220/9320).

7. **Pasos siguientes del asistente** (una vez resuelta la conectividad):
   - Especificar la ubicación del directorio de contenido: `C:\arcgisdatastore`.
   - Especificar el tipo de almacén a configurar: **Spatiotemporal**.
   - Revisar el resumen de configuración.
   - Crear el data store.

8. **Pantalla de éxito ("Configuration status"):**
   > *"To complete the configuration process, you must now federate the ArcGIS Server site with your Portal... 1. Spatiotemporal... Portal URL: https://obonilla.esrinosa.local/portal"*

9. **Validación final** en Portal (Organización → Configuración → Servidores → [servidor] → Data Stores): tabla con **Relacional**, **Administrado Objeto** y **Espaciotemporal**, cada uno validado individualmente con el botón **"Validar"** (o "Validar todo").

## Fase 7 — Federación de ArcGIS Velocity con Portal

1. En Portal: **Organización → Configuración → Servidores → Agregar sitio de servidor ("Add Server Site")**.
2. Pantalla **"Federate server site"**:
   - **Services URL:** `https://win-oc2b24k34hs:7143/arcgis` (⚠️ se detectó y corrigió un error de tipeo: `htttps` con tres "t" en lugar de `https`).
   - **Administration URL:** mismo valor.
   - **Server credentials — Username/Password:** credenciales del Primary Site Administrator de Velocity.
3. Siguiente paso del asistente: **"Configure server role"**, donde se selecciona el rol **ArcGIS Velocity**.
4. **Estado al cierre de la conversación:** en proceso — bloqueado por error de conectividad al puerto 7143 entre la máquina física (Portal) y la VM (Velocity), con el mismo patrón de diagnóstico (ping, `Test-NetConnection`, reglas de firewall) ya aplicado a los demás componentes, pendiente de confirmación final.

---

# Configuraciones realizadas

## DNS / Resolución de nombres

No existía un servidor DNS interno funcional para resolver los hostnames internos (`obonilla.esrinosa.local`, `geoportal.esri.co` entre máquinas). Se resolvió en ambos entornos mediante **entradas manuales en el archivo `hosts`** de Windows.

**Archivo editado:**
```
C:\Windows\System32\drivers\etc\hosts
```

**Comando para editarlo (requiere permisos de Administrador):**
```cmd
notepad C:\Windows\System32\drivers\etc\hosts
```

**Entradas agregadas:**

| Máquina donde se edita | Línea agregada |
|---|---|
| VM Data Store del usuario (`WIN-OC2B24K34HS`) | `192.168.100.248    obonilla.esrinosa.local` |
| VM Data Store de la compañera (`WIN-KQ3HPTQDHP1`) | `<IP-de-geoportal.esri.co>    geoportal.esri.co` |
| Máquina física del usuario (`obonilla.esrinosa.local`) | `192.168.100.250    WIN-OC2B24K34HS` (agregada tras detectar que el ping en sentido inverso —física → VM— fallaba) |

**Verificación tras cada cambio:**
```cmd
ping obonilla.esrinosa.local
ping WIN-OC2B24K34HS
```

**Nota sobre permisos:** si Notepad da error "Acceso denegado" al guardar, es porque no se abrió como Administrador. Solución: cerrar Notepad, clic derecho sobre el ícono en el menú inicio → "Ejecutar como administrador", y abrir el archivo desde **File → Open**, cambiando el filtro de "Text Documents" a "All Files" (el archivo `hosts` no tiene extensión `.txt`).

## Firewall — Windows Defender Firewall

Se identificó que la comunicación entre la máquina física y la(s) VM(s) requiere **reglas de firewall bidireccionales** — no basta con abrir puertos en un solo sentido.

### Comandos de diagnóstico usados

```powershell
# Ver estado de los 3 perfiles de firewall
netsh advfirewall show allprofiles state

# Desactivar temporalmente TODOS los perfiles (solo para diagnóstico)
netsh advfirewall set allprofiles state off

# Reactivar después del diagnóstico
netsh advfirewall set allprofiles state on

# Probar conectividad a un puerto específico
Test-NetConnection -ComputerName <host> -Port <puerto>

# Ver si un puerto está en escucha (LISTENING) en la máquina local
netstat -ano | findstr <puerto>

# Ver reglas de firewall existentes por nombre
Get-NetFirewallRule -DisplayName "<nombre-regla>"

# Ver reglas de bloqueo activas que puedan tener prioridad
Get-NetFirewallRule -Direction Inbound -Action Block -Enabled True | Where-Object {$_.DisplayName -like "*6443*" -or $_.DisplayName -like "*ArcGIS*"}

# Ver el perfil de red activo de la conexión
Get-NetConnectionProfile

# Cambiar el perfil de red a Privado (si aparecía como Público)
Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
```

### Reglas de entrada (Inbound) creadas

**En la máquina física (host) del usuario — para permitir tráfico entrante desde la VM hacia Portal/Server:**
```powershell
New-NetFirewallRule -DisplayName "ArcGIS Server 6443" -Direction Inbound -LocalPort 6443 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS Portal 7443" -Direction Inbound -LocalPort 7443 -Protocol TCP -Action Allow
```

**En la VM de Data Store del usuario (`WIN-OC2B24K34HS`) — para permitir tráfico entrante desde la máquina física hacia el Data Store:**
```powershell
New-NetFirewallRule -DisplayName "Allow ICMP Ping" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS DataStore 2443" -Direction Inbound -LocalPort 2443 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS DataStore Spatiotemporal 9220" -Direction Inbound -LocalPort 9220 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS DataStore Spatiotemporal 9320" -Direction Inbound -LocalPort 9320 -Protocol TCP -Action Allow
```
> Nota: también se crearon inicialmente reglas para el puerto 9876 y el rango 29079-29080, basadas en una suposición incorrecta sobre qué puerto usaba el Spatiotemporal store (ver "Problemas encontrados" para la corrección). Estas reglas no son perjudiciales (abren puertos legítimos del Relational store / tile cache) pero **no eran las que resolvían el problema real**.

**En la VM de Velocity (`WIN-OC2B24K34HS`, mismo VM que Data Store en el Entorno A):**
```powershell
New-NetFirewallRule -DisplayName "ArcGIS Velocity 7143" -Direction Inbound -LocalPort 7143 -Protocol TCP -Action Allow
```

### Cómo abrir PowerShell como Administrador (paso crítico repetido)

Se detectó más de una vez que los comandos fallaban con `Acceso denegado` / `PermissionDenied` porque PowerShell **no** se ejecutaba elevado, aunque el prompt mostrara el usuario correcto:

1. Menú inicio → escribir `PowerShell`.
2. **Clic derecho** sobre "Windows PowerShell" → **"Ejecutar como administrador"**.
3. Aceptar el diálogo de Control de Cuentas de Usuario (UAC) si aparece.
4. **Verificación:** el título de la ventana debe decir explícitamente **"Administrador: Windows PowerShell"** (o "Administrador: Terminal"). Si no lo dice, no tiene privilegios elevados aunque el usuario del sistema sea administrador.

## Antivirus de terceros (Kaspersky) — Entorno B

Se detectó, mediante el siguiente comando, que la máquina física de la compañera tenía **Kaspersky Endpoint Security for Windows** además de Windows Defender:

```powershell
Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntivirusProduct | Select-Object displayName, productState
```

**Resultado obtenido:**
```
displayName                              productState
-----------                              ------------
Windows Defender                         393472
Kaspersky Endpoint Security for Windows  266240
```

**Comando adicional de diagnóstico sugerido** (para detectar VPN/EDR corporativo en general):
```powershell
Get-Service | Where-Object {$_.DisplayName -match "VPN|Endpoint|Defender|Security|Firewall|Zscaler|Cisco|CrowdStrike|SentinelOne|Symantec"}
```

**Acciones recomendadas dentro de Kaspersky** (para permitir el puerto necesario):
- **Configuración → Protección esencial → Firewall → Reglas de paquetes de red → Agregar** (Protocolo: TCP, Dirección: Entrante, Puerto: el requerido, Acción: Permitir).
- Revisar también el módulo **"Network Attack Blocker"**, distinto del "Firewall", que puede bloquear tráfico independientemente de que el firewall esté desactivado.
- Revisar clasificación de red en **Kaspersky → Configuración → Firewall → Redes**, ya que a veces clasifica la red de la VM como "Pública" y aplica restricciones más agresivas.
- Si Kaspersky está gestionado centralmente por **Kaspersky Security Center**, la política puede no ser modificable localmente; en ese caso se requiere contactar al administrador de TI/seguridad de la organización para agregar una excepción de puerto o marcar la IP de la VM como "de confianza".

## Configuración de VirtualBox

### Portapapeles bidireccional (Shared Clipboard)

1. VirtualBox Manager → VM → **Configuración → General → Avanzado**.
2. Cambiar **"Portapapeles compartido"** de "Deshabilitado" a **"Bidireccional"**.
3. (Opcional) Activar también **"Arrastrar y soltar"** en modo Bidireccional.
4. **Requisito indispensable:** tener las **Guest Additions** instaladas dentro de la VM:
   - Con la VM encendida: **Dispositivos → Insertar imagen CD de las Guest Additions...**
   - Ejecutar `VBoxWindowsAdditions.exe` dentro de la VM.
   - Reiniciar la VM.
5. Verificar que el servicio **"VBoxService"** esté corriendo dentro de la VM (`services.msc`).

### Transferencia de archivos completos (no solo texto/portapapeles)

Se aclaró que el portapapeles bidireccional **no** sirve para transferir archivos completos, solo texto/imágenes pequeñas. Opciones evaluadas:

1. **Arrastrar y soltar (Drag and Drop):** requiere Guest Additions; configurar en modo Bidireccional en el mismo panel de Configuración → General → Avanzado. Señalado como potencialmente inestable con archivos grandes.
2. **Carpetas compartidas (Shared Folders) — opción recomendada finalmente:**
   - VirtualBox Manager → VM → **Configuración → Carpetas compartidas → "+"**.
   - Seleccionar una ruta física (ej. `C:\Compartido`).
   - Asignar un nombre simple (ej. `Compartido`).
   - Marcar **"Montaje automático"** y **"Acceso total"**.
   - Acceso desde dentro de la VM: `\\VBOXSVR\Compartido` (o vía **Este equipo → Red → VBOXSVR\Compartido**).

## Cuentas de servicio de Windows (Data Store)

Durante la instalación de ArcGIS Data Store se solicitó la pantalla **"Specify ArcGIS Data Store Account"**:

- **Username:** `arcgis` (prellenado por el instalador).
- **Password:** definida específicamente para esa VM (no requiere coincidir con las contraseñas de Portal ni de Server, al tratarse de máquinas Windows independientes).
- **Distinción clave documentada:** esta cuenta es una **cuenta de sistema operativo (Windows)** que ejecuta el servicio en segundo plano — **no debe confundirse** con `portaladmin` (login web de Portal) ni `siteadmin` (Primary Site Administrator de ArcGIS Server), que son cuentas de aplicación web.

## Configuración de puertos para `configuredatastore.bat`

**Sintaxis general utilizada:**
```cmd
configuredatastore.bat <ArcGIS Server admin URL> <usuario admin> <password admin> <directorio de datos> --stores spatiotemporal
```

**Ejemplo real usado en la conversación:**
```cmd
cd "C:\Program Files\ArcGIS\DataStore\tools"
configuredatastore.bat https://obonilla.esrinosa.local:6443/arcgis admin Esri12345 C:\arcgisdatastore --stores spatiotemporal
```

**Notas de sintaxis:**
- Si la contraseña contiene caracteres especiales (espacios, `&`, `%`, etc.), debe ir entre comillas dobles: `"MiPass@123!"`.
- El directorio de datos debe existir o el instalador lo crea.
- Debe ejecutarse desde **CMD** (símbolo del sistema), como estándar para scripts `.bat` de Esri (aunque PowerShell también podría funcionar).
- Debe ejecutarse como **Administrador**.

---

# Problemas encontrados

| # | Problema | Causa | Solución | Estado |
|---|---|---|---|---|
| 1 | Duda sobre si desinstalar el Web Adaptor de Portal, el de Server, o ambos, antes de instalar uno nuevo | Confusión sobre si son instalaciones independientes | Se aclaró que Web Adaptor de Portal y de Server son independientes; solo se desinstala el que corresponde al componente que se actualiza | ✅ Resuelto (aclaración conceptual) |
| 2 | No saber qué archivo seleccionar en el "Software Authorization Wizard" | Falta de familiaridad con el proceso de licenciamiento de Esri | Se identificó que se requiere el archivo `.ecp` descargable desde my.esri.com → Licensing; alternativamente, autorización en línea con número de autorización | ✅ Resuelto |
| 3 | Error de federación: *"Error al validar la base de datos administrada del servidor '/enterpriseDatabases/AGSDataStore_ds_nmpts56z'"* | ArcGIS Data Store no estaba corriendo o no había completado su configuración inicial | Se recomendó verificar el servicio de Data Store, revisar `datastoreadmin`, conectividad por puerto 2443 y logs de Server | ✅ Diagnóstico entregado (resolución continuó más adelante en el chat con la reinstalación completa del Data Store) |
| 4 | Incertidumbre sobre si la versión de Enterprise instalada permitía instalar Velocity | Versión instalada no confirmada explícitamente por el usuario (se vio un número de build "20112" en una captura previa, posiblemente de una versión anterior) | Se investigó que ArcGIS Enterprise 12.1 es la versión mínima requerida para Velocity self-hosted | ⚠️ Parcialmente resuelto — la versión exacta original nunca fue confirmada explícitamente por el usuario; se asumió 12.1 por el título de ventana visto después ("ArcGIS Data Store 12.1 Setup") |
| 5 | Duda sobre si instalar primero el Spatiotemporal Data Store antes que Velocity | No estaba claro el orden de dependencia entre componentes | Se aclaró que Velocity no depende de tener el spatiotemporal ya configurado para instalarse, pero sí lo necesita para poder escribir salidas a feature layers espacio-temporales | ✅ Resuelto |
| 6 | Duda sobre en qué máquina se instala el Spatiotemporal Data Store: ¿en la de Velocity o en la del hosting server? | Confusión sobre la arquitectura de componentes | Se confirmó (con respaldo de documentación oficial) que el Data Store vive junto al hosting server, no en la máquina de Velocity | ✅ Resuelto |
| 7 | Necesidad de copiar texto entre la máquina física y la VM de VirtualBox | Portapapeles compartido desactivado por defecto | Activar "Portapapeles compartido" en modo Bidireccional + instalar Guest Additions | ✅ Resuelto |
| 8 | Necesidad de transferir un **archivo completo** (no solo texto) a la VM | El portapapeles no soporta archivos completos de forma confiable | Se recomendó usar **Carpetas compartidas** de VirtualBox (más estable que Drag & Drop para archivos grandes) | ✅ Resuelto |
| 9 | Error ejecutando `configuredatastore.bat --stores spatiotemporal`: *"El almacén 'spatiotemporal' no está instalado en la configuración actual"* | El instalador de ArcGIS Data Store no tenía seleccionada la característica "Spatiotemporal big data" (solo se instaló Relational por defecto) | Inicialmente se intentó "Modificar" la instalación vía Agregar/quitar programas; finalmente se optó por reinstalar desde cero en una nueva VM seleccionando los 3 tipos desde el inicio | ✅ Resuelto (con cambio de enfoque — ver Decisiones técnicas) |
| 10 | Confusión: *"pero mi admin no es portaladmin?"* | El usuario no distinguía entre la cuenta de Portal (`portaladmin`), la de Server (`siteadmin`/PSA) y la cuenta de Windows del servicio de Data Store | Se explicaron las 3 cuentas como entidades separadas y para qué sirve cada una | ✅ Resuelto |
| 11 | Error: *"Failed to log in. Invalid username or password specified."* al ejecutar `configuredatastore.bat` | Se usaron credenciales incorrectas (posiblemente las de Portal en lugar de las del Primary Site Administrator de ArcGIS Server) | Se indicó verificar credenciales directamente en `https://obonilla.esrinosa.local:6443/arcgis/admin` | ✅ Resuelto (tras corregir credenciales, apareció el siguiente error, el #9) |
| 12 | Confusión: *"pero recuerda que es del datastore que está en una máquina virtual ajena"* — ¿la cuenta de Windows debe coincidir con la del Server/Portal? | El usuario asumía que las cuentas de servicio de Windows debían coincidir entre máquinas | Se aclaró que, al ser VMs separadas, cada una tiene su propia cuenta de Windows local independiente | ✅ Resuelto |
| 13 | Pantalla del asistente de configuración de Data Store (`https://localhost:2443/arcgis/datastore/`) se veía como texto plano sin estilos | Certificado autofirmado no confiable bloqueando sub-recursos (CSS/JS), navegador incompatible, o caché corrupta | Se resolvió probando otro navegador y/o aceptando el certificado / limpiando caché | ✅ Resuelto |
| 14 | Confusión sobre qué poner en el campo **"Hosting server"** del asistente | El usuario pensaba que debía referirse a la propia VM de Data Store, cuando en realidad se refiere a la máquina del ArcGIS Server/hosting server | Se aclaró que debe apuntar al hostname:puerto del ArcGIS Server ya federado con Portal (no a la VM local) | ✅ Resuelto |
| 15 | Error: *"Could not connect to server on machine 'obonilla.esrinosa.local'..."* al completar "Hosting server details" | La VM de Data Store no podía resolver el hostname `obonilla.esrinosa.local` (sin DNS interno) | Se agregó entrada manual en el archivo `hosts` de la VM con la IP física correspondiente | ✅ Resuelto |
| 16 | Múltiples confusiones sobre qué `ipconfig` corresponde a qué máquina (física vs. VM Data Store vs. VM Portal/Server hipotética) | Se ejecutó `ipconfig` varias veces en máquinas equivocadas (adaptador Host-Only 192.168.56.1 del host físico, luego repetidamente la IP de la propia VM de Data Store) | Se guio paso a paso hasta clarificar que Portal/Server estaban en la **máquina física**, no en una VM separada | ✅ Resuelto (tras varias iteraciones) |
| 17 | Ping en un sentido (VM → física) funcionaba, pero en sentido inverso (física → VM, `ping WIN-OC2B24K34HS`) fallaba al 100% | El Firewall de Windows en la **VM de Data Store** bloqueaba conexiones entrantes (ICMP y TCP) desde la máquina física | Se crearon reglas de entrada en el firewall de la VM (ICMP, puertos 2443, 9876, rango 29079-29080) | ✅ Resuelto (ping bidireccional confirmado con 0% de pérdida) |
| 18 | Firewall de Windows en la **máquina física** bloqueando el puerto 6443 entrante desde la VM | Regla de firewall no existente para el puerto 6443 en el host | Se creó la regla `New-NetFirewallRule -DisplayName "ArcGIS Server 6443" -Direction Inbound -LocalPort 6443 -Protocol TCP -Action Allow` | ✅ Resuelto |
| 19 | Error `Acceso denegado` / `PermissionDenied` al ejecutar `New-NetFirewallRule` | PowerShell no se ejecutaba como Administrador, pese a mostrar el usuario correcto en el prompt | Se explicó cómo abrir PowerShell elevado y verificar el título de ventana ("Administrador: ...") | ✅ Resuelto |
| 20 | Mismo error de conexión (*"Could not connect to server on machine 'geoportal.esri.co'"*) reproducido en el entorno de la compañera | Mismo patrón: DNS + firewall, en un entorno completamente distinto | Se replicó exactamente el mismo procedimiento (hosts + firewall) para la compañera | ✅ Resuelto |
| 21 | Tras abrir el firewall de Windows en la máquina de la compañera, el puerto 6443 seguía sin responder (`TcpTestSucceeded: False`), incluso con **todos los perfiles de firewall desactivados** | **Kaspersky Endpoint Security for Windows** tenía su propio motor de firewall, independiente del Firewall de Windows nativo (no se desactiva con `netsh advfirewall`) | Se identificó Kaspersky vía `Get-CimInstance` y se indicó revisar/desactivar su Firewall y el módulo "Network Attack Blocker" | ✅ Resuelto (la compañera confirmó "está apagado el firewall" de Kaspersky, y posteriormente su Data Store avanzó exitosamente) |
| 22 | Confusión sobre si la federación del Spatiotemporal Data Store debía hacerse con el servidor de **Velocity** o con el **GIS Server** | El usuario no tenía claro qué componente actúa como "hosting server" para efectos del Data Store | Se aclaró que la federación debe ser con el **ArcGIS Server (GIS Server / hosting server)**, no con Velocity | ✅ Resuelto |
| 23 | Se estuvo a punto de crear una federación de servidor **duplicada** usando "Add Server Site", cuando el ArcGIS Server ya estaba federado | Malentendido sobre la causa del ❗ (rojo) en Espaciotemporal — se asumió que faltaba federar, cuando ya estaba federado | Se instruyó **cancelar** el formulario "Add Server Site" y verificar primero la lista de "Servidores federados" existente | ✅ Resuelto (se confirmó que `obonilla.esrinosa.local:6443` ya aparecía como "All systems operational") |
| 24 | Data Store **Espaciotemporal** seguía en ❗ (rojo) en la tabla de validación, aunque Relacional y Administrado Objeto ya estaban en ✅ (verde) | Falta de comunicación **bidireccional** de red entre la máquina física y la VM del Data Store (el ping en sentido físico→VM fallaba al 100%) | Ver problema #17 — al resolver la comunicación bidireccional, el estado cambió a ✅ verde | ✅ Resuelto — confirmado con los 3 Data Stores en verde para el usuario |
| 25 | Mismo problema (#24) reproducido en el entorno de la compañera: Espaciotemporal en rojo pese a que Relacional y Objeto ya validaban en verde | Firewall de la VM de la compañera (`WIN-KQ3HPTQDHP1`) bloqueando tráfico entrante | Se aplicaron las mismas reglas de firewall en su VM | ⚠️ Parcialmente resuelto — el puerto 2443 conectó (`True`), pero persistió un fallo en el puerto usado para diagnosticar (ver problema #26) |
| 26 | Se probó el puerto **9876** para diagnosticar el bloqueo del Espaciotemporal, y seguía fallando (`TcpTestSucceeded: False`) incluso tras crear reglas de firewall para ese puerto | **Error de diagnóstico:** el puerto 9876 corresponde al **Relational store**, no al **Spatiotemporal big data store** (confirmado consultando la documentación oficial de Esri) | Se corrigió el diagnóstico usando los puertos correctos del Spatiotemporal: **9220** y **9320** | ✅ Resuelto — ambos puertos correctos confirmaron `TcpTestSucceeded: True` y `LISTENING` |
| 27 | Posible existencia de una carpeta de contenido **duplicada** (`C:\arcgisdatastore` vs. `C:\Enterprise\arcgisdatastore`) en la VM de la compañera, por reintentos fallidos previos del asistente | Un primer intento de configuración pudo haber quedado a medias usando `C:\arcgisdatastore`, y un reintento posterior usó una ruta distinta (`C:\Enterprise\arcgisdatastore`) que sí se completó | Se recomendó primero revalidar en Portal (posiblemente ya resuelto solo con la corrección de puertos), y solo si persistía, verificar la ruta activa en la configuración del Data Store y eliminar la carpeta residual si estaba vacía/incompleta | ⚠️ **Sin confirmación final en la conversación** — información no especificada si finalmente se eliminó alguna carpeta |
| 28 | Error de validación de formulario al federar Velocity: *"A service URL is required" / "An administration URL is required"* | **Error de tipeo**: se escribió `htttps://win-oc2b24k34hs:7143/arcgis` (con **tres** "t") en lugar de `https://` | Se identificó y corrigió el typo en ambos campos (Services URL y Administration URL) | ✅ Resuelto (identificado; corrección pendiente de confirmación explícita por el usuario) |
| 29 | Error en paralelo: *"Could not access any server machines. Please contact your system administrator."* en `obonilla.esrinosa.local/server/manager` | **Causa no confirmada en la conversación** — posible relación con cambios de red/firewall realizados mientras se trabajaba en la federación de Velocity | Se sugirió verificar el estado del servicio "ArcGIS Server" en `services.msc` | ⚠️ **Sin resolución confirmada al cierre de la conversación** — información no especificada |
| 30 | Error: *"No se puede acceder a la dirección URL de administración de ArcGIS Server 'https://win-oc2b24k34hs:7143/arcgis' desde Portal for ArcGIS"* al federar Velocity | Mismo patrón de conectividad ya visto: puerto 7143 (Velocity) posiblemente bloqueado entre la máquina física y la VM | Se indicó repetir el mismo procedimiento de diagnóstico: `Test-NetConnection` al puerto 7143, y de ser necesario, crear regla de firewall `New-NetFirewallRule -DisplayName "ArcGIS Velocity 7143" -Direction Inbound -LocalPort 7143 -Protocol TCP -Action Allow` en la VM de Velocity | ⚠️ **Pendiente de confirmación final** — la conversación concluye (para dar paso a la solicitud de este documento) antes de confirmar si el puerto 7143 quedó abierto y la federación de Velocity se completó |

---

# Preguntas y respuestas

> Reconstrucción de cada pregunta relevante planteada durante la conversación, en orden cronológico, con su respuesta, explicación técnica y resultado.

### P1. ¿Cómo verifico que Portal se instaló correctamente, antes de desinstalar el Web Adaptor?
- **Respuesta:** Verificar el servicio de Windows, acceso directo por HTTPS al puerto 7443, carpeta de instalación, logs, y REST API de portaladmin.
- **Explicación técnica:** Portal es independiente del Web Adaptor; este último es solo un reverse-proxy.
- **Resultado:** El usuario confirmó configuración inicial completada ("ya está").

### P2. ¿Debo desinstalar el Web Adaptor de Portal y de Server, o solo el de Portal, para instalar el nuevo?
- **Respuesta:** Son instalaciones independientes; solo se desinstala el que corresponde al componente actualizado.
- **Explicación técnica:** Cada Web Adaptor apunta a un componente distinto (Portal o Server) aunque compartan el mismo servidor IIS.
- **Resultado:** Se ofrecieron 4 escenarios posibles vía pregunta de opciones; el usuario respondió "sin preferencia", por lo que se entregó la guía completa asumiendo el escenario más probable (Portal recién instalado, primer Web Adaptor).

### P3. (Captura) "Software Authorization Wizard" — ¿qué pongo aquí?
- **Respuesta:** Usar la opción ya seleccionada (archivo de autorización `.ecp` recibido de Esri), buscarlo con "Browse", o alternativamente autorizar en línea con el número de autorización.
- **Explicación técnica:** El archivo `.ecp` se descarga desde my.esri.com → Licensing → My Organizations' Licenses.
- **Resultado:** No se confirmó explícitamente en el chat cuál opción usó finalmente el usuario.

### P4. (Captura) Error de federación: *"Error al validar la base de datos administrada del servidor..."*
- **Respuesta:** Indica que ArcGIS Server se federó correctamente, pero falla la validación del Relational Data Store.
- **Explicación técnica:** Puede deberse a Data Store no iniciado, mal configurado, problema de red/DNS, puerto 2443 bloqueado, o configuración incompleta.
- **Resultado:** Este hilo de diagnóstico se retomó y resolvió más adelante, tras la reinstalación completa del Data Store en una VM nueva.

### P5. ¿Cuál es la última versión de ArcGIS Enterprise?
- **Respuesta:** ArcGIS Enterprise **12.1**, con nuevas funciones como ArcGIS Data Pipelines y mejoras de observabilidad.
- **Explicación técnica:** A partir de la versión 12.0, Esri distingue soporte long-term (~4 años) y short-term (~1 año).
- **Resultado:** Se recomendó verificar la versión exactamente instalada antes de planear actualización a 12.1.

### P6. ¿Puedo instalar Velocity Server? ¿Desde qué versión de Enterprise?
- **Respuesta:** Sí, self-hosted, mínimo desde **ArcGIS Enterprise 12.1**.
- **Explicación técnica:** Antes de 12.1, Velocity solo existía como SaaS de ArcGIS Online; versiones anteriores solo tenían GeoEvent Server on-premises (que será descontinuado en la versión 12.3, ~2027).
- **Resultado:** Se determinó que si la versión instalada era anterior a 12.1, sería necesario actualizar toda la suite primero.

### P7. ¿Cómo instalo Velocity?
- **Respuesta:** Prerrequisitos de hardware/DNS, descarga desde My Esri, ejecución de `VelocitySetup.exe`, configuración vía navegador (federación con Portal, licenciamiento propio).
- **Explicación técnica:** Velocity requiere federación con un portal Enterprise ya desplegado; la versión self-hosted inicial no incluye Big Data Analytics.
- **Resultado:** Se procedió a instalar Velocity en una VM separada.

### P8. ¿Cómo se hace el Data Store espacio temporal? Ya tengo el despliegue base.
- **Respuesta:** Instalar ArcGIS Data Store con el tipo Spatiotemporal, y ejecutar `configuredatastore.bat ... --stores spatiotemporal`.
- **Explicación técnica:** El spatiotemporal big data store es necesario para datos observacionales de alto volumen usados por Velocity/GeoEvent.
- **Resultado:** Se inició el proceso, que tomaría varias iteraciones hasta completarse exitosamente.

### P9. ¿Eso (el spatiotemporal) es antes de instalar Velocity en la otra máquina?
- **Respuesta:** No es estrictamente obligatorio antes, pero es buena práctica tenerlo listo si se sabe que se necesitará; es opcional según el volumen de datos esperado.
- **Explicación técnica:** Velocity puede funcionar con el Data Store relacional para casos simples; el spatiotemporal solo es necesario para alto volumen de datos observacionales.
- **Resultado:** Se recomendó instalar Velocity primero y agregar spatiotemporal después si se necesitaba, sin necesidad de reinstalar Velocity.

### P10. ¿Esto (`configuredatastore.bat`) lo hago en terminal CMD, verdad?
- **Respuesta:** Sí, desde CMD como Administrador, navegando a `C:\Program Files\ArcGIS\DataStore\tools`.
- **Explicación técnica:** Es el estándar para scripts `.bat` de Esri.
- **Resultado:** Se ejecutó el comando, arrojando posteriormente un error de autenticación.

### P11. ¿Pero mi admin no es `portaladmin`?
- **Respuesta:** `portaladmin`, `siteadmin` (PSA de Server) y la cuenta de Windows del servicio de Data Store son 3 cuentas distintas.
- **Explicación técnica:** El comando requiere específicamente el Primary Site Administrator de ArcGIS Server.
- **Resultado:** Se identificó que las credenciales usadas ("admin"/"Esri12345") eran incorrectas para ese propósito.

### P12. (Captura) Error: *"Failed to log in. Invalid username or password specified."*
- **Respuesta:** Confirmar credenciales directamente en `https://obonilla.esrinosa.local:6443/arcgis/admin`.
- **Explicación técnica:** Las credenciales usadas no correspondían al Primary Site Administrator real del Server.
- **Resultado:** Tras corregir credenciales, se obtuvo un nuevo error (característica no instalada).

### P13. (Captura) Error: *"El almacén 'spatiotemporal' no está instalado en la configuración actual."*
- **Respuesta:** Usar la opción "Modificar" en Agregar o quitar programas para añadir la característica Spatiotemporal big data.
- **Explicación técnica:** El instalador de Data Store solo había instalado el tipo Relational por defecto.
- **Resultado:** Se inició el proceso de modificación, pero fue abandonado poco después a favor de una reinstalación limpia en nueva VM.

### P14. ¿Pero ese Data Store no se instala en la máquina donde está Velocity?
- **Respuesta:** No; el Data Store vive junto al hosting server (ArcGIS Server), no en la máquina de Velocity.
- **Explicación técnica:** Velocity es solo el motor de procesamiento; las salidas de sus analíticas se almacenan en el Data Store del hosting server, accedido vía Portal federado.
- **Resultado:** Confirmado con diagrama de arquitectura; el usuario continuó configurando el Data Store en la máquina/VM correcta.

### P15. ¿Cómo activo el portapapeles en doble sentido para mi máquina virtual de VirtualBox?
- **Respuesta:** Configuración → General → Avanzado → Portapapeles compartido → Bidireccional, más instalación de Guest Additions.
- **Explicación técnica:** Sin Guest Additions instaladas, la opción no tiene efecto aunque esté activada.
- **Resultado:** Pregunta de seguimiento reveló que en realidad se necesitaba transferir un archivo completo (ver P16).

### P16. Quiero pegar un archivo completo.
- **Respuesta:** El portapapeles no sirve para archivos completos; usar Carpetas compartidas (recomendado) o Drag & Drop.
- **Explicación técnica:** Carpetas compartidas es más estable para archivos grandes como instaladores de ArcGIS.
- **Resultado:** Se detalló el procedimiento completo de configuración de carpeta compartida.

### P17. (Captura "Select Features") ¿Ya no pongo Relational y Object?
- **Respuesta:** Se debe dejar TODO instalado; solo se agrega Spatiotemporal, no se quita nada existente.
- **Explicación técnica:** Al estar en modo "Modificar" sobre una instalación existente, la idea es añadir, no remover características.
- **Resultado:** El usuario decidió abandonar este enfoque momentos después (ver P18).

### P18. "No no ya olvida eso, ya es una nueva máquina virtual."
- **Respuesta:** Se preguntó en qué punto estaba la nueva VM para retomar la guía desde ahí.
- **Explicación técnica:** Cambio de estrategia — instalar Data Store desde cero en VM nueva, seleccionando Spatiotemporal desde el inicio.
- **Resultado:** Se reinició el hilo de instalación con la nueva VM.

### P19. (Captura cuenta de Data Store) ¿Qué usuario pongo? Portal es `portaladmin` y Server es `siteadmin`.
- **Respuesta:** Es una cuenta de Windows (sistema operativo) distinta a `portaladmin`/`siteadmin`; se puede dejar `arcgis` (prellenado) con una contraseña nueva.
- **Explicación técnica:** Esta pantalla configura la cuenta bajo la cual corre el **servicio de Windows** de Data Store, no una cuenta de aplicación web.
- **Resultado:** El usuario aclaró que esta VM era distinta a la de Portal/Server (ver P20), ajustando la respuesta.

### P20. "Pero recuerda que es del Data Store que está en una máquina virtual ajena."
- **Respuesta:** Al ser VM separada, la cuenta de Windows es local a esa máquina y no necesita coincidir con Portal/Server.
- **Explicación técnica:** Cada máquina Windows (física o virtual) tiene su propio conjunto de cuentas de sistema operativo, independiente entre sí.
- **Resultado:** Se dejó `arcgis` como usuario con una contraseña definida específicamente para esa VM.

### P21. (Captura servidores federados con hostname genérico "WIN-...") ¿Qué significa ese "win"?
- **Respuesta:** Es el nombre de máquina autogenerado por Windows cuando no se personaliza durante la instalación del sistema operativo.
- **Explicación técnica:** No es un error — el estado "All systems operational" confirma que el servidor (Velocity, en ese ejemplo) se federó correctamente pese al nombre genérico.
- **Resultado:** Se recomendó, como buena práctica, renombrar la máquina antes de producción (con la advertencia de que esto rompe la federación existente y requiere re-federar).

### P22. (Captura wizard sin estilos CSS) "Ya puse el Data Store en mi VM, pero sale esto."
- **Respuesta:** Problema visual (certificado autofirmado / navegador / caché), no funcional.
- **Explicación técnica:** El formulario puede seguir funcionando aunque se vea sin estilo.
- **Resultado:** Se resolvió cambiando de navegador / aceptando certificado — confirmado por el usuario ("ya se solucionó").

### P23. (Captura "Hosting server details") "Ahora me pide un hosting server, pero es una máquina virtual. ¿Qué hago?"
- **Respuesta:** "Hosting server" se refiere a la máquina del ArcGIS Server ya federado con Portal, no a la VM local de Data Store.
- **Explicación técnica:** Se debe ingresar hostname:puerto del hosting server (ej. `jquinteros.esrinosa.local:6443` en el ejemplo mostrado, o `obonilla.esrinosa.local:6443` en el entorno real del usuario).
- **Resultado:** El usuario intentó conectar y obtuvo un error de conexión (ver P24).

### P24. (Captura) Error: *"Could not connect to server on machine 'obonilla.esrinosa.local'..."*
- **Respuesta:** Problema de conectividad de red; diagnosticar DNS, firewall, servicio activo, y modo de red de VirtualBox.
- **Explicación técnica:** Se sugirió una secuencia de diagnóstico: ping → hosts file → telnet/Test-NetConnection → modo de red VirtualBox.
- **Resultado:** Inició una larga cadena de diagnóstico de conectividad (P25-P34).

### P25-P30. Secuencia de confusión sobre `ping`, `ipconfig` y qué máquina es cuál
- **Preguntas:** "¿Esto dónde pongo, `ping obonilla.esrinosa.local`?", múltiples capturas de `ipconfig` que resultaron ser de la máquina física (host, adaptador Host-Only 192.168.56.1) o de la propia VM de Data Store repetida (192.168.100.250), en lugar de la máquina de Portal/Server.
- **Respuesta:** Se guio reiteradamente a identificar qué IP correspondía a cada máquina, aclarando la diferencia entre el adaptador VirtualBox Host-Only (solo visible en el host físico) y las IPs de red "puente"/Wi-Fi compartidas con las VMs.
- **Explicación técnica:** El adaptador `192.168.56.1` es exclusivo de VirtualBox en el host físico y nunca aparece dentro de una VM.
- **Resultado:** Se llegó a la aclaración clave (P27): Portal y Server estaban instalados directamente en la máquina física del usuario, no en una VM separada.

### P27. "Es que Portal y Server están en mi máquina host."
- **Respuesta:** Esto simplifica todo: usar la IP física ya obtenida (`192.168.100.248`) para el archivo `hosts` de la VM.
- **Explicación técnica:** Se debía además abrir el firewall del **host físico** para el puerto 6443 (y potencialmente 7443), ya que el tráfico entrante desde la VM podía estar bloqueado ahí.
- **Resultado:** Se procedió a editar el archivo `hosts` y crear reglas de firewall.

### P31. "¿Cómo es eso? ¿Cuál es el comando?" (para abrir el firewall)
- **Respuesta:** `New-NetFirewallRule -DisplayName "ArcGIS Server 6443" -Direction Inbound -LocalPort 6443 -Protocol TCP -Action Allow`, ejecutado en PowerShell como Administrador en la máquina física.
- **Explicación técnica:** También se dio la alternativa gráfica vía Firewall de Windows Defender → Reglas de entrada → Nueva regla.
- **Resultado:** Primer intento falló por falta de privilegios elevados (ver P32).

### P32. (Captura) Error "Acceso denegado" al ejecutar el comando de firewall
- **Respuesta:** PowerShell no se estaba ejecutando como Administrador; instrucciones para abrirlo correctamente (clic derecho → Ejecutar como administrador) y verificar el título de la ventana.
- **Explicación técnica:** El prompt puede mostrar un usuario administrador del sistema sin que la sesión de PowerShell tenga privilegios elevados.
- **Resultado:** Al reintentar correctamente elevado, la regla se creó con éxito (ver P33).

### P33. (Captura) Regla de firewall creada exitosamente
- **Respuesta:** Confirmación de éxito (`Enabled: True`, `PrimaryStatus: OK`); reintentar el asistente de Data Store.
- **Resultado:** El usuario continuó con el proceso, luego trasladó la atención al entorno de su compañera.

### P34-P50. Réplica completa del proceso de diagnóstico para el entorno de la compañera (`geoportal.esri.co`)
- **Preguntas encadenadas:** Mismo error de conexión, mismas pruebas de `ping`/`Test-NetConnection`, verificación de que el servicio de Server funcionaba localmente (`https://localhost:6443/arcgis/admin` sí cargaba en su máquina física), verificación de `netstat` confirmando `0.0.0.0:6443 LISTENING`, prueba de desactivar el firewall completo (`netsh advfirewall set allprofiles state off/on`) sin éxito.
- **Respuesta clave:** Se sospechó y confirmó la presencia de un antivirus de terceros mediante `Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntivirusProduct`, revelando **Kaspersky Endpoint Security for Windows**.
- **Explicación técnica:** Kaspersky mantiene su propio motor de firewall (WFP) independiente del Firewall de Windows nativo; `netsh advfirewall` no lo desactiva.
- **Resultado:** Tras revisar la configuración de Kaspersky ("está apagado el firewall"), el entorno de la compañera pudo avanzar y finalmente completar la configuración del Data Store.

### P51. "Ya, ahora ya está" (configuración inicial completada) + inicio de validación de Data Stores
- **Respuesta:** Validar cada Data Store individualmente en Portal (Relacional, Administrado Objeto, Espaciotemporal) usando los botones "Validar"/"Validar todo".
- **Resultado:** Relacional y Administrado Objeto validaron en verde; Espaciotemporal quedó en rojo (❗).

### P52. ¿Esa federación (del Espaciotemporal) se debe hacer con el servidor Velocity o el GIS Server?
- **Respuesta:** Con el **GIS Server** (ArcGIS Server / hosting server), no con Velocity.
- **Explicación técnica:** El Data Store es una extensión de almacenamiento del hosting server; Velocity se federa de forma completamente independiente.
- **Resultado:** Se verificó que el ArcGIS Server ya estaba federado (ver P53).

### P53. (Captura "Add server site" vacío) "¿Qué pongo aquí?"
- **Respuesta:** **Cancelar** — no se debía agregar un nuevo servidor, ya que el existente ya estaba federado.
- **Explicación técnica:** Agregar otro servidor generaría una federación duplicada/conflictiva.
- **Resultado:** Se verificó en la lista de "Servidores federados" que `obonilla.esrinosa.local:6443` ya aparecía con estado "All systems operational".

### P54-P55. "Le doy clic [Validar] y no pasa nada" / "No aparece ningún error"
- **Respuesta:** Revisar logs directamente (Server Manager → Logs, o los logs propios del Data Store en `C:\arcgisdatastore\logs`), dado que la interfaz no mostraba el detalle del error.
- **Resultado:** El usuario reportó (más adelante) un hallazgo clave por su cuenta: el ping en sentido físico→VM fallaba completamente (ver P56).

### P56. `ping WIN-OC2B24K34HS` desde la máquina física falla al 100% (Tiempo de espera agotado)
- **Respuesta:** Esta es la causa raíz — se requiere comunicación **bidireccional**, no solo VM→física.
- **Explicación técnica:** El Firewall de Windows de la VM bloqueaba conexiones entrantes (ICMP y TCP) desde la máquina física.
- **Resultado:** Se crearon reglas de firewall en la VM (ICMP, 2443, 9876, rango 29079-29080); el ping bidireccional quedó confirmado con 0% de pérdida.

### P57. Tras resolver el ping, ¿cómo valido en Portal?
- **Respuesta:** Regresar a la tabla de Data Stores, marcar el checkbox de Espaciotemporal, y hacer clic en "Validar".
- **Resultado:** ✅ Los 3 Data Stores quedaron en verde para el usuario.

### P58. "A mí ya se me validó, pero a mi compañera se le sigue saliendo en rojo."
- **Respuesta:** Aplicar la misma solución (reglas de firewall en su VM) y verificar ping bidireccional desde su máquina física.
- **Resultado:** Ping funcionó, pero la validación de Espaciotemporal seguía fallando (ver P59-P63).

### P59-P63. Diagnóstico de puertos específicos para la compañera (2443 y 9876)
- **Preguntas:** "¿Qué debe aparecer con `Test-NetConnection ... -Port 9876`?", confirmaciones de resultados `True`/`False` en distintos puertos, confusión sobre en qué máquina ejecutar `netstat`.
- **Respuesta:** El puerto 2443 conectaba bien (`True`), pero el 9876 seguía fallando (`False`), incluso confirmando con `netstat` que nada escuchaba en ese puerto **dentro de la VM** (aunque el servicio de Data Store sí estaba "Running").
- **Resultado:** Esto llevó a **corregir el diagnóstico** (ver P64).

### P64. ¿Y si desinstalo el Data Store de la compañera y reinstalo?
- **Respuesta:** No recomendado — el problema es de red/firewall, no de instalación (que ya se había completado exitosamente, llegando a la pantalla de "Configuration status").
- **Explicación técnica:** Reinstalar implicaría repetir todo el proceso y enfrentar el mismo problema de raíz.
- **Resultado:** Se recomendó primero probar el puerto correcto (ver P65) antes de considerar reinstalar.

### P65. Corrección: los puertos correctos del Spatiotemporal son 9220 y 9320 (no 9876)
- **Respuesta:** Se consultó la documentación oficial de Esri, confirmando que el puerto 9876 pertenece al **Relational store**, y que el Spatiotemporal usa **9220** (comunicación con hosting server) y **9320** (comunicación interna de clúster).
- **Resultado:** Al probar y abrir estos puertos correctos, ambos confirmaron `TcpTestSucceeded: True` y `LISTENING`.

### P66. ¿Cómo valido el portal después de eso? / ¿Con qué comando verifico que ya hay conexión?
- **Respuesta:** `Test-NetConnection` a los puertos 9220/9320, y luego regresar a Portal a marcar/validar Espaciotemporal.
- **Resultado:** Confirmado exitoso por el usuario ("ya sale esto, y el otro sí es listening").

### P67. Duda sobre una posible carpeta duplicada (`C:\arcgisdatastore` vs `C:\Enterprise\arcgisdatastore`) en el entorno de la compañera
- **Respuesta:** Primero revalidar en Portal (probablemente ya resuelto solo con la corrección de puertos); si persiste, verificar la ruta activa vía la configuración del Data Store y eliminar la carpeta residual solo si está confirmado que no tiene datos importantes.
- **Resultado:** **Sin confirmación final en la conversación** sobre si se eliminó alguna carpeta.

### P68. (Capturas) Error de tipeo "htttps" + error "Could not access any server machines" en Server Manager, al federar Velocity
- **Respuesta:** Corregir el typo (`htttps` → `https`); investigar por separado el error de Server Manager (verificar servicio "ArcGIS Server" en `services.msc`).
- **Resultado:** El usuario aclaró que `win-oc2b24k34hs:7143` correspondía a su propia VM con Velocity instalado (ver P69).

### P69. "Ese win-oc3 es el nombre de mi VM y el 7143 es el Velocity que está en mi VM."
- **Respuesta:** Confirmado — proceder a corregir el typo y completar la federación; para el error de Server Manager, se sugirió verificar el puerto 7143 con `Test-NetConnection` y crear la regla de firewall correspondiente en la VM.
- **Resultado:** Pendiente de confirmación final al momento de solicitarse este documento.

### P70. Solicitud de generar este documento de reconstrucción técnica
- **Respuesta:** Este mismo documento.

---

# Decisiones técnicas

### ¿Por qué ArcGIS Data Store (todos los tipos) y ArcGIS Velocity quedaron en la misma VM, en lugar de máquinas separadas?

Esri recomienda para producción una arquitectura de 3 máquinas separadas (Web GIS Server, Real-Time Server, Big Data Server), y el propio asistente de instalación advierte explícitamente: *"While more than one type of data store can be configured on a single machine, it is not recommended for production systems due to performance considerations."* Sin embargo, por tratarse de un **entorno de laboratorio/pruebas en VirtualBox**, se optó por combinar Relational, Object, Spatiotemporal y Velocity en una única VM (`WIN-OC2B24K34HS`), priorizando simplicidad sobre rendimiento en esta etapa. **Alternativa descartada:** separar cada componente en su propia VM, descartada por motivos de recursos/tiempo en el contexto de pruebas.

### ¿Por qué Portal y ArcGIS Server quedaron en la máquina física, y no en una VM?

No se documentó una justificación explícita en la conversación — el usuario simplemente confirmó que "Portal y Server están en mi máquina host" cuando se le preguntó. *Información no especificada en la conversación* respecto a si esta fue una decisión deliberada de arquitectura o una consecuencia de cómo se instaló originalmente el entorno.

### Se abandonó el enfoque de "Modificar" la instalación existente de Data Store para agregar Spatiotemporal

**Contexto:** Tras el error *"El almacén 'spatiotemporal' no está instalado en la configuración actual"*, se inició el proceso de ir a Panel de Control → Programas y características → ArcGIS Data Store → Modificar, para marcar la característica "Spatiotemporal big data" como instalable. **Esta ruta fue abandonada** cuando el usuario indicó *"no no ya olvida eso, ya es una nueva máquina virtual"*. **Decisión final:** crear una VM completamente nueva e instalar ArcGIS Data Store desde cero, seleccionando los tres tipos de almacén (Relational, Object, Spatiotemporal) desde el instalador inicial. **Razón implícita:** evitar posibles inconsistencias de una instalación modificada a medias, prefiriendo partir de un estado limpio.

### Corrección de diagnóstico: puerto 9876 → puertos 9220/9320 para el Spatiotemporal store

**Decisión revisada durante la sesión:** inicialmente se asumió (sin verificar contra documentación oficial) que el puerto 9876 —usado también para el Relational store— era relevante para diagnosticar el bloqueo del Spatiotemporal store en el entorno de la compañera. Esto generó varias iteraciones de prueba infructuosas. **Se corrigió** tras consultar la documentación oficial de Esri (`enterprise.arcgis.com` / `doc.esri.com`), confirmando que los puertos correctos para el Spatiotemporal big data store son **9220** y **9320**. Esta corrección se documenta explícitamente para que el próximo ingeniero no repita el mismo error de diagnóstico.

### Autorización de software: archivo `.ecp` vs. autorización en línea

Se presentaron dos alternativas para autorizar el software: (a) usar un archivo `.ecp` ya recibido de Esri, o (b) autorizar en línea con un número de autorización, si hay conexión a internet. **El usuario ya tenía la opción (a) preseleccionada** en el wizard al mostrar la captura, por lo que se continuó por esa vía sin necesidad de cambiar a la alternativa en línea.

### Uso de "Carpetas compartidas" de VirtualBox en lugar de Drag & Drop

Para transferir archivos completos (p. ej. instaladores grandes) entre el host físico y las VMs, se evaluaron **Drag & Drop** y **Carpetas compartidas**. **Se recomendó y adoptó Carpetas compartidas** por ser más estable con archivos grandes, mientras que Drag & Drop fue señalado como potencialmente inestable para ese caso de uso.

---

# Lecciones aprendidas

1. **La comunicación entre máquinas de un despliegue de ArcGIS Enterprise debe verificarse en ambos sentidos.** Un `ping` o `Test-NetConnection` exitoso en una dirección (VM → física) no garantiza que la dirección opuesta (física → VM) funcione; varios componentes de ArcGIS (especialmente Data Store) requieren comunicación bidireccional para validar correctamente.

2. **No asumir qué puerto usa un componente sin verificar la documentación oficial.** Se perdió tiempo diagnosticando el puerto 9876 (Relational store) para un problema que en realidad afectaba al Spatiotemporal store (puertos 9220/9320). Antes de abrir reglas de firewall "a ciegas", conviene consultar la tabla oficial de puertos de Esri para el componente y versión exactos.

3. **Los antivirus/EDR de terceros (p. ej. Kaspersky, pero aplica a cualquier otro) pueden mantener su propio motor de firewall, independiente de `netsh advfirewall`.** Si `netsh advfirewall set allprofiles state off` no resuelve un bloqueo de puerto pese a estar realmente aplicado, el siguiente sospechoso es un antivirus corporativo con firewall propio — verificable con `Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntivirusProduct`.

4. **Distinguir claramente entre las distintas "cuentas de administrador"** presentes en un despliegue de ArcGIS Enterprise: cuenta de Portal (`portaladmin`), Primary Site Administrator de ArcGIS Server (`siteadmin`), y cuenta de servicio de Windows para ArcGIS Data Store (p. ej. `arcgis`). Confundirlas es una fuente frecuente de errores de autenticación al ejecutar `configuredatastore.bat` u otros comandos administrativos.

5. **Verificar siempre que PowerShell/CMD se ejecute con privilegios elevados** antes de ejecutar comandos administrativos (`New-NetFirewallRule`, `netsh advfirewall`, etc.). El prompt puede mostrar un usuario con permisos de administrador en el sistema sin que la sesión de PowerShell esté realmente elevada; la única confirmación fiable es que el título de la ventana diga explícitamente "Administrador".

6. **En laboratorios con VirtualBox, fijar direcciones IP estáticas (o reservas DHCP) para cada máquina/VM** habría evitado la confusión observada en el entorno de la compañera, donde la misma dirección IP (`192.168.100.249`) pareció corresponder, en distintos momentos, tanto a la máquina física como a la VM de Data Store.

7. **Sin DNS interno funcional, el archivo `hosts` de Windows es una solución rápida y efectiva** para resolver nombres entre máquinas de un laboratorio, pero debe aplicarse **en ambas direcciones** cuando la comunicación es bidireccional (se debió editar el `hosts` tanto en la VM como en la máquina física).

8. **Combinar más de un tipo de Data Store (y, en este caso, también Velocity) en una sola máquina es válido para pruebas, pero Esri lo desaconseja explícitamente para producción** por motivos de rendimiento. Cualquier plan de productivización de este entorno debería separar estos componentes.

9. **Renombrar una máquina después de federarla rompe la federación existente.** Si se planea usar nombres de host descriptivos (en lugar de los autogenerados `WIN-XXXXXXXXXX`), esto debe hacerse **antes** de federar los servidores con Portal, o estar preparado para re-federar después del cambio.

10. **Revisar los logs (Server Manager, Portal, y los propios logs de Data Store en `C:\arcgisdatastore\logs`) cuando la interfaz gráfica de validación no muestra un mensaje de error claro.** La interfaz de "Validar" en Portal no siempre expone el detalle técnico suficiente para diagnosticar.

---

# Checklist final

Checklist para validar una nueva instalación replicando este despliegue:

- [ ] Portal for ArcGIS instalado y con configuración inicial (administrador primario) completada
- [ ] Software autorizado (archivo `.ecp` aplicado o autorización en línea completada)
- [ ] Servicio de Windows "Portal for ArcGIS" en estado Running
- [ ] Acceso verificado a `https://<host>:7443/arcgis/home`
- [ ] ArcGIS Server instalado, con Primary Site Administrator (PSA) configurado y **documentado por separado** de la cuenta de Portal
- [ ] Servicio de Windows "ArcGIS Server" en estado Running
- [ ] ArcGIS Server federado con Portal (estado "All systems operational" en Organización → Configuración → Servidores)
- [ ] Web Adaptor de Portal instalado y configurado (y, si aplica, Web Adaptor de Server, como instalación independiente)
- [ ] Archivo `hosts` actualizado en **todas** las máquinas/VMs que necesiten resolver nombres internos entre sí (bidireccional)
- [ ] Reglas de Firewall de Windows creadas en **ambos sentidos** (física ↔ VM) para los puertos: 6443 (Server), 7443 (Portal), 2443 (Data Store), 9220 y 9320 (Spatiotemporal), 9876/9840/9820/9850 (Relational, si aplica), 7143 (Velocity)
- [ ] Verificado que no exista un antivirus/EDR de terceros bloqueando puertos de forma independiente al Firewall de Windows (`Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntivirusProduct`)
- [ ] Conectividad bidireccional confirmada con `ping` y `Test-NetConnection` (`TcpTestSucceeded: True`) entre todas las máquinas del despliegue
- [ ] ArcGIS Data Store instalado seleccionando **todos los tipos de almacén necesarios** desde el instalador inicial (Relational, Object, Spatiotemporal), evitando depender de "Modificar" después
- [ ] Cuenta de servicio de Windows para Data Store (ej. `arcgis`) documentada por VM, sin asumir que debe coincidir con `portaladmin`/`siteadmin`
- [ ] `configuredatastore.bat` ejecutado con éxito para cada tipo de almacén requerido, usando las credenciales del **Primary Site Administrator del ArcGIS Server** (no las de Portal)
- [ ] Los 3 Data Stores (Relacional, Administrado Objeto, Espaciotemporal) validados en verde (✅) en Portal → Organización → Configuración → Servidores → Data Stores
- [ ] Licencia de ArcGIS Velocity asignada a la organización
- [ ] ArcGIS Velocity instalado, con requisitos de hardware verificados (≥16 GB RAM, ≥4 núcleos físicos, ≥20 GB disco)
- [ ] ArcGIS Velocity federado con Portal (URL sin errores de tipeo en `https://`, puerto 7143 verificado como accesible)
- [ ] Verificación final de que `obonilla.esrinosa.local/server/manager` (o equivalente) no muestra el error "Could not access any server machines"
- [ ] (Opcional, solo si se requiere transferir archivos grandes entre host y VM) Carpeta compartida de VirtualBox configurada con Guest Additions instaladas
- [ ] Nombres de máquina (hostnames) revisados/renombrados **antes** de federar, si se desea evitar nombres autogenerados tipo `WIN-XXXXXXXXXX`
- [ ] (Recomendado para producción, no aplicado en este entorno de pruebas) Separar Relational, Object, Spatiotemporal y Velocity en máquinas independientes, siguiendo la arquitectura de 3 niveles recomendada por Esri

---

# Anexo

## A. Notas sobre Anthropic/Claude y el contexto de este documento

Este documento fue generado a partir de una conversación de soporte técnico conducida en español, en la plataforma de chat de Claude (Anthropic), donde se combinaron respuestas basadas en conocimiento general de ArcGIS Enterprise con búsquedas web puntuales para verificar información sensible a cambios de versión (última versión de ArcGIS Enterprise, requisitos de ArcGIS Velocity, tabla oficial de puertos de ArcGIS Data Store).

## B. Comandos de referencia rápida (resumen consolidado)

```cmd
:: Verificar servicios
services.msc

:: Navegar al directorio de herramientas de Data Store
cd "C:\Program Files\ArcGIS\DataStore\tools"

:: Configurar Data Store tipo Spatiotemporal
configuredatastore.bat https://<hosting-server>:6443/arcgis <usuario_PSA> <password_PSA> C:\arcgisdatastore --stores spatiotemporal

:: Editar archivo hosts (como Administrador)
notepad C:\Windows\System32\drivers\etc\hosts

:: Ping de verificación
ping <hostname>

:: Reiniciar IIS (si aplica al Web Adaptor)
iisreset
```

```powershell
# Ver / modificar estado del firewall
netsh advfirewall show allprofiles state
netsh advfirewall set allprofiles state off   # Solo diagnóstico
netsh advfirewall set allprofiles state on

# Crear reglas de entrada necesarias
New-NetFirewallRule -DisplayName "ArcGIS Server 6443" -Direction Inbound -LocalPort 6443 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS Portal 7443" -Direction Inbound -LocalPort 7443 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "Allow ICMP Ping" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS DataStore 2443" -Direction Inbound -LocalPort 2443 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS DataStore Spatiotemporal 9220" -Direction Inbound -LocalPort 9220 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS DataStore Spatiotemporal 9320" -Direction Inbound -LocalPort 9320 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "ArcGIS Velocity 7143" -Direction Inbound -LocalPort 7143 -Protocol TCP -Action Allow

# Verificar conectividad
Test-NetConnection -ComputerName <host> -Port <puerto>

# Ver antivirus instalados
Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntivirusProduct | Select-Object displayName, productState

# Ver perfil de red activo
Get-NetConnectionProfile
Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
```

## C. Glosario rápido

| Término | Significado |
|---|---|
| PSA | Primary Site Administrator — cuenta administrativa de un sitio de ArcGIS Server |
| Hosting Server | El ArcGIS Server designado para alojar servicios/feature layers hospedados de Portal |
| Web Adaptor | Componente que expone Portal/Server públicamente (generalmente vía IIS), actuando como reverse-proxy |
| Data Store | Componente de almacenamiento de ArcGIS Enterprise; puede ser Relational, Tile Cache, Spatiotemporal, Object o Graph |
| Federación | Proceso de vincular un sitio de ArcGIS Server con un Portal para lograr inicio de sesión único y publicación de servicios |
| PSA vs. portaladmin vs. cuenta de Windows | Tres cuentas distintas: administración de Server, administración de Portal, y cuenta de sistema operativo para el servicio de Data Store, respectivamente |

## D. Información explícitamente no especificada en la conversación

- Versión exacta de ArcGIS Enterprise instalada originalmente por el usuario antes de las actualizaciones (solo se vio un número de build "20112" en una captura temprana).
- Caso de uso de negocio final para ArcGIS Velocity (tipo de feeds/sensores a conectar).
- Confirmación final de si se eliminó o no la carpeta `C:\arcgisdatastore` residual en el entorno de la compañera.
- Causa raíz confirmada del error *"Could not access any server machines"* en el Server Manager del usuario.
- Resultado final de la federación de ArcGIS Velocity con Portal (la conversación concluye con este punto en progreso).
- Justificación explícita de por qué Portal y ArcGIS Server se instalaron en la máquina física en lugar de una VM.
- Versión exacta de ArcGIS Velocity instalada (no se mostró un número de versión específico para este componente en las capturas compartidas).

---

*Fin del documento.*

