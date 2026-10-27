# Odoo 20 – Entorno de desarrollo local (Windows)

Guía paso a paso para levantar este Odoo en una PC con Windows, desde cero hasta
tener la app corriendo con los módulos propios de `custom_addons`.

## Estructura de carpetas esperada

```
F:\Proyectos\Odoo\
├── odoo\              ← este repositorio (core de Odoo)
│   ├── odoo-bin
│   ├── addons\        ← módulos estándar de Odoo
│   └── requirements.txt
└── custom_addons\     ← módulos propios (cada subcarpeta es un módulo)
```

Todos los comandos de esta guía se ejecutan **desde la carpeta `odoo\`**, por eso la
ruta a los módulos propios es `..\custom_addons`.

---

## 1. Requisitos previos

| Herramienta | Versión | Notas |
|---|---|---|
| Python | 3.12 o 3.13 (64 bits) | Odoo 20 soporta 3.12 a 3.14. Marcar *Add python.exe to PATH* al instalar. |
| PostgreSQL | 13 o superior | Anotar la contraseña del usuario `postgres` que se define al instalar. |
| Git | cualquiera | Para clonar/actualizar el repo. |
| Microsoft C++ Build Tools | opcional | Solo si `pip` falla compilando algún paquete. |
| wkhtmltopdf | 0.12.6 | Opcional, necesario para imprimir reportes PDF. Agregarlo al PATH. |

Verificar que estén instalados:

```powershell
py -0                 # lista las versiones de Python instaladas
python --version
psql --version        # si no lo encuentra, agregar C:\Program Files\PostgreSQL\<ver>\bin al PATH
```

## 2. Crear el usuario de base de datos

Odoo no se conecta con el usuario `postgres`; hay que crear uno propio con permiso
para crear bases. Desde PowerShell:

```powershell
psql -U postgres -c "CREATE ROLE odoo_user WITH LOGIN CREATEDB PASSWORD 'claveXXX';"
```

> También se puede hacer desde pgAdmin: *Login/Group Roles → Create*, con
> *Can login* y *Create databases* activados.

## 3. Crear el entorno virtual

```powershell
cd F:\Proyectos\Odoo\odoo
py -3.12 -m venv venv
.\venv\Scripts\Activate.ps1
```

Si PowerShell bloquea la activación por la política de ejecución, correr una sola vez:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Con el entorno activo el prompt muestra `(venv)`. **Hay que activarlo cada vez que se
abre una terminal nueva** antes de correr Odoo.

## 4. Instalar dependencias

```powershell
python -m pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
```

Si además se usan módulos de `custom_addons` que declaran dependencias Python
externas (`external_dependencies` en su `__manifest__.py`), instalarlas también con `pip`.

## 5. Levantar Odoo por primera vez

```powershell
python odoo-bin -r odoo_user -w claveXXX --addons-path=addons,..\custom_addons -d odoo_DB
```

- `-r` / `-w`: usuario y contraseña de PostgreSQL creados en el paso 2.
- `--addons-path`: carpetas donde Odoo busca módulos, separadas por coma
  (primero los estándar `addons`, después los propios `..\custom_addons`).
- `-d`: nombre de la base. Si no existe, Odoo la crea e inicializa al arrancar.

Abrir **http://localhost:8069** e ingresar con `admin` / `admin` (cambiar la clave
después del primer ingreso).

Para detener el servidor: `Ctrl + C` en la terminal.

## 6. Módulos propios (`custom_addons`)

### Referenciar la carpeta

Odoo solo ve los módulos de `custom_addons` si la carpeta está en `--addons-path`
(o en `addons_path` del archivo de configuración, ver paso 7). Siempre incluir
`addons` **y** `..\custom_addons`.

Si se agregó un módulo nuevo a la carpeta y no aparece en *Aplicaciones*, activar el
modo desarrollador y usar *Aplicaciones → Actualizar lista de aplicaciones*, o
instalarlo directamente por consola como se muestra abajo.

### Instalar un módulo (`-i`)

```powershell
python odoo-bin -r odoo_user -w claveXXX --addons-path=addons,..\custom_addons -d odoo_DB -i custom-modul-name
```

### Actualizar un módulo (`-u`)

Necesario cada vez que se modifican modelos/campos Python, vistas XML, datos,
seguridad (`ir.model.access.csv`) o el `__manifest__.py`:

```powershell
python odoo-bin -r odoo_user -w claveXXX --addons-path=addons,..\custom_addons -d odoo_DB -u custom-modul-name
```

Tips:

- Varios módulos a la vez: separarlos con coma, sin espacios: `-u modulo1,modulo2`.
- Para instalar/actualizar y que el proceso termine solo (sin dejar el servidor
  levantado), agregar `--stop-after-init`.
- Durante el desarrollo, `--dev=xml,reload` recarga vistas XML y reinicia el server
  al guardar cambios Python, sin tener que correr `-u` por cada cambio de vista.

## 7. (Opcional) Archivo de configuración

Para no repetir todos los parámetros, se puede generar un `odoo.conf` con la opción
`-s` (guarda la configuración y sigue arrancando):

```powershell
python odoo-bin -r odoo_user -w claveXXX --addons-path=addons,..\custom_addons -d odoo_DB -c odoo.conf -s
```

A partir de ahí alcanza con:

```powershell
python odoo-bin -c odoo.conf                          # levantar
python odoo-bin -c odoo.conf -i custom-modul-name     # instalar
python odoo-bin -c odoo.conf -u custom-modul-name     # actualizar
```

> `odoo.conf` contiene la contraseña de la base: no subirlo al repositorio.

## Resumen rápido (día a día)

```powershell
cd F:\Proyectos\Odoo\odoo
.\venv\Scripts\Activate.ps1
python odoo-bin -r odoo_user -w claveXXX --addons-path=addons,..\custom_addons -d odoo_DB
```

## Problemas frecuentes

| Síntoma | Causa / solución |
|---|---|
| `FATAL: password authentication failed` | Usuario/clave de `-r`/`-w` incorrectos o el rol no existe (paso 2). |
| `permission denied to create database` | Al rol le falta `CREATEDB`: `ALTER ROLE odoo_user CREATEDB;` |
| `Module not found` / el módulo no aparece | Falta `..\custom_addons` en `--addons-path`, o la carpeta del módulo no tiene `__manifest__.py`. |
| `ModuleNotFoundError` de Python al arrancar | El venv no está activado o faltó `pip install -r requirements.txt`. |
| Los cambios en vistas/campos no se ven | Correr con `-u nombre_modulo` y refrescar el navegador con `Ctrl + F5`. |
| Puerto 8069 ocupado | Hay otra instancia corriendo; cerrarla o usar `--http-port=8070`. |

## Documentación oficial

- Instalación: https://www.odoo.com/documentation/master/administration/install/install.html
- Opciones de línea de comandos: https://www.odoo.com/documentation/master/developer/reference/cli.html
- Tutoriales de desarrollo: https://www.odoo.com/documentation/master/developer/tutorials.html
