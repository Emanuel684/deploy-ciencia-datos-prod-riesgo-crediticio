# Jenkins: local, pipeline y disparadores

Este repositorio incluye un [Jenkinsfile](Jenkinsfile) declarativo y [docker-compose.yml](docker-compose.yml) para levantar Jenkins en Docker. El `Jenkinsfile` hace **checkout con submódulos**, una **verificación de estructura de carpetas** y, al terminar, un **POST opcional a un webhook** si configuraste la credencial.

## Jenkins local (Docker) frente a Jenkins en la nube

| | Local (`docker compose up`) | Nube (VM, GKE, servicio administrado) |
|---|-----------------------------|----------------------------------------|
| Coste | Sin coste de infraestructura | Depende del proveedor |
| Disparo al hacer merge | Sin URL pública: usa **polling** del `Jenkinsfile` o un túnel (ngrok) para **webhook** | Webhook directo a tu Jenkins si tiene DNS/IP accesible |
| Mismo `Jenkinsfile` | Sí | Sí; solo cambian URL del controlador y credenciales |

## 1. Arrancar Jenkins local

Desde la raíz de este repo:

```bash
docker compose up -d
```

Abre `http://localhost:8090`, completa el asistente inicial e instala plugins sugeridos (incluye **Pipeline** y **Git**). Si hace falta, instala también **Credentials Binding** (suele venir en el bundle recomendado).

## 2. Credenciales de Git (clone + submódulo)

1. **Manage Jenkins → Credentials → (global) → Add Credentials**.
2. Crea credenciales para clonar el remoto del **repositorio deploy** (usuario/contraseña con token de GitHub, o SSH private key).
3. Si el submódulo `ci-p03-python-etl` es **privado** y no usa la misma autenticación, el checkout usa `parentCredentials: true` para reutilizar las credenciales del job en los submódulos.

## 3. Crear el job (Multibranch Pipeline recomendado)

1. **New Item** → nombre, por ejemplo `deploy-riesgo-crediticio` → **Multibranch Pipeline** → OK.
2. **Branch Sources → Add source → Git**:
   - **Repository URL**: URL HTTPS o SSH del repo que contiene este `Jenkinsfile` (raíz del deploy).
   - **Credentials**: las del paso anterior.
3. En **Behaviours**, añade si lo necesitas: **Discover branches** (p. ej. solo `master`).
4. **Build Configuration**: **by Jenkinsfile** → **Script Path**: `Jenkinsfile` (por defecto en la raíz).
5. Guarda y ejecuta **Scan Multibranch Pipeline Now** la primera vez.

Con esto Jenkins **clona** el repositorio en `workspace/<job>/<rama>/<build>` y ejecuta las etapas del `Jenkinsfile`.

## 4. Triggers automáticos al merge

### Opción A: Polling (ya definido en el `Jenkinsfile`)

Hay un bloque `pollSCM('H/15 * * * *')` (aprox. cada 15 minutos). No requiere que GitHub alcance tu máquina. Tras el primer build, Jenkins consultará el remoto según el cron.

### Opción B: Webhook desde GitHub (build casi inmediato)

1. Instala **GitHub Branch Source** (si usas Multibranch con GitHub API) o configura el **GitHub hook** clásico según tu tipo de job.
2. En el repositorio GitHub: **Settings → Webhooks → Add webhook**:
   - **Payload URL**: `http://<tu-jenkins-publico>:8090/github-webhook/` (ajusta host/puerto; detrás de proxy usa HTTPS).
   - **Content type**: `application/json`.
   - Eventos: **Just the push event** (o los que recomiende tu plugin).
3. Si Jenkins solo es local, usa un túnel (p. ej. ngrok) hacia el puerto **8090** o cambia a la opción A.

Para **Multibranch** con GitHub, el flujo habitual es webhook de GitHub → notificación al plugin → **re-scan** de ramas; el build corre cuando la rama escaneada coincide con tu configuración.

## 5. Notificación al terminar (webhook HTTP)

