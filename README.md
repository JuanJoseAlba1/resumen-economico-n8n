# ☕ Resumen Económico Diario: Automatización con n8n

Flujo de automatización que **cada mañana a las 7:00** recopila noticias económicas de dos fuentes de referencia, filtra únicamente lo relevante para la toma de decisiones y entrega un **resumen HTML listo para leer** en tu bandeja de entrada.

> **Impacto:** ahorra aproximadamente **10 horas al mes** frente a la revisión manual de fuentes, y garantiza que la información llegue a tiempo y con criterios de selección consistentes.

---

## 📌 Tabla de contenidos

- [El problema](#-el-problema)
- [La solución](#-la-solución)
- [Arquitectura del flujo](#-arquitectura-del-flujo)
- [Capturas](#-capturas)
- [Requisitos](#-requisitos)
- [Instalación y puesta en marcha](#-instalación-y-puesta-en-marcha)
- [Configuración de credenciales de Gmail](#-configuración-de-credenciales-de-gmail)
- [Personalización](#-personalización)
- [Solución de problemas](#-solución-de-problemas)
- [Sobre el autor](#-sobre-el-autor)

---

## 🎯 El problema

Mantenerse al día con indicadores y noticias macroeconómicas exige revisar varias fuentes a diario. Es un trabajo repetitivo, poco escalable y propenso a omisiones. Cada minuto invertido en buscar y descartar información irrelevante es un minuto que no se dedica al análisis.

## ✅ La solución

Un workflow en **n8n** que automatiza todo el ciclo de vigilancia informativa:

| Etapa | Descripción |
|---|---|
| **Disparo** | Ejecución programada todos los días a las 07:00 |
| **Captura** | Lectura en paralelo de 3 fuentes RSS: BBC Mundo, El Economista y FMI |
| **Consolidación** | Unificación de todas las noticias en un único flujo de datos |
| **Filtrado** | Selección por palabras clave: *inflación, PIB, tasas, empleo, recesión* |
| **Formato** | Generación de un correo HTML limpio, con título enlazado y fuente por noticia |
| **Entrega** | Envío automático por Gmail (OAuth2) |

## 🧭 Arquitectura del flujo

```text
                    ┌─────────────────┐
                    │ Schedule Trigger│  ⏰ Todos los días 07:00
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼                             ▼
        ┌──────────┐                  ┌──────────┐
        │ RSS Read │                  │ RSS Read │
        │BBC Mundo │                  │ Economist│
        └────┬─────┘                  └────┬─────┘
             │                             │
             └──────────────┼──────────────┘
                            ▼
                     ┌────────────┐
                     │   Merge    │  Modo: append
                     └──────┬─────┘
                            ▼
                     ┌────────────┐
                     │   Filter   │  Palabras clave (regex)
                     └──────┬─────┘
                            ▼
                  ┌───────────────────┐
                  │ Code in JavaScript│  Genera el HTML del correo
                  └─────────┬─────────┘
                            ▼
                     ┌────────────┐
                     │   Gmail    │  Send a message
                     └────────────┘
```

## 🖼️ Capturas

**Vista general del workflow en n8n**

![Workflow completo](docs/img/workflow.png)

**Configuración del filtro por palabras clave**

![Nodo Filter](docs/img/filter.png)

**Ejemplo del correo recibido**

![Correo resultante](docs/img/email.png)

## 📋 Requisitos

- **Docker** (Docker Desktop en Windows/macOS, o Docker Engine en Linux) para ejecutar n8n.
- **Cuenta de Gmail** desde la que se enviará el resumen.
- **Proyecto en Google Cloud** con la **Gmail API** habilitada y credenciales OAuth 2.0 (ver [configuración](#-configuración-de-credenciales-de-gmail)).
- Navegador web para acceder a la interfaz de n8n en `http://localhost:5678`.

## 🚀 Instalación y puesta en marcha

### 1. Clonar el repositorio

```bash
git clone https://github.com/<tu-usuario>/<nombre-del-repositorio>.git
cd <nombre-del-repositorio>
```

### 2. Levantar n8n con Docker

Opción rápida con un solo comando:

```bash
docker run -d --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Europe/Madrid" \
  -e TZ="Europe/Madrid" \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

O con **Docker Compose** (`docker-compose.yml`):

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=Europe/Madrid
      - TZ=Europe/Madrid
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

```bash
docker compose up -d
```

> ⚠️ **Zona horaria:** define `GENERIC_TIMEZONE` y `TZ` con tu zona. De lo contrario, el disparo de las 07:00 se ejecutará en la hora del contenedor (normalmente UTC).

Abre **http://localhost:5678** y crea tu cuenta de administrador local.

### 3. Importar el workflow

1. En n8n, ve a **Overview** y pulsa **Create workflow** (o abre un workflow vacío).
2. Menú **⋯** (arriba a la derecha) → **Import from file…**
3. Selecciona el archivo `workflow/resumen-economico-diario.json` de este repositorio.
4. El flujo aparecerá completo en el lienzo.

### 4. Revisar las fuentes RSS

Abre cada nodo **RSS Read** y confirma que la **URL** del feed corresponde a la fuente que deseas (BBC Mundo, El Economista, FMI).

### 5. Conectar Gmail

Sigue la sección siguiente y asigna la credencial al nodo **Send a message**. Indica también en ese nodo el **destinatario** del resumen.

### 6. Probar y publicar

1. Pulsa **Execute** para lanzar una ejecución manual y verifica que llega el correo.
2. Pulsa **Publish** para activar la ejecución programada.

> ℹ️ El workflow solo se ejecuta si **n8n está en funcionamiento** a la hora programada. Para uso continuo, despliega el contenedor en un servidor o equipo que permanezca encendido (con `restart: unless-stopped`).

## 🔐 Configuración de credenciales de Gmail

1. Accede a [Google Cloud Console](https://console.cloud.google.com) y crea un proyecto.
2. **APIs y servicios → Biblioteca** → habilita **Gmail API**.
3. **Pantalla de consentimiento de OAuth**:
   - Tipo de usuario: **Externo**.
   - Añade tu correo en **Usuarios de prueba** (imprescindible; de lo contrario Google devuelve el error `403: access_denied`).
4. **Credenciales → Crear credenciales → ID de cliente de OAuth**:
   - Tipo: **Aplicación web**.
   - **URI de redireccionamiento autorizado:** `http://localhost:5678/rest/oauth2-credential/callback`
5. Copia el **ID de cliente** y el **Secreto de cliente** en la credencial *Gmail OAuth2* de n8n.
6. Pulsa **Sign in with Google** y autoriza el acceso.

> 🔒 **Seguridad:** nunca subas al repositorio tus Client ID, Client Secret ni tokens. Las credenciales se almacenan cifradas dentro de n8n y **no** se incluyen en el archivo `.json` exportado.

## 🛠️ Personalización

### Palabras clave del filtro

El nodo **Filter** utiliza una única condición booleana con una expresión regular. Edita la lista separada por `|` para ajustar los temas:

```javascript
{{ /inflaci[oó]n|pib|tasas|empleo|recesi[oó]n/i.test(($json.title || '') + ' ' + ($json.contentSnippet || '')) }}
```

La expresión no distingue mayúsculas de minúsculas y acepta las palabras con o sin tilde.

### Hora de ejecución

En el nodo **Schedule Trigger**, modifica la hora o añade días específicos (por ejemplo, solo laborables).

### Fuentes adicionales

Duplica un nodo **RSS Read**, configura la nueva URL y conéctalo a una entrada libre del nodo **Merge** (aumentando el número de entradas si es necesario).

### Diseño del correo

El nodo **Code in JavaScript** contiene la plantilla HTML con estilos *inline* (compatibles con la mayoría de clientes de correo). Puedes cambiar colores, tipografía o el título directamente en el código.

## 🩺 Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Error 403: access_denied` al conectar Gmail | Tu correo no está como usuario de prueba | Añádelo en *Pantalla de consentimiento → Usuarios de prueba* |
| El workflow se detiene tras el Filter | Ninguna noticia cumple las palabras clave | Revisa la pestaña **Discarded** del Filter y amplía la lista de términos |
| No llega el correo a las 7:00 | n8n apagado o zona horaria incorrecta | Verifica que el contenedor esté activo y que `GENERIC_TIMEZONE` esté definido |
| La conexión con Gmail caduca a los 7 días | La app de Google está en modo *Testing* | Vuelve a autorizar o pasa la app a *En producción* en la pantalla de consentimiento |

## 👤 Sobre el autor

**[Tu nombre]**: Ingeniero de Sistemas con experiencia en roles gerenciales y Máster en Organización Industrial, especializado en la **automatización de procesos con inteligencia artificial**. Diseño soluciones que reducen tareas manuales, estandarizan procesos y liberan tiempo para actividades de mayor valor.

- 💼 LinkedIn: [tu-perfil](https://www.linkedin.com/in/tu-perfil)
- ✉️ Contacto: tu-correo@ejemplo.com

---

<sub>Proyecto de uso personal y educativo. Respeta los términos de uso de cada fuente de noticias.</sub>
