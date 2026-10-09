# Prex Challenge

API REST de cuatro servicios integrada con GIPHY: Laravel 13.35.0, Passport 13.9.0, PHP 8.4.26 y MySQL 8.4.11. Arquitectura hexagonal con Dominio, Aplicación e Infraestructura.

## Instalación desde cero

Seguir los pasos en orden. Los comandos están preparados para Linux o WSL2 con Bash y una instalación nueva, sin `.env`, dependencias ni base de datos previa. Ejecutarlos con tu usuario habitual (UID/GID >= 1000), desde la carpeta del repositorio después de clonarlo.

### 1. Preparar los requisitos

Necesitás Git, Docker Engine operativo, Docker Compose v2 con soporte de `--wait`, acceso a Internet, Postman y una API Key de GIPHY válida. PHP, Composer y MySQL se ejecutan dentro de Docker; no necesitás instalarlos en tu máquina.

Comprobar que Docker está disponible:

```bash
git --version
docker --version
docker compose version
docker info
```

Los puertos locales `8080` (API) y `3306` (MySQL) deben estar libres. Si están ocupados, podrás cambiarlos en el paso 3.

### 2. Clonar el repositorio

```bash
git clone https://github.com/FacundoBustelo/PrexChallenge.git
cd PrexChallenge
```

Todos los comandos siguientes se ejecutan desde esta carpeta.

### 3. Copiar y completar la configuración

```bash
cp .env.example .env
id -u
id -g
```

Abrir `.env` y completar `DB_PASSWORD` y `MYSQL_ROOT_PASSWORD` con contraseñas distintas, y `GIPHY_API_KEY` con una clave válida. La clave GIPHY permanece en el servidor; no se configura en Postman. Si falta, el arranque informa una advertencia: login puede funcionar, pero búsqueda, consulta y favoritos requieren configurarla. No se consulta GIPHY durante la instalación.

`APP_UID` y `APP_GID` tienen valor inicial `1000`. Ajustarlos a los resultados de `id -u` e `id -g` de tu usuario de Linux/WSL antes de construir la imagen, para que Composer, Artisan y FPM puedan escribir con los permisos correctos. Usar un usuario habitual con UID/GID >= 1000 y una carpeta de trabajo que le pertenezca.

Revisar los puertos `HTTP_PORT=8080` y `DB_FORWARD_PORT=3306`. Si están ocupados, elegir puertos libres explícitamente; Compose no elige otro automáticamente. Al cambiar HTTP_PORT, actualizar también APP_URL, por ejemplo:

```dotenv
HTTP_PORT=8081
APP_URL=http://127.0.0.1:8081
DB_FORWARD_PORT=3307
```

Conservar `DB_HOST=mysql`, `DB_PORT=3306`, `CACHE_STORE=database` y `APP_ENV=local`. APP_KEY queda vacía en una instalación nueva.

### 4. Iniciar e instalar automáticamente

```bash
docker compose up -d --build --wait
docker compose ps -a
curl --fail http://127.0.0.1:8080/up
```

Usar el puerto elegido en el último comando. `init` espera a MySQL saludable, instala Composer desde `composer.lock`, limpia la caché de configuración, genera APP_KEY solo si está vacía, ejecuta migraciones pendientes y prepara Passport y el usuario local. `app` espera la finalización exitosa de `init`; el healthcheck de Nginx comprueba `/up` para que `--wait` espere una respuesta HTTP real.

El resultado esperado es `init` terminado con código **0**, y `app`, `nginx` y `mysql` disponibles, con Nginx y MySQL saludables. `/up` responde **200**. Ante una etapa fallida, `init` termina con error y bloquea el arranque inicial de la API. Consultar `docker compose logs init` para identificarla.

Credenciales de prueba creadas únicamente en `APP_ENV=local`:

| Campo | Valor |
|---|---|
| Email | `user@example.test` |
| Password | `contraseña-local` |

El seeder crea el usuario si no existe; no cambia contraseñas previas ni restaura usuarios borrados. Las claves Passport se generan únicamente si faltan ambas; un par incompleto detiene la instalación. El cliente personal activo compatible con `users` se reutiliza, sin crear duplicados. No ejecutar `install:api`, `migrate:fresh` ni regenerar claves en cada inicio.

### 5. Importar la colección y configurar Postman

En Postman, usar **Import** e importar estos dos archivos del repositorio:

- [Colección Prex API v1](docs/postman/Prex.postman_collection.json).
- [Environment Prex local](docs/postman/Local.postman_environment.json).

Seleccionar el environment **Prex local** y completar sus variables con los valores de tu instalación:

