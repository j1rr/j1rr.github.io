# 🎬 Self-Hosted Streaming Portfolio

Portafolio personal que además funciona como demo en vivo de un servidor de streaming
auto-hospedado. En vez de solo *contar* que sé usar Docker, Cloudflare y APIs REST, este
proyecto lo demuestra: es una biblioteca de Jellyfin real, corriendo en mi propia máquina,
expuesta a internet de forma segura, y consumida en vivo desde esta misma página.

**🔗 Demo en vivo:** https://death-zip.github.io
**🔗 Servidor Jellyfin:** https://media.jarr.cc.cd
**🔗 CV interactivo:** https://death-zip.github.io/cv.html

---

## Cómo está organizado el sitio

Todo el sitio es un único flujo, no páginas sueltas:

```
index.html (portafolio)
   │
   ├── Hero → botón "Ver CV" ────────────► cv.html
   │                                          │
   │                                          ├── botón "Guardar como PDF"
   │                                          │   (usa los estilos @media print
   │                                          │    ya definidos, sin backend)
   │                                          │
   │                                          └── botón "← Portafolio" (regresa)
   │
   └── Sección "Proyecto destacado" ────────► consume en vivo la API de
                                               media.jarr.cc.cd (Jellyfin)
```

- **`index.html`** es la puerta de entrada: presentación, educación, certificaciones y el
  proyecto en vivo.
- **`cv.html`** es un CV completo, bilingüe (ES/EN), pensado para leerse en pantalla o
  imprimirse directo a PDF desde el navegador (`Ctrl+P` o el botón dedicado) — sin necesitar
  generar y mantener un PDF estático aparte.
- El **proyecto destacado** dentro del portafolio no es una captura de pantalla ni una
  descripción: es la biblioteca real de mi servidor Jellyfin, consumida en vivo por el
  navegador de quien visite la página.

Ambos documentos comparten paleta de colores, tipografía (Inter) y tono — se navegan como
una sola experiencia, no como archivos independientes.

---

## Por qué existe esto

La mayoría de portafolios de estudiantes son una lista de proyectos de clase. Quería algo
distinto: un sistema real, con partes que fallan de verdad y que hay que debuggear de verdad.
Este README documenta no solo qué hice, sino los problemas concretos que tuve que resolver
en el camino — porque esa parte es la que demuestra la habilidad, no el resultado final.

## Arquitectura

```
┌─────────────────┐      HTTPS      ┌──────────────────┐
│  GitHub Pages    │ ───────────────▶│  Cloudflare Edge  │
│  (este portafolio)│                │  (Tunnel + DNS)   │
└─────────────────┘                 └─────────┬────────┘
                                               │ túnel cifrado
                                               │ (sin puertos abiertos
                                               │  en el router)
                                     ┌─────────▼────────┐
                                     │  WSL2 (Ubuntu)    │
                                     │  ┌──────────────┐ │
                                     │  │ nginx (CORS  │ │
                                     │  │  proxy)      │ │
                                     │  └──────┬───────┘ │
                                     │         │         │
                                     │  ┌──────▼───────┐ │
                                     │  │  Jellyfin    │ │
                                     │  │  (Docker)    │ │
                                     │  └──────────────┘ │
                                     └────────────────────┘
```

**Flujo de una petición:** el navegador del visitante pide datos a `media.jarr.cc.cd` →
Cloudflare enruta por el túnel cifrado hasta WSL2 → nginx agrega los headers CORS
correctos y reenvía a Jellyfin → Jellyfin responde con el JSON de la biblioteca →
el portafolio lo renderiza en tiempo real.

## Stack técnico

| Capa | Tecnología | Por qué |
|---|---|---|
| Servidor multimedia | Jellyfin | Alternativa open-source a Plex, expone una API REST completa |
| Contenerización | Docker + Docker Compose | Aislamiento, reproducibilidad, `restart: unless-stopped` para resiliencia |
| Proxy / CORS | nginx (Alpine) | Jellyfin no está pensado para consumirse desde otro origen; nginx resuelve eso |
| Exposición a internet | Cloudflare Tunnel | Sin abrir puertos en el router, tráfico cifrado de punta a punta |
| Persistencia de servicios | systemd (servicio + timer) | El túnel y el watchdog sobreviven a reinicios sin depender de una terminal abierta |
| Frontend | HTML/CSS/JS vanilla | Sin build step: consume la API de Jellyfin directo con `fetch` |
| Hosting del sitio | GitHub Pages | Gratuito, siempre disponible independientemente de si mi PC está prendida |

## Problemas reales que resolví