El `Jenkinsfile` envía un JSON al finalizar **si existe** la credencial de tipo **Secret text**:

- **ID de la credencial** (fijo en el pipeline): `notify-webhook-url`
- **Secret**: URL completa del webhook entrante (Slack Incoming Webhook, Microsoft Power Automate/Teams, Discord, etc.).

Pasos:

1. **Manage Jenkins → Credentials → Add Credentials** → **Secret text**.
2. **ID**: exactamente `notify-webhook-url`.
3. **Secret**: pega la URL del webhook.

Cuerpo enviado (ejemplo de campos): `status` (SUCCESS, FAILURE, UNSTABLE, etc.), `job`, `build`, `buildUrl`.

Si la credencial no existe o falla el `curl`, el build **no se invalida** por la notificación; solo verás un mensaje en el log.

### Alternativa por correo (no está en el `Jenkinsfile`)

Puedes añadir el plugin **Email Extension** y un bloque `emailext` en `post { always { ... } }` usando destinatarios por variable de entorno del job, sin guardar correos en Git.

## 6. Verificación manual del stage de estructura

El stage **Verify structure** comprueba rutas como `ci-p03-python-etl/mlops_pipeline/src` y archivos clave. Si el submódulo no se pudo inicializar (credenciales o `.gitmodules`), el build fallará ahí con un mensaje claro.

Para probar submódulos en tu máquina antes de Jenkins:

```bash
git submodule update --init --recursive
```

## 7. Configurar el pipeline desde la UI (paso a paso)

### Antes: desbloquear Jenkins

1. Primera vez: en la consola del contenedor aparece la contraseña de administrador (`docker compose logs jenkins` o `docker logs jenkins`).
2. Crea el usuario admin que quieras y en **Customize Jenkins** elige **Install suggested plugins**.

### Credenciales Git (una sola vez)

1. **Manage Jenkins → Security → Manage Credentials** (o **Credentials** desde el menú).
2. **System → Global credentials → Add Credentials**.
3. Tipo **Username with password** (recomendado para GitHub HTTPS): usuario de GitHub y **Personal Access Token** como contraseña (no tu contraseña de login).
4. **ID** sugerido: `github-deploy-repo` (luego lo eliges en el job). Guarda.

### Crear el Multibranch Pipeline

1. En el dashboard: **New Item**.
2. Nombre: por ejemplo `deploy-riesgo-crediticio`.
3. Tipo: **Multibranch Pipeline** → **OK**.
4. Pestaña **Branch Sources** → **Add source** → **Git**:
   - **Project Repository**: URL del repo **deploy** (el que tiene el `Jenkinsfile` en la raíz).
   - **Credentials**: el que creaste (`github-deploy-repo` o el que pusiste).
5. **Behaviours**: deja **Discover branches** (o añade filtro si tu versión lo permite) para incluir `master`.
6. **Build Configuration**: **Mode: by Jenkinsfile** — **Script Path**: `Jenkinsfile`.
7. **Save**.
8. En la página del job: **Scan Multibranch Pipeline Now**. Tras el escaneo debería aparecer la rama `master` (u otras).
9. Abre la rama → **Build Now** (o espera al siguiente ciclo de **pollSCM** del `Jenkinsfile`).

### Opción más simple: Pipeline “desde SCM” (una sola rama)

Si no quieres Multibranch:

1. **New Item** → **Pipeline** → OK.
2. Abajo, **Definition**: **Pipeline script from SCM**.
3. **SCM**: **Git** → misma URL y credenciales.
4. **Branch Specifier**: `*/master`.
5. **Script Path**: `Jenkinsfile` → Save → **Build Now**.

### Webhook (opcional)

Con job **Pipeline** clásico + Git: en el repo GitHub, **Settings → Webhooks**, URL `http://<host>:8090/github-webhook/` solo si Jenkins es alcanzable desde internet. Con Multibranch + **GitHub** como fuente, suele usarse el plugin **GitHub Branch Source** y la configuración de credenciales de la organización/repo.