| Variable | Valor inicial |
|---|---|
| `base_url` | `http://127.0.0.1:8080` (o el puerto elegido), sin `/api/v1` |
| `email` | `user@example.test` |
| `password` | `contraseña-local` |
| `query` | `cats` |
| `alias` | `Mi GIF local` |
| `token` | Vacío; se completa tras el login |
| `user_id` | Vacío; se completa tras el login |
| `gif_id` | Vacío; copiarlo de la búsqueda en el siguiente paso |

Guardar los valores localmente. Evitar variables globales con los mismos nombres y mantener seleccionada **Prex local** al enviar las solicitudes.

### 6. Probar las APIs en orden

Abrir la colección **Prex API v1** y enviar cada solicitud con **Send**:

1. **01 Login**: espera **200** con `access_token`, `expires_in=1800` y `user.id`. El script guarda automáticamente el token en el environment. Copiar manualmente `user.id` a `user_id` y comprobar que ambas variables quedaron completas. El login usa **No Auth**; las demás solicitudes heredan **Bearer {{token}}** de la colección.
2. **02 Buscar GIFs**: espera **200** con una lista `data`. Copiar el `id` de uno de los resultados a la variable `gif_id`, conservando exactamente mayúsculas y minúsculas. Si la lista está vacía, cambiar `query` y repetir la búsqueda.
3. **03 Consultar GIF**: espera **200** y el mismo ID elegido en la búsqueda.
4. **04 Crear favorito**: revisar `gif_id`, `alias` y `user_id`; espera **201**. El favorito queda asociado al usuario autenticado.
5. **05 Comprobar duplicado**: enviar con los mismos valores; espera **409**, porque ese favorito ya existe para ese usuario.

El token vence a los **30 minutos**. Si una solicitud protegida devuelve **401** por vencimiento, volver a ejecutar el login. Para repetir una creación con **201**, elegir otro GIF; el mismo usuario/GIF vuelve a producir **409**. Guardar favoritos también consulta GIPHY, por lo que necesita una API Key válida y disponibilidad del proveedor.

Si password o alias contienen comillas, barras invertidas o saltos de línea, ajustar el escape JSON en el cuerpo de la solicitud. No exportar ni publicar el environment con contraseñas o tokens reales.

Con este recorrido quedan comprobados login, búsqueda, consulta y guardado de favoritos desde Postman.

## Configuración y persistencia

Después de editar configuración con la aplicación levantada, ejecutar `docker compose exec app php artisan config:clear`. Mantener `CACHE_STORE=database` para compartir el límite de login entre procesos.

Conservar APP_KEY, `storage/oauth-private.key`, `storage/oauth-public.key` y el cliente personal activo (`provider=users`, grant `personal_access`) de `oauth_clients`. Las claves persisten en el checkout y el entrypoint las deja en 600. El volumen `mysql_data` conserva usuarios, cliente, tokens, favoritos y auditoría al ejecutar `stop`, `down` o recrear contenedores. No regenerar claves al reiniciar. `down -v` elimina datos. Cambiar passwords de `.env` no rota credenciales de un volumen inicializado. Respaldar entorno, claves y base de forma segura.

Para detener y retirar los contenedores conservando la base de datos:

```bash
docker compose down
```

Para volver a iniciarlos con los mismos datos y configuración:

```bash
docker compose up -d --wait
```

Para eliminar también los volúmenes y la base de datos:

```bash
docker compose down -v
```

**`down -v` elimina usuarios, cliente Passport, tokens, favoritos y auditoría.** El `.env`, `vendor/` y las claves Passport permanecen en la carpeta del proyecto. El próximo `docker compose up -d --build --wait` reconstruye tablas, cliente personal y usuario local automáticamente, conservando APP_KEY y las claves Passport existentes. Los tokens y datos eliminados no se recuperan.

MySQL publica un puerto en `127.0.0.1` para clientes como DBViewer: usar `DB_FORWARD_PORT`, base `DB_DATABASE`, usuario `DB_USERNAME` y contraseña `DB_PASSWORD` de `.env`. Nginx publica solo el puerto HTTP y ejecuta `public/index.php`; FPM no publica puertos. El código y `vendor/` se montan desde la carpeta del proyecto para desarrollo. El entrypoint prepara los permisos de `storage/` y `bootstrap/cache/`; el servicio `init` prepara dependencias, claves y migraciones. Un `docker compose restart` solo reinicia procesos; usar `up -d --build --wait` para ejecutar la preparación tras cambios o un `down`.

`TRUSTED_PROXIES` admite IP/CIDR explícitos de proxies controlados, separados por comas; vacío no confía en ninguno. Se ignoran comodines, /0 y valores inválidos. El proxy debe reemplazar headers de forwarding recibidos del cliente. No se confía en X-Forwarded-Host.