Esta sección es, honestamente, la parte más importante del proyecto — cualquiera puede
seguir un tutorial. Esto es lo que pasó cuando el tutorial no cubría el problema:

### 1. CORS con headers duplicados
Al poner nginx delante de Jellyfin para agregar headers CORS, el navegador seguía
rechazando las peticiones con un error confuso:
```
The 'Access-Control-Allow-Origin' header contains multiple values '*, https://death-zip.github.io'
```
Jellyfin ya manda su propio header `Access-Control-Allow-Origin: *` — nginx agregaba el suyo
encima, y el navegador ve dos valores en el mismo header y rechaza ambos. La solución fue usar
`proxy_hide_header` en nginx para eliminar los headers CORS originales de Jellyfin antes de
agregar los correctos.

### 2. Cloudflare Workers bloqueado en la interfaz web
El plan inicial era usar un Cloudflare Worker como proxy CORS, pero la interfaz web tenía la
opción "Start with Hello World!" deshabilitada sin explicación clara. En vez de perder tiempo
en eso, cambié de estrategia: monté el proxy yo mismo con nginx dentro de mi propio
docker-compose, con control total sobre la configuración.

### 3. WSL2 se suspende y todo se cae con él
Cuando cerraba todas las terminales de WSL, Windows lo suspendía después de un rato — y con
él, Docker y el túnel de Cloudflare. La solución tiene dos partes:
- Una tarea programada de Windows (`Register-ScheduledTask`) que mantiene WSL vivo en segundo
  plano al iniciar sesión, sin necesidad de ninguna ventana abierta.
- Cloudflared instalado como **servicio systemd** (no como proceso manual), para que sobreviva
  a la instancia de WSL independientemente de si hay una terminal abierta.

### 4. Recuperación automática ante fallos
En vez de revisar manualmente si todo sigue corriendo, hay un script (`reactivar.sh`) + un
timer de systemd que corre cada 2 minutos: verifica el túnel, relanza los contenedores de
Docker si se cayeron, y deja un log. Es un watchdog mínimo, pero significa que el sistema se
autorepara sin que yo tenga que estar pendiente.

## Seguridad: decisiones y pendientes

- La API key de Jellyfin usada por el frontend es de **solo consumo de biblioteca**, no la
  de administrador.
- **Pendiente conocido:** la key vive en el HTML público (visible en "ver código fuente").
  Es aceptable para un portafolio de demostración con datos no sensibles, pero el siguiente
  paso natural sería moverla detrás de una función serverless (Cloudflare Worker o similar)
  para que el navegador nunca la vea directamente. Lo dejo documentado aquí a propósito:
  reconocer el límite de una solución también forma parte de diseñar bien.
- El túnel de Cloudflare significa que no hay ningún puerto abierto directamente en el router
  de casa — toda la superficie de ataque pasa por la red de Cloudflare.

## Estructura del repo

```
.
├── index.html            # Portafolio + cliente que consume la API de Jellyfin
├── cv.html               # CV interactivo, bilingüe, imprimible a PDF
├── 1773352902571.jpg     # Foto de perfil usada en portafolio y CV
├── nginx-cors.conf       # Configuración del proxy CORS
├── docker-compose.yml    # Definición de los servicios (jellyfin + cors-proxy)
├── reactivar.sh          # Script de verificación/recuperación del stack
└── README.md
```

*(nginx-cors.conf y docker-compose.yml viven en el servidor, no en este repo de GitHub Pages;
se documentan aquí por completitud del proyecto.)*

## Cómo correrlo tú mismo

```bash
# 1. Levanta el stack (docker-compose + nginx config)
cd ~/jellyfin-server
docker compose up -d

# 2. Instala cloudflared como servicio
sudo cloudflared service install
sudo systemctl enable --now cloudflared

# 3. Watchdog automático (opcional pero recomendado)
sudo systemctl enable --now jellyfin-watchdog.timer
```

## Roadmap / próximos pasos

- [ ] Mover la API key detrás de un backend ligero para no exponerla en el cliente
- [ ] Agregar métricas básicas (uptime del túnel, uso de CPU/memoria de los contenedores)
- [ ] Migrar el proxy CORS de nginx a un Cloudflare Worker cuando resuelva el bug de la interfaz
- [ ] Tests automatizados del pipeline de despliegue

## Sobre mí

Estudiante de Ingeniería en Desarrollo de Software en Universidad Tecmilenio, especializándome
en DevOps. Buscando mi primera oportunidad junior en Cloud Support o DevOps.

- Portafolio: https://death-zip.github.io
- LinkedIn: https://www.linkedin.com/in/juan-antonio-rivera-rodriguez-25aa56143/
- Email: riverarodriguezjuanantonio@gmail.com