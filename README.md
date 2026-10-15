# CAFAS · Render gratuito + Neon gratuito

Esta versión usa **PostgreSQL externo en Neon**, con conexión cifrada TLS. Render ejecuta la web y el servidor en su plan Free. **No usa SQLite en producción, ni disco persistente, ni base de datos de pago de Render.** Los registros permanecen en Neon aunque Render suspenda, reinicie o redespliegue la aplicación.

## 1. Crear la base gratuita

1. Entra en https://neon.com y crea una cuenta.
2. Crea un proyecto llamado CAFAS y mantén el plan **Free**. Elige una región cercana a la que usarás en Render.
3. Abre **Connect**, selecciona tu base (normalmente `neondb`) y copia la cadena de conexión PostgreSQL. Puedes usar la conexión agrupada (pooled).
4. Guarda esa cadena para ingresarla en Render como `DATABASE_URL`. Incluye usuario, contraseña, servidor y base. No la envíes por chat ni la incluyas en el repositorio.

No hay que crear tablas manualmente: CAFAS crea el esquema y carga los 20 usuarios en el primer arranque. Los arranques siguientes conservan los datos y las contraseñas cambiadas.

## 2. Publicar en Render sin plan de pago

1. Sube el contenido de la carpeta `CAFAS-Render-Gratis` a la raíz de un repositorio **privado** de GitHub. Incluye `private/users.json`: contiene las 20 personas del Excel y nunca se sirve por HTTP.
2. En https://dashboard.render.com elige **New → Blueprint** y conecta ese repositorio.
3. Render leerá `render.yaml`, configurado con `plan: free` y sin disco.
4. Cuando solicite **DATABASE_URL**, pega la conexión de Neon. Aplica la configuración.
5. Espera a que termine el despliegue y abre la URL HTTPS que Render asigne.
6. Ingresa con cédula y contraseña inicial igual a la cédula. Mi perfil permite cambiar la contraseña.

### Configuración manual / servicio existente

| Campo | Valor |
|---|---|
| Tipo de servicio | Web Service |
| Runtime | Node |
| Instance Type | Free |
| Build command | `npm ci --include=dev && npm run build` |
| Start command | `npm start` |
| Health check | `/healthz` |
| NODE_VERSION | `24.19.0` |
| NODE_ENV | `production` |
| DATABASE_URL | Conexión secreta de Neon |
| Disco | Ninguno |

Si el código está en una subcarpeta, configura Root Directory. No lo instales como Static Site: el login y el guardado requieren servidor.

Si ya habías instalado la versión con SQLite y tienes evaluaciones reales, realiza la migración de la sección siguiente **antes de eliminar su disco o cambiar el servicio**. Este ZIP no modifica automáticamente tu cuenta de Render ni importa registros de un servidor al que no tenemos acceso.

## Usuarios y funcionamiento

- 20 usuarios importados del Excel; ningún usuario ficticio.
- Evaluador y evaluado: OSPINA TRUJILLO DIEGO ARMANDO, MARTINEZ MONTENEGRO JUAN PABLO, FERNANDEZ PEREZ SERGIO IVAN, NARANJO VALENCIA ANDRES DANIEL y BECERRA MARIN JOSE LUIS.
- Los otros 15 usuarios solo son evaluados.
- Se conservan login, cambio de contraseña, búsqueda por cédula, pruebas 1–4, CSV filtrado, IMC, aprobación y planes condicionales.
- Los evaluados solo reciben sus propios datos. Los evaluadores pueden consultar y registrar evaluaciones de los usuarios. Los cinco evaluadores pueden cambiar a la vista de sus propios resultados.
- Una transacción y un bloqueo por persona impiden dos pruebas pendientes simultáneas. La categoría se calcula en el servidor; no se confía en la calificación enviada desde el navegador.
- Los datos del Excel corresponden a APELLIDOS Y NOMBRES, CEDULA, SEXO, ESTAT y PESO ACTUAL. M/F del Excel se convierten a H/M en el motor de baremos.
- Estatura, peso y edad estaban vacíos: se mantienen pendientes, se completan en Mi perfil o al crear la prueba. No se inventaron datos.
- Pulsa **Actualizar resultados** para consultar cambios de otros usuarios. Se quitaron las consultas periódicas en segundo plano para reducir consumo.
- El control de salud `/healthz` verifica el proceso, sin consultar Neon continuamente. La conexión usa un máximo de cinco conexiones y libera las que estén inactivas.
- Las actividades y rutas finalizadas se guardan por cuenta. El GPS en curso vive en memoria, requiere pantalla visible y se pausa al bloquear el dispositivo. Finaliza y guarda antes de cerrar.

