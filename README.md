# Prex Challenge

API REST de cuatro servicios integrada con GIPHY: Laravel 13.35.0, Passport 13.9.0, PHP 8.4.26 y MySQL 8.4.11. Arquitectura hexagonal con Dominio, Aplicación e Infraestructura.

## Instalación desde un checkout limpio

Requisitos del host: Docker Engine operativo, Docker Compose con soporte de `service_healthy`, shell e Internet para descargar imágenes/paquetes. No requiere PHP, Composer ni Node del host. Ejecutar desde la raíz. En Linux se usa el UID/GID del usuario para evitar dependencias propiedad de root; se recomienda un usuario normal con UID/GID >= 1000.

```bash
# Los valores bootstrap solo permiten evaluar Compose antes de crear .env.
# No inicializan MySQL ni son credenciales persistidas.
APP_UID=$(id -u) APP_GID=$(id -g) DB_PASSWORD=bootstrap MYSQL_ROOT_PASSWORD=bootstrap docker compose build app
DB_PASSWORD=bootstrap MYSQL_ROOT_PASSWORD=bootstrap docker compose run --rm --no-deps --user "$(id -u):$(id -g)" --entrypoint php app scripts/init-env.php

docker compose run --rm --no-deps app composer install --no-interaction --prefer-dist
docker compose run --rm --no-deps app php artisan key:generate
docker compose up -d --wait mysql
docker compose run --rm app php artisan migrate --force
docker compose run --rm app php artisan passport:keys
docker compose run --rm app php artisan passport:client --personal --name="Prex API personal" --provider=users --no-interaction
# Opcional, solo para desarrollo local:
docker compose run --rm app php artisan db:seed --class=LocalUserSeeder --force
docker compose up -d --wait
curl --fail http://127.0.0.1:8080/up
```

El generador crea `.env` con dos contraseñas aleatorias independientes, permisos 600 y UID/GID locales; se niega a sobrescribir un `.env` existente. APP_KEY y GIPHY_API_KEY permanecen vacías hasta sus respectivos pasos explícitos. `key:generate` genera APP_KEY; la API Key de GIPHY se completa manualmente para usar la búsqueda real. `.env.example` contiene únicamente ejemplos. No compartir `.env`, claves Passport ni tokens. No ejecutar `install:api`: las cinco migraciones oficiales de Passport 13.9.0 ya están incluidas sin cambios de esquema.

Para cambiar el puerto HTTP, editar `HTTP_PORT` y `APP_URL` en `.env` antes de arrancar. El bind permanece en `127.0.0.1`. MySQL publica `127.0.0.1:3306` para clientes locales como DBViewer; el puerto del host se puede cambiar con `DB_FORWARD_PORT` en `.env` (default 3306). En DBViewer usar host `127.0.0.1`, ese puerto, base `DB_DATABASE`, usuario `DB_USERNAME` y contraseña `DB_PASSWORD`. Laravel sigue conectando por la red interna con `DB_HOST=mysql` y `DB_PORT=3306`; FPM no publica puertos. El healthcheck ejecuta `SELECT 1` con el usuario de aplicación sobre su base; app espera ese estado saludable. Nginx monta únicamente `public/` y solo ejecuta `public/index.php`.

El código y vendor se montan desde el checkout para desarrollo. Composer se ejecuta dentro de PHP 8.4, bajo la misma identidad que los workers FPM. El entrypoint prepara storage y bootstrap/cache con directorios 775 y archivos 664, sin 777. No instala dependencias, genera secretos ni ejecuta migraciones al arrancar. El Dockerfile es de desarrollo, no una imagen de producción con código incorporado.

## Configuración y persistencia

Para el recorrido local, el seeder anterior crea `user@example.test` / `contraseña-local` si no existe; no sobrescribe usuarios existentes ni restaura borrados. Completar `GIPHY_API_KEY` únicamente en `.env` antes del recorrido. Después de editar configuración ejecutar `docker compose exec app php artisan config:clear`. Mantener `CACHE_STORE=database` para compartir el límite de login entre procesos.