### Recuperación de una inicialización interrumpida

Revisar `docker compose logs init`, corregir configuración, conexión o permisos y repetir `docker compose up -d --build --wait`. Las etapas completadas se conservan: Composer usa el lockfile, migrate ejecuta solo pendientes y el seeder y cliente personal no duplican registros. Una instalación fallida puede haber completado algunas migraciones; no se revierte borrando datos.

Si falta una sola clave Passport, restaurar **ambas claves del mismo respaldo** en `storage/oauth-private.key` y `storage/oauth-public.key`. Mantener su propietario APP_UID/APP_GID y permisos 600. No generar una clave aislada ni sobrescribir la restante. Si no hay respaldo, conservar la clave restante fuera de `storage`, respaldar la base y retirar el par de esa ubicación deliberadamente antes de repetir la inicialización; al generar un par nuevo los tokens anteriores dejan de ser válidos. No hay regeneración automática de un par incompleto.

Si cambiás las contraseñas MySQL de `.env` con un volumen existente, restaurar los valores anteriores o rotarlas explícitamente en MySQL. El arranque no modifica las credenciales de una base inicializada. Usar `down -v` solo para una reconstrucción deliberada que puede perder todos los datos.

Como alternativa opcional al `cp`, `scripts/init-env.php` genera contraseñas aleatorias y UID/GID y se niega a reemplazar `.env`. Si tenés PHP CLI con extensión POSIX, ejecutar `php scripts/init-env.php`; también se puede usar Docker antes de tener `.env`:

```bash
APP_UID=$(id -u) APP_GID=$(id -g) DB_PASSWORD=bootstrap MYSQL_ROOT_PASSWORD=bootstrap docker compose build app
DB_PASSWORD=bootstrap MYSQL_ROOT_PASSWORD=bootstrap docker compose run --rm --no-deps --user "$(id -u):$(id -g)" --entrypoint php app scripts/init-env.php
```

Los valores `bootstrap` solo permiten evaluar Compose en esos comandos; el script genera las contraseñas reales. Luego completar GIPHY y revisar los puertos antes del arranque automático.

## Arquitectura y servicios

`app/Domain` contiene valores, DTOs, errores y puertos `Authenticator`, `GifCatalog`, `FavoriteGifRepository` e `InteractionRepository`. `app/Application` contiene login, búsqueda, consulta, preparación/persistencia de favoritos y redacción/registro de interacciones. `app/Infrastructure` implementa Passport, HTTP GIPHY, repositorios Eloquent, controladores y coordinación transaccional. Los controladores delegan; no consultan la base directamente.

| Servicio | Método y ruta | Entradas | Éxito |
|---|---|---|---|
| Login público | POST `/api/v1/login` | JSON email, password | 200; token de 1800 segundos y user.id |
| Buscar | GET `/api/v1/gifs` | QUERY; LIMIT 1–50 (25); OFFSET 0–4999 (0) | 200, incluso vacío |
| Consultar | GET `/api/v1/gifs/{id}` | ID textual exacto | 200 |
| Favorito | POST `/api/v1/favorites` | JSON GIF_ID, ALIAS, USER_ID | 201 |

Los últimos tres requieren Bearer. Enviar Accept y, para POST, Content-Type `application/json`. QUERY: no vacío, hasta 50 caracteres; ALIAS: no vacío, hasta 100; IDs externos: hasta 128 caracteres. USER_ID debe coincidir con el actor autenticado. Duplicado por usuario/GIF: 409 sin cambiar alias. Validación 422; credenciales/token 401; propietario 403; ausencia 404; proveedor incompatible 502, indisponibilidad/cuota 503, timeout 504. Login: cinco peticiones/minuto/IP; sexta 429 con Retry-After. Timeouts GIPHY: conexión 2 s / total 5 s, sin reintentos.

Favoritos consulta GIPHY antes de la transacción. Dentro de ella inserta favorito, serializa respuesta y persiste auditoría sanitizada; tras commit marca el contexto confirmado. Fallos revierten ambas escrituras y permiten auditar el error final fuera de la transacción. Toda interacción `/api` se captura con actor verificado, servicio, request/response sanitizados, status, IP, fecha y UUID del servidor. No se registran headers/cookies; cuerpos opacos o streaming se omiten.

## Pruebas

Desde una instalación con vendor, Unit no arranca Laravel; Feature usa SQLite en memoria. Los tests bloquean HTTP externo y usan claves efímeras.

