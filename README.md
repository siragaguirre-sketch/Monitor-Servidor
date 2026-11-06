# TP Monitor de Servidor con Laravel

**Sistema Operativo utilizado:** Windows 10 / 11

## Evidencias y dashboards

La galería de capturas se encuentra en [`public/evidencias/index.html`](public/evidencias/index.html). Con el servidor Laravel o Apache de XAMPP en ejecución, se puede abrir en `/evidencias/`. Incluye imágenes del monitor `/servidor`, la respuesta JSON `/api/metrics`, el dashboard de seguridad y los gráficos `/graficos`, además de enlaces a esas vistas en vivo.

Las capturas son instantáneas; para ver métricas actualizadas, abre los enlaces del panel o la galería. El dashboard de seguridad requiere que la base de datos configurada en `.env` esté disponible.

## Respuestas al Cuestionario

### 1. ¿Por qué la temperatura suele aparecer como "No disponible" en Windows?

En Windows, consultar la temperatura del procesador mediante WMI (`root/wmi:MSAcpi_ThermalZoneTemperature`) requiere **privilegios de Administrador**. Además, depende de que el fabricante de la placa madre exponga esta interfaz ACPI hacia el sistema operativo. Si el servicio web corre con permisos de usuario estándar o el hardware no lo soporta, el bloque `try-catch` evita que falle la aplicación devolviendo `null` ("No disponible").

### 2. ¿Por qué el script de PowerShell nunca debe incluir datos que vengan del usuario?

Inyectar variables provenientes de peticiones HTTP (query strings, inputs de formularios, headers) directamente en un script ejecutado por `shell_exec()` permite ataques de **Inyección de Comandos (Command Injection)**. Un atacante podría incluir caracteres especiales (como `;` o `|`) y ejecutar comandos arbitrarios en el servidor con los permisos de PHP/PowerShell.

### 3. ¿Qué riesgo tiene dejar `/api/metrics` sin autenticación en un servidor real?

- **Divulgación de Información (Information Disclosure):** Expone detalles internos de la infraestructura (nombre de máquina, versión exacta de PHP, uso de disco/memoria, tiempo activo), facilitando que un atacante planifique exploits específicos.
- **Denegación de Servicio (DoS):** Como cada petición ejecuta un proceso hijo de PowerShell via `shell_exec()`, el costo de procesamiento es alto. Múltiples peticiones concurrentes sin restricción pueden consumir todos los recursos del servidor.

### 4. Con polling cada 5 segundos, ¿cuántos pedidos por minuto recibe el servidor si hay 50 personas mirando?

Cada usuario realiza 12 peticiones por minuto (60 segundos / 5 segundos).
Para 50 personas conectadas simultáneamente:
`50 usuarios * 12 peticiones/minuto = 600 peticiones por minuto`.

### 5. Si una app solo necesita la temperatura, ¿qué datos recibe de más al llamar a `/api/metrics`?

Recibe todo el resto del payload JSON no solicitado: `hostname`, `os`, `php`, `disk` (objeto completo), `memory` (objeto completo), `cpu_pct`, `uptime_s` y `at`. Este problema de recibir información de más se conoce como **over-fetching**, el cual se puede solucionar implementando GraphQL.

## Trabajo Práctico 2: Monitor en tiempo real

Esta parte reemplaza el polling de `/servidor` por eventos WebSocket con Laravel Reverb y Echo. El comando `metrics:broadcast` obtiene una lectura de `ServerMetrics` y publica `MetricsUpdated` en el canal público `server.metrics` cada cinco segundos. Tanto el monitor como `/graficos` escuchan el mismo evento. Los gráficos de CPU y memoria conservan hasta 30 lecturas y el disco se muestra como medidor.

### Instalación y ejecución local

Requisitos: PHP y Composer del proyecto, Node.js/npm y la base de datos configurada en `.env` activa. En una instalación nueva:

```powershell
composer install
php artisan install:broadcasting --reverb --without-node
npm install
php artisan migrate
```

La instalación de Reverb configura claves locales en `.env`. No compartas ese archivo ni publiques sus claves. Después, ejecuta cada proceso en una terminal distinta, desde la raíz del proyecto:

| Terminal | Comando |
| --- | --- |
| 1 — WebSocket | `php artisan reverb:start --debug` |
| 2 — Métricas cada 5 segundos | `php artisan schedule:work` |
| 3 — Assets del navegador | `npm run dev` |
| 4 — Aplicación web | `php artisan serve` |

Abre `http://127.0.0.1:8000/servidor` y `http://127.0.0.1:8000/graficos`. En el monitor, el estado debe pasar a **Conectado** y las métricas cambiar sin recargar la página. En las herramientas del navegador, la pestaña Red → WS permite observar la conexión persistente y los mensajes entrantes.

