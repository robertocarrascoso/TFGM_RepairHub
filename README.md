# RepairHub

Aplicación web para la gestión de talleres de reparación de dispositivos electrónicos (móviles, tablets, portátiles, ordenadores, consolas y equipos similares).

RepairHub digitaliza el ciclo completo de una reparación —desde que el dispositivo llega al taller hasta su entrega al cliente— sustituyendo los cuadernos en papel, las hojas de cálculo y las pizarras que todavía se usan en el sector por una base de datos centralizada accesible desde cualquier navegador de la red local.

Proyecto desarrollado como TFG del **CFGM en Sistemas Microinformáticos y Redes (SMR)** — Las Naves Salesianos (Alcalá de Henares), curso 2025/2026.
Autor: **Roberto Carrascoso Jordán** · Tutor: Víctor Ramos.

> La documentación completa (memoria del proyecto) está en [`documentacion/`](documentacion/).

---

## Funciones principales

- **Gestión de clientes** con detección de duplicados (por teléfono o correo) y autocompletado.
- **Gestión de reparaciones** con código único automático `REP-AAAA-NNNNN` (contador independiente por año).
- **Máquina de estados** que controla las transiciones permitidas de cada reparación.
- **Presupuestos y precio final**, con registro de la aceptación/rechazo del cliente.
- **Generación de PDF** con resguardo para el cliente y etiqueta recortable para el dispositivo (un folio A4).
- **Búsqueda global en tiempo real** sobre reparaciones y clientes.
- **Exportación de listados a CSV**.
- **Panel de control (dashboard)** con indicadores y gráficos (Chart.js).
- **Autenticación con dos roles**: administrador y técnico, con contraseñas en hash.

---

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Backend | Python + Flask |
| Base de datos | MariaDB (conector `mysql-connector-python`) |
| Generación PDF | ReportLab |
| Frontend | HTML5 + Jinja2 + CSS3 + JavaScript vanilla |
| Gráficos | Chart.js |
| Servidor WSGI (producción) | Gunicorn |
| Proxy inverso (producción) | Nginx |
| Despliegue | systemd + rsync · Debian 13 |

---

## Estructura del proyecto

```
TFGM_RepairHub/
├── app.py                 # Backend completo (Flask, 22 rutas)
├── config.py              # Configuración (lee variables de entorno desde .env)
├── requirements.txt       # Dependencias Python
├── base-de-datos/
│   ├── schema.sql         # Creación de tablas
│   └── seed.py            # Datos de prueba (usuarios, clientes, reparaciones)
├── estaticos/
│   ├── css/style.css      # Hoja de estilos (tema dual claro/oscuro con variables CSS)
│   ├── js/main.js         # Lógica de cliente
│   └── img/               # Logos y favicon
├── plantillas/            # Plantillas Jinja2 (heredan de base.html)
├── utilidades/
│   └── pdf_generator.py   # Generación del resguardo + etiqueta en PDF
├── documentacion/         # Memoria del proyecto (PDF y DOCX)
└── presentacion/          # Presentación del proyecto
```

---

## Modelo de datos

Cinco tablas (codificación `utf8mb4`):

- **`clientes`** — datos de contacto. Se exige al menos uno entre teléfono o correo (validado en la aplicación).
- **`reparaciones`** — tabla principal; apunta a un cliente. Incluye código, dispositivo, avería, estado, presupuesto y precio final.
- **`historial_estados`** — registra cada cambio de estado (auditoría). El técnico se guarda como texto libre para no romper el historial si se borra un usuario.
- **`contadores`** — número correlativo de códigos por año (`year`, `ultimo_num`).
- **`usuarios`** — cuentas que pueden iniciar sesión. Roles `admin` y `tecnico`.

---

## Máquina de estados

```
Recibido → Diagnosticado → Presupuesto enviado ┬─ (acepta) → Presupuesto aceptado → Reparando ⇄ Esperando pieza
                                                │                                        └→ Listo → Entregado
                                                └─ (rechaza) ─────────────────────────────────────→ Entregado
```

- Pasar a **Presupuesto enviado** exige introducir el importe.
- Pasar a **Listo** pide el precio final (sugiere el presupuesto por defecto).
- El **rechazo** del presupuesto envía la reparación directamente a *Entregado*.
- **Entregado** es el estado final.

---

## Puesta en marcha (desarrollo)

Requisitos: Python 3, MariaDB.

```bash
# 1. Clonar el repositorio
git clone https://github.com/robertocarrascoso/TFGM_RepairHub.git
cd TFGM_RepairHub

# 2. Entorno virtual e instalación de dependencias
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. Crear la base de datos
mysql -u root -p < base-de-datos/schema.sql
```

Crear un archivo `.env` en la raíz (no se versiona):

```env
DB_HOST=localhost
DB_USER=repairhub_user
DB_PASSWORD=tu_contraseña
DB_NAME=repairhub
SECRET_KEY=una-clave-secreta
```

```bash
# 4. (Opcional) Cargar datos de prueba
python base-de-datos/seed.py

# 5. Arrancar el servidor de desarrollo
python app.py        # http://127.0.0.1:5000
```

### Usuarios de prueba (tras ejecutar `seed.py`)

| Rol | Correo | Contraseña |
|-----|--------|-----------|
| Administrador | `admin@repairhub.com` | `admin123` |
| Técnico | `roberto@repairhub.com` | `tecnico123` |

---

## Despliegue en producción

Arquitectura desplegada sobre una MV **Debian 13** en red local:

```
Navegador → Nginx (:80) ─┬─ archivos estáticos (servidos directamente, caché 1 semana)
                         └─ proxy inverso → Gunicorn (:5000) → Flask → MariaDB
```

- **Gunicorn** se registra como servicio **systemd** (dos workers, arranque automático y reinicio ante fallos).
- **Nginx** actúa de proxy inverso y sirve los estáticos.
- El nombre `repairhub.local` se resuelve añadiendo una entrada en el archivo de *hosts* de cada equipo.
- El despliegue se automatiza con un script que sincroniza los cambios con **rsync** y reinicia el servicio (~15 s).

---

## Seguridad

- Todas las rutas protegidas mediante decoradores (sesión requerida; rol admin para el panel de administración).
- Consultas SQL siempre **parametrizadas** (prevención de inyección SQL).
- Contraseñas almacenadas con **hash + sal** (Werkzeug).
- Sesión firmada con `SECRET_KEY`; mensaje de error genérico en el login para no revelar correos existentes.
- La aplicación se conecta a MariaDB con un usuario limitado, nunca con el administrador de la BD.

---

## Licencia

Proyecto académico (TFG SMR).

> Declaración de uso de IA: durante el desarrollo se utilizó el asistente Claude Code (Anthropic) como apoyo en tareas de programación y revisión de errores.