## Límites gratuitos (revisados el 15 de octubre de 2026)

Render Free suspende una web tras 15 minutos sin tráfico; la primera visita posterior puede tardar alrededor de un minuto en responder. El almacenamiento local de Render es temporal, por eso toda la información persistente está en Neon.

Neon Free no tiene vencimiento fijo e incluye actualmente 1 GB de almacenamiento por proyecto, 100 CU-horas de cómputo por proyecto/mes y 5 GB de transferencia pública por proyecto/mes. El cómputo se suspende tras cinco minutos de inactividad. El uso gratuito está sujeto a esos límites, no es capacidad ilimitada: al agotarse cómputo o transferencia se pausa hasta el siguiente período o una ampliación; al alcanzar el límite de almacenamiento se restringen escrituras. Mantén Free para no contratar consumo de pago. Las rutas GPS extensas consumen más almacenamiento que las evaluaciones.

La base PostgreSQL gratuita de Render vence a los 30 días. Este proyecto no crea esa base: usa Neon.

Fuentes oficiales:
- https://render.com/docs/free
- https://neon.com/pricing
- https://neon.com/docs/introduction/plans

## Si ya registraste evaluaciones en la versión SQLite

1. Genera una copia consistente con el comando `npm run backup` de la versión anterior y descárgala. No copies únicamente el archivo principal de una base SQLite que esté en ejecución.
2. Crea en Neon una base vacía. Aún no inicies la nueva aplicación: su primer arranque normal cargaría los usuarios iniciales.
3. En tu computador, con Node 24 y este paquete, ejecuta `npm ci --include=dev`.
4. Define `DATABASE_URL` con esa base y ejecuta:

```sh
npm run migrate:sqlite -- /ruta/a/la/copia.sqlite
```

La herramienta migra usuarios, hashes de contraseñas, evaluaciones, aprobaciones, actividades y auditoría en una transacción. No modifica la copia de origen y rechaza un destino con usuarios existentes. Después puedes desplegar normalmente. Las sesiones anteriores no se transfieren: hay que iniciar sesión de nuevo.

## Copias de seguridad de PostgreSQL

Desde tu computador, define `DATABASE_URL` y ejecuta:

```sh
npm run backup -- cafas-respaldo.json
```

Genera una copia consistente de usuarios, evaluaciones, información personal y auditoría. Guarda el archivo fuera de Render y mantenlo privado: contiene cédulas y hashes de contraseñas. No requiere Render Shell (que puede no estar disponible en Free). Las sesiones no se incluyen.

Para recuperar en una **base vacía** de Neon, define su `DATABASE_URL` y ejecuta antes del primer arranque:

```sh
npm run restore -- cafas-respaldo.json
```

Nunca sobrescribe una base que ya contenga usuarios.

## Desarrollo y pruebas

```sh
npm ci --include=dev
npm run build
npm test
```

Para iniciar, define `DATABASE_URL` y ejecuta `npm start`. También puedes copiar `.env.example` a `.env`, completar los valores reales y ejecutar `node --env-file=.env server/index.cjs`. Para abrir en localhost, usa `NODE_ENV=development` (las cookies Secure de producción requieren HTTPS).

Las pruebas usan PGlite, el motor PostgreSQL embebido, con bases temporales. Verifican SQL, roles, creación, aprobación, concurrencia, filtros, cambio de contraseña, persistencia tras reiniciar y flujos de interfaz DOM conectados al servidor HTTP. En producción se usa `pg` por TLS hacia Neon; PGlite no se utiliza en Render. La conexión a tu proyecto Neon y el despliegue final requieren que configures tu DATABASE_URL; no se han realizado desde aquí.

Archivos principales: `server/db.cjs` (conexión/transacciones), `server/schema.sql` (tablas), `server/index.cjs` (API), `private/users.json` (carga inicial), `render.yaml` (plan gratuito), `scripts/` (compilación, respaldo y migración). No hay secretos ni credenciales de Neon incluidos.