```bash
docker compose run --rm --no-deps app vendor/bin/phpunit --testsuite Unit --display-phpunit-notices
docker compose run --rm --no-deps -e DB_CONNECTION=sqlite -e DB_DATABASE=:memory: app vendor/bin/phpunit --display-phpunit-notices
docker compose run --rm --no-deps app vendor/bin/pint --test
# Proyecto exclusivo: nunca aplicar este override al proyecto principal.
docker compose -p prex-stage8-mysql -f compose.yaml -f compose.audit-test.yaml up -d --wait mysql
docker compose -p prex-stage8-mysql -f compose.yaml -f compose.audit-test.yaml run --rm app vendor/bin/phpunit -c phpunit.mysql.xml --display-phpunit-notices
docker compose -p prex-stage8-mysql -f compose.yaml -f compose.audit-test.yaml down
```

La suite MySQL fuerza `prex_audit_test` y rechaza otras bases antes de migrar. Comprueba integridad, comparación exacta, rollback y concurrencia real.

## Verificar instalación limpia

Usar una copia independiente sin `.env`, vendor, claves ni cachés. En esa raíz ejecutar los pasos 3 y 4 de la instalación anterior, agregando `-p prex-stage8-clean` a **todos** los comandos Compose. Tras copiar `.env.example` a `.env`, antes de iniciar MySQL, cambiar HTTP_PORT=18088, APP_URL=http://127.0.0.1:18088 y DB_FORWARD_PORT=13388 (o puertos libres). Completar GIPHY_API_KEY en esa copia para el recorrido real. No compartir caches ni checkout entre proyectos con entornos diferentes.

```bash
docker compose -p prex-stage8-clean exec app php artisan migrate:status
docker compose -p prex-stage8-clean exec app php scripts/verify-runtime.php write
docker compose -p prex-stage8-clean exec app php scripts/verify-login.php write
docker compose -p prex-stage8-clean down
docker compose -p prex-stage8-clean up -d --wait
# Dentro de los 30 minutos de validez del token:
docker compose -p prex-stage8-clean exec app php scripts/verify-runtime.php read
docker compose -p prex-stage8-clean exec app php scripts/verify-login.php read
docker compose -p prex-stage8-clean exec app php scripts/verify-runtime.php cleanup
docker compose -p prex-stage8-clean exec app php scripts/verify-login.php cleanup
```

Comprobar `/up` 200 y `/.env`, `/composer.json`, `/artisan`, `/vendor/autoload.php`, `/storage/oauth-private.key`, `/storage/logs/laravel.log` 404. Revisar `docker compose ps`: puertos publicados solo en loopback. Completar el recorrido de Postman de los pasos 5 y 6 y detener la copia con `down`, conservando el volumen.

## Diagramas y Postman

[Índice UML y DER](docs/diagrams/README.md): siete fuentes PlantUML y SVG, DER del proyecto (`erd`) y DER con tablas auxiliares (`erd-completo`), renderizador local fijado y relaciones físicas/lógicas diferenciadas.

[Guía Postman](docs/postman/README.md): detalles del recorrido de los pasos 5 y 6. La colección guarda automáticamente el token tras el login; user_id se completa manualmente; gif_id se completa manualmente desde la búsqueda. Exportaciones con token, IDs y password vacíos; la API Key permanece en el servidor.

## Diagnóstico y límites

Usar `docker compose logs --tail=100 init app nginx mysql`, `php artisan migrate:status`, `php artisan route:list`, `composer validate --strict` y `composer check-platform-reqs` mediante `docker compose exec app`. No compartir `.env`, salidas de `config:show`/Compose interpolado, tokens ni logs sin revisar. Login 500: comprobar migraciones, claves legibles y cliente personal users; 503 GIPHY: comprobar configuración de clave y respuesta del proveedor sin imprimirla. `/up` comprueba el framework, no disponibilidad de MySQL.

Diferencias acordadas respecto del PDF: IDs textuales de GIPHY frente a numéricos; login propio con tokens personales Passport como interpretación de OAuth2.0, sin Authorization Code + PKCE. Las restricciones del proveedor sobre proxy/llamadas desde cliente entran en conflicto con la integración backend pedida: requieren aclaración para despliegue real. Una respuesta observada no demuestra cuota disponible. El fallback de auditoría en stderr no garantiza persistencia/recuperación durante una caída MySQL; commit tampoco garantiza entrega de respuesta al cliente. La sanitización no reconoce secretos arbitrarios sin formato identificable.

El repositorio incluye instalación con Docker, pruebas, diagramas y colección de Postman. La verificación manual de la colección debe realizarse siguiendo el recorrido anterior. CI, OpenAPI, ejecución automatizada de Postman, recuperación durable de auditoría y reintentos quedan fuera del alcance actual.