Para mostrar los botones de Calculadora, Bloc de notas y Paint en Windows, configura `DEMO_APPS=true` en `.env` y limpia la caché con `php artisan config:clear`. Déjalos desactivados por defecto. Los botones solo aceptan esas tres aplicaciones de una lista fija; la ruta es POST, está protegida con CSRF y limitada a dos solicitudes por minuto. Habilítalos únicamente durante una demostración en una red privada y confiable.

### Respuestas a las preguntas del TP

1. **¿Qué diferencia se ve en la pestaña Red entre el TP 1 y este TP?**  
   Con polling, cada navegador solicita `/api/metrics` repetidamente (cada cinco segundos). Con WebSockets se mantiene una conexión abierta y el servidor envía los eventos por esa conexión, sin una solicitud HTTP nueva por cada lectura.

2. **¿Qué pasa al detener `reverb:start` o `schedule:work`?**  
   Al detener Reverb, el navegador pierde la conexión y muestra **Sin conexión**; no recibe eventos. Si se detiene el scheduler, Reverb puede seguir conectado, pero las métricas dejan de actualizarse porque ya no se ejecuta `metrics:broadcast`.

3. **¿Por qué PowerShell se ejecuta la misma cantidad de veces con 1 o con 30 personas mirando?**  
   El comando programado toma una lectura central cada cinco segundos y difunde el resultado a los clientes conectados. No depende del número de navegadores; más clientes reciben el mismo evento sin iniciar una lectura de PowerShell cada uno.

4. **¿Por qué `DemoController` usa una lista blanca?**  
   El navegador envía una clave permitida, no un comando ejecutable. El servidor traduce esa clave a uno de los ejecutables definidos en el código y rechaza cualquier otra ruta, evitando que datos del usuario se conviertan en comandos arbitrarios.

5. **¿Qué diferencia hay entre `ShouldBroadcast` y `ShouldBroadcastNow`?**  
   `ShouldBroadcast` usa la cola configurada, por lo que su envío puede quedar pendiente hasta que un worker procese el job. `ShouldBroadcastNow` publica sincrónicamente durante la solicitud o comando, sin depender de un worker de colas; aquí se usa para simplificar el envío periódico de las métricas y la actividad de la demo.

6. **¿Por qué los gráficos no requieren cambios en el servidor y qué ventaja tiene el componente?**  
   El servidor ya envía las métricas en `MetricsUpdated`; la nueva vista escucha el mismo canal que el monitor y dibuja los valores en el navegador. El componente Blade reutiliza la tarjeta y sus atributos para representar cada métrica con su tipo y color, evitando duplicar el marcado.

# Informe de Auditoría y Monitoreo de Seguridad

## 8. Matriz de Pruebas Obligatorias y Validación

Las pruebas se ejecutaron en el entorno local de desarrollo (`127.0.0.1`) utilizando credenciales de prueba, garantizando que el flujo de login del sistema se mantenga totalmente funcional tras la integración del módulo de seguridad y auditoría.

- **Prueba 1: Login válido**
    - **Escenario:** Ingreso exitoso con credenciales correctas.
    - **Resultado Esperado:** Evento `INFO` en logs y registro con estado `success` en la tabla `login_attempts`.
    - **Resultado Obtenido:** Registro insertado con estado `success` en MySQL y redirección correcta.
    - **Estado:** PASÓ

- **Prueba 2: Dos contraseñas incorrectas**
    - **Escenario:** Ingreso de credenciales erróneas en 2 oportunidades seguidas.
    - **Resultado Esperado:** Dos eventos `WARNING` en logs y 2 registros con estado `failed`.
    - **Resultado Obtenido:** Registros insertados correctamente con estado `failed`.
    - **Estado:** PASÓ

- **Prueba 3: Tres fallos en 5 minutos**
    - **Escenario:** Acumulación de 3 intentos fallidos consecutivos dentro de la ventana de 5 minutos.
    - **Resultado Esperado:** Generación de evento `ALERT` en el intento siguiente y despliegue del badge `ALERTA SOSPECHOSA` en el dashboard.
    - **Resultado Obtenido:** IP `127.0.0.1` detectada e identificada con la etiqueta de alerta en el panel de auditoría.
    - **Estado:** PASÓ

- **Prueba 4: Cinco fallos en 5 minutos**
    - **Escenario:** Acumulación de 5 intentos fallidos consecutivos en la ventana de 5 minutos.
    - **Resultado Esperado:** Activación del bloqueo por rate-limiting durante 10 minutos en el intento siguiente.
    - **Resultado Obtenido:** Bloqueo temporal activado correctamente en la caché de la aplicación.
    - **Estado:** PASÓ