Conservar APP_KEY, `storage/oauth-private.key`, `storage/oauth-public.key` y el cliente personal activo (`provider=users`, grant `personal_access`) de `oauth_clients`. Las claves persisten en el checkout y el entrypoint las deja en 600. El volumen `mysql_data` conserva usuarios, cliente, tokens, favoritos y auditoría al ejecutar `stop`, `down` o recrear contenedores. No regenerar claves al reiniciar. `down -v` elimina datos. Cambiar passwords de `.env` no rota credenciales de un volumen inicializado. Respaldar entorno, claves y base de forma segura.

`TRUSTED_PROXIES` admite IP/CIDR explícitos de proxies controlados, separados por comas; vacío no confía en ninguno. Se ignoran comodines, /0 y valores inválidos. El proxy debe reemplazar headers de forwarding recibidos del cliente. No se confía en X-Forwarded-Host.

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

Usar una copia independiente sin `.env`, vendor, claves ni cachés. En esa raíz ejecutar la única secuencia de instalación anterior, agregando `-p prex-stage8-clean` a **todos** los comandos Compose. Tras generar `.env`, antes de iniciar MySQL, cambiar HTTP_PORT=18088, APP_URL=http://127.0.0.1:18088 y DB_FORWARD_PORT=13388 (o puertos libres). Completar GIPHY_API_KEY en esa copia para el recorrido real. No compartir caches ni checkout entre proyectos con entornos diferentes.

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

Comprobar `/up` 200 y `/.env`, `/composer.json`, `/artisan`, `/vendor/autoload.php`, `/storage/oauth-private.key`, `/storage/logs/laravel.log` 404. Revisar `docker compose ps`: puertos publicados solo en loopback. Completar el recorrido de Postman indicado abajo y detener la copia con `down`, conservando el volumen.

## Diagramas y Postman

[Índice UML y DER](docs/diagrams/README.md): siete fuentes PlantUML y SVG, DER del proyecto (`erd`) y DER con tablas auxiliares (`erd-completo`), renderizador local fijado y relaciones físicas/lógicas diferenciadas.

[Guía Postman](docs/postman/README.md): importar colección v2.1 y environment, completar credenciales locales y ejecutar login → búsqueda → consulta → creación → duplicado. Solo el token se guarda automáticamente; completar user_id y gif_id manualmente. Exportaciones con token, IDs y password vacíos; la API Key nunca va en Postman. La comprobación manual en la aplicación permanece pendiente del usuario.

## Diagnóstico y límites

Usar `docker compose logs --tail=100 app nginx mysql`, `php artisan migrate:status`, `php artisan route:list`, `composer validate --strict` y `composer check-platform-reqs` mediante `docker compose exec app`. No compartir `.env`, salidas de `config:show`/Compose interpolado, tokens ni logs sin revisar. Login 500: comprobar migraciones, claves legibles y cliente personal users; 503 GIPHY: comprobar configuración de clave y respuesta del proveedor sin imprimirla. `/up` comprueba el framework, no disponibilidad de MySQL.

Diferencias acordadas respecto del PDF: IDs textuales de GIPHY frente a numéricos; login propio con tokens personales Passport como interpretación de OAuth2.0, sin Authorization Code + PKCE. Las restricciones del proveedor sobre proxy/llamadas desde cliente entran en conflicto con la integración backend pedida: requieren aclaración para despliegue real. Una respuesta observada no demuestra cuota disponible. El fallback de auditoría en stderr no garantiza persistencia/recuperación durante una caída MySQL; commit tampoco garantiza entrega de respuesta al cliente. La sanitización no reconoce secretos arbitrarios sin formato identificable.

Entregables locales E2–E7 incluidos. Aceptación de etapa 8 sujeta a verificaciones registradas en el plan y comprobación manual de Postman. E1, publicación Git externa, pendiente de autorización expresa. El challenge no se declara completo. CI, OpenAPI, ejecución automatizada de Postman, recuperación durable y reintentos quedan fuera del alcance.
