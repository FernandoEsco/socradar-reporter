# SOCRadar Reporter

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![Tests](https://img.shields.io/badge/tests-pytest-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

**Herramienta de línea de comandos que automatiza la generación de reportes semanales de Threat Intelligence a partir de la API de [SOCRadar](https://socradar.io).**

Con un solo comando consulta cuatro fuentes de inteligencia (cuentas que suplantan a la marca, credenciales de botnets, fugas de datos de empleados y dominios de phishing) y entrega un Excel por cada una, ya filtrado por fechas y con formato. Así el analista puede revisarlo, clasificarlo y pasarlo al reporte para el cliente.

> En un SOC, preparar este reporte a mano significa entrar a varios módulos de la plataforma, filtrar por fechas, exportar y dar formato. Con esta herramienta el proceso tarda unos segundos y siempre sale igual.

---

## Contenido

- [Qué hace](#qué-hace)
- [Reportes incluidos](#reportes-incluidos)
- [Arquitectura](#arquitectura)
- [Instalación](#instalación)
- [Configuración](#configuración)
- [Uso](#uso)
- [Ejemplo de salida](#ejemplo-de-salida)
- [Seguridad y manejo de datos](#seguridad-y-manejo-de-datos)
- [Pruebas](#pruebas)
- [Cómo añadir un reporte nuevo](#cómo-añadir-un-reporte-nuevo)
- [Limitaciones conocidas](#limitaciones-conocidas)

---

## Qué hace

- **Consulta cuatro endpoints de SOCRadar** con una sola sesión HTTP, con timeout y reintentos automáticos ante errores `429` y `5xx`.
- **Calcula el periodo de forma automática.** Por defecto toma los últimos 7 días completos, terminando ayer. También acepta un rango manual (`--desde` / `--hasta`).
- **Normaliza la respuesta de la API.** Parsea fechas en distintos formatos, rellena valores vacíos, quita duplicados y ordena por fecha.
- **Exporta a Excel con formato.** Encabezados con estilo, filtros, primera fila fija, ancho de columnas ajustado e hipervínculos clicables.
- **Protege las credenciales.** La API key se lee del entorno y nunca aparece en logs ni en mensajes de error.
- **Avisa cuando algo cambia.** Si la API cambia su estructura, la herramienta muestra un error o una advertencia clara en vez de generar un Excel vacío sin decir nada.

## Reportes incluidos

| Reporte | Comando | Endpoint de SOCRadar | Contenido del Excel |
|---|---|---|---|
| **Impersonating Accounts** | `impersonating-accounts` | `brand-protection/impersonating-accounts/v2` | Una hoja por red social con fecha, cuenta (con hipervínculo al perfil), status y una columna de **Severidad** que llena el analista. |
| **Botnet** | `botnet` | `leaks/.../latest` (`leak_type=BOTNET MARKET`) | Credenciales robadas por *infostealers*: fecha, usuario y URL del servicio comprometido. |
| **PII Exposure** | `pii-exposure` | `leaks/.../latest` | Correos de empleados que aparecen en filtraciones, con la fecha del hallazgo. |
| **Impersonating Domains** | `impersonating-domains` | `brand-protection/impersonating-domains/v2` | Dominios que imitan a la marca: *phishing score*, estado del sitio, registro MX y avance del *takedown*. Como el endpoint no acepta fechas, el filtrado se hace localmente. |

## Arquitectura

```
socradar-reporter/
├── socradar_reporter/
│   ├── cli.py              # Argumentos (argparse), orquestación y resumen final
│   ├── config.py           # Lectura de variables de entorno / .env
│   ├── client.py           # Cliente HTTP: sesión, reintentos, timeout, errores sin la key
│   ├── fechas.py           # Cálculo de rangos y parseo de fechas (compartido)
│   ├── excel.py            # Exportación y formato de los archivos .xlsx
│   └── reportes/
│       ├── base.py                     # Modelo Reporte y extracción robusta del JSON
│       ├── impersonating_accounts.py
│       ├── botnet.py
│       ├── pii_exposure.py
│       └── impersonating_domains.py
├── tests/                  # Pruebas con pytest (sin red, solo datos ficticios)
├── .github/workflows/      # CI: corre las pruebas en cada push
├── .env.example            # Plantilla de configuración
└── pyproject.toml
```

Cada capa tiene una sola responsabilidad:

```
CLI ──► Reporte.generar(client, rango) ──► SOCRadarClient.get() ──► API
              │
              └──► Reporte (hojas = DataFrames) ──► excel.exportar() ──► .xlsx
```

Los módulos de reporte **solo transforman datos**: no saben nada de HTTP ni de archivos. Por eso se pueden probar con un cliente simulado.

## Instalación

Requiere **Python 3.10 o superior**.

```bash
git clone https://github.com/FernandoEsco/socradar-reporter.git
cd socradar-reporter

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -e .                 # instala el comando `socradar-reporter`
# o bien: pip install -r requirements.txt
```

## Configuración

Las credenciales **no se escriben en el código**. Copia la plantilla y complétala:

```bash
cp .env.example .env
```

```dotenv
SOCRADAR_API_KEY=tu_api_key_aqui
SOCRADAR_COMPANY_ID=tu_company_id_aqui
```

| Variable | Obligatoria | Descripción |
|---|---|---|
| `SOCRADAR_API_KEY` | Sí | API key de la plataforma SOCRadar |
| `SOCRADAR_COMPANY_ID` | Sí | ID de la compañía monitoreada (también se puede pasar con `--company-id`) |
| `SOCRADAR_BASE_URL` | No | URL base de la API (por defecto `https://platform.socradar.com/api`) |
| `SOCRADAR_TIMEOUT` | No | Timeout por petición en segundos (por defecto `30`) |

`.env` está en `.gitignore`, así que no se subirá al repositorio por accidente.

## Uso

```bash
# Los 4 reportes de los últimos 7 días (terminando ayer)
socradar-reporter

# Solo algunos reportes
socradar-reporter -r botnet pii-exposure

# Últimos 30 días
socradar-reporter -d 30

# Rango específico, con una etiqueta en el nombre del archivo y otra carpeta de salida
socradar-reporter --desde 2026-09-01 --hasta 2026-09-30 -e "cliente-a" -o ./entregables

# Otra compañía sin modificar el .env
socradar-reporter --company-id 12345

# Modo detallado para depurar (la API key nunca se imprime)
socradar-reporter -v
```

También funciona sin instalar el paquete: `python -m socradar_reporter [opciones]`.

| Opción | Descripción | Por defecto |
|---|---|---|
| `-r, --reportes` | Reportes a generar (`impersonating-accounts`, `botnet`, `pii-exposure`, `impersonating-domains`, `all`) | `all` |
| `-d, --dias` | Días completos hacia atrás, terminando ayer | `7` |
| `--desde` / `--hasta` | Rango explícito `YYYY-MM-DD` (se usan juntas) | — |
| `-o, --salida` | Carpeta de salida | `./reportes` |
| `-e, --etiqueta` | Texto que se agrega al nombre de los archivos | — |
| `-c, --company-id` | Sobrescribe `SOCRADAR_COMPANY_ID` | — |
| `--espera` | Segundos entre llamadas (rate limit) | `1` |
| `-v, --verbose` | Muestra el detalle de las peticiones | — |

**Códigos de salida:** `0` = todo bien · `1` = falló algún reporte · `2` = error de configuración o de argumentos. Así se puede usar en `cron` o en un programador de tareas.

## Ejemplo de salida

```text
$ socradar-reporter -e demo
INFO    Periodo: 2026-09-24 a 2026-09-30 | Reportes: impersonating-accounts, botnet, pii-exposure, impersonating-domains

=== RESUMEN ===
Impersonating Accounts   OK      2 registros → reportes/impersonating_accounts_demo_2026-09-24_2026-09-30.xlsx
Botnet                   OK      1 registros → reportes/botnet_demo_2026-09-24_2026-09-30.xlsx
PII Exposure             OK      1 registros → reportes/pii_exposure_demo_2026-09-24_2026-09-30.xlsx
Impersonating Domains    OK      1 registros → reportes/impersonating_domains_demo_2026-09-24_2026-09-30.xlsx
```

Ejemplo de la hoja *Dominios* (datos ficticios):

| Fecha | Dominio | Phishing Score | Estado del sitio | DNS MX | Takedown |
|---|---|---|---|---|---|
| 2026-09-26 | acme-login.test | 87 | Active | NA | Requested |

Si un reporte falla (credenciales inválidas, timeout, etc.), los demás se generan de todos modos y el resumen indica cuál falló y por qué.

## Seguridad y manejo de datos

Esta herramienta maneja **credenciales** y **datos personales (PII)**. Por eso se diseñó con estas medidas:

- **Nada sensible en el código.** La API key y el company ID se leen de variables de entorno.
- **La key no se filtra en los errores.** La API recibe la key como parámetro de la URL, así que la herramienta nunca imprime URLs completas, oculta la key si la API la devuelve en una respuesta y silencia los logs de `urllib3`.
- **Los reportes no se suben al repositorio.** El `.gitignore` excluye `reportes/`, `*.xlsx` y `*.csv`, porque esos archivos contienen correos y usuarios comprometidos.
- **Las pruebas solo usan datos ficticios.** Usan dominios `.test` y nunca se conectan a la API real.

> ⚠️ Los Excel generados contienen datos personales. Guárdalos y compártelos según la política de tu organización y la normativa que aplique (LFPDPPP, RGPD, etc.).

## Pruebas

```bash
pip install -e ".[dev]"
pytest -q
```

Las pruebas cubren el cálculo de fechas, el parseo de cada reporte, el filtrado de dominios por rango, la agrupación por plataforma, los hipervínculos en Excel, los nombres de hoja válidos, la CLI y que la **API key nunca aparezca en un error**. GitHub Actions las corre automáticamente en cada push con Python 3.10 y 3.12.

## Cómo añadir un reporte nuevo

1. Crea `socradar_reporter/reportes/mi_reporte.py` con `TITULO`, `COLUMNAS` y una función `generar(client, rango) -> Reporte`.
2. Regístralo en `socradar_reporter/reportes/__init__.py`.
3. Para que una columna sea un hipervínculo, agrega una columna oculta `_link_<NombreColumna>` con la URL. El exportador se encarga del resto.

## Limitaciones conocidas

- **Paginación:** la herramienta procesa los registros que la API devuelve en una sola respuesta. Si una consulta trae muchos resultados y el endpoint pagina, habría que agregar el manejo de páginas en `client.py`.
- Los nombres de los campos dependen de la versión actual de la API de SOCRadar (`v2` en *brand protection*).

## Aviso

Proyecto independiente, sin relación con SOCRadar ni avalado por la empresa. Necesitas una cuenta y una API key propias, y su uso está sujeto a los términos de servicio de SOCRadar.

## Licencia
Fernando Escobar
[MIT](LICENSE)