- **Prueba 5: Acceso con IP bloqueada**
    - **Escenario:** Intento de envío del formulario de login desde la IP en estado de bloqueo.
    - **Resultado Esperado:** Respuesta HTTP 429 Too Many Requests e inserción del evento `blocked` en la BD.
    - **Resultado Obtenido:** Respuesta 429 generada por el servidor y registro con estado `blocked` almacenado.
    - **Estado:** PASÓ

- **Prueba 6: Expiración de bloqueo (10 min)**
    - **Escenario:** Transcurso del período de restricción de 10 minutos.
    - **Resultado Esperado:** Restablecimiento del acceso y admisión de nuevos intentos de autenticación.
    - **Resultado Obtenido:** Control de rate-limiting liberado; permite enviar el formulario nuevamente.
    - **Estado:** PASÓ

### Notas de Ejecución

- **Entorno:** Localhost (`127.0.0.1`) / Laravel Herd + MySQL (XAMPP/phpMyAdmin).
- **Integración:** Se verificó que el flujo normal de inicio de sesión (`/security-login`) no se vio afectado y continúa procesando las peticiones de forma fluida.

## 9. Análisis y Toma de Decisiones

### Respuestas a las Preguntas de Auditoría

1. **¿Qué IP concentra más fallos?**
    - **Respuesta:** La dirección IP `127.0.0.1` concentra la totalidad de los intentos fallidos registrados (4 intentos fallidos en la ventana de evaluación), superando el umbral de alerta configurado de 3 fallos en 5 minutos.

2. **¿Puede afirmarse que existe un ataque?**
    - **Respuesta:** Si bien existe un patrón anómalo de reintentos continuos en un período corto (comportamiento compatible con un ataque de fuerza bruta o escaneo de credenciales), al provenir de la IP local (`127.0.0.1`) en un entorno de desarrollo, califica técnicamente como un escenario simulado de pruebas. En un entorno de producción real, este comportamiento justificará la activación inmediata de un protocolo de mitigación.

3. **¿Qué consecuencias tendría bloquear una IP compartida?**
    - **Respuesta:** Bloquear una IP compartida (por ejemplo, clientes detrás de un router NAT empresarial, una red Wi-Fi pública o un proxy/VPN) generaría un impacto negativo por **denegación de servicio a usuarios legítimos** (falsos positivos). Múltiples usuarios inocentes perderían acceso al sistema únicamente por compartir la misma dirección IP pública que el atacante.

4. **¿Qué otros indicadores usaría el equipo de seguridad?**
    - **Indicador de tasa de fallos por cuenta (Account Lockout Target):** Monitorear la concentración de intentos sobre un mismo usuario desde múltiples IPs (ataque distribuido o _Password Spraying_).
    - **Geolocalización de la IP y reputación:** Verificación de origen geográfico inusual o pertenencia a listas negras públicas (Open Proxies, nodos Tor, botnets conocidos).
    - **Análisis de User-Agent y Huella de Navegador:** Identificación de patrones automatizados mediante scripts, cURL, o herramientas de prueba de penetración sin cabeceras estándar.

5. **¿Qué acciones deben ser automáticas y cuáles requieren revisión humana?**
    - **Acciones Automáticas:**
        - Registro e inserción inmediata del intento en la tabla de auditoría (`login_attempts`).
        - Bloqueo temporal por rate-limiting (ej: throttling de 15 minutos mediante Middleware) para frenar la automatización del ataque.
        - Generación del badge de alerta visual (`ALERTA SOSPECHOSA`) en el dashboard de monitoreo.
    - **Acciones con Revisión Humana:**
        - Bloqueo definitivo de direcciones IP a nivel de Firewall / Red.
        - Suspensión o baneo permanente de cuentas de usuario afectadas.
        - Notificación formal de incidentes y análisis forense de logs de auditoría.

---

## Actividad Integradora: Mejoras Seleccionadas

Para potenciar la robustez del módulo de auditoría del sistema, se seleccionaron las siguientes dos mejoras:

1. **Notificaciones de Alerta en Tiempo Real:**
    - **Descripción:** Envío automático de alertas críticas (vía Email o webhook de Slack/Discord) a los administradores de seguridad cuando una IP supere el umbral de 3 fallos en menos de 5 minutos, permitiendo una respuesta inmediata sin necesidad de mantener el dashboard abierto constantemente.

2. **Exportación de Evidencias (Logs de Auditoría):**
    - **Descripción:** Funcionalidad para exportar el historial de intentos filtrados (`success`, `failed`, `blocked`) en formatos estándar como CSV o PDF. Esto asegura la trazabilidad, preservación de cadena de custodia y facilitación de informes de cumplimiento o auditorías externas.
