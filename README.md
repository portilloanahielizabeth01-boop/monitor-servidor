# Monitor de Servidor con Laravel

Trabajo práctico de **Programación IV** · UTN FRRe · Cohorte 2026

App Laravel que lee datos del sistema operativo Windows (memoria, CPU, disco, tiempo encendido y temperatura) y los muestra en una vista web que se actualiza sola (polling) y en una API JSON.

## Entorno usado

| | |
|---|---|
| Sistema operativo | Windows 11 |
| PHP | 8.4.22 (XAMPP) |
| Laravel | 13 |
| Equipo | LAPTOP-VKKQ5N28 |

## Cómo ejecutarlo

```powershell
composer install
copy .env.example .env
php artisan key:generate
php artisan migrate
php artisan serve
```

- Vista web: http://127.0.0.1:8000/servidor
- API JSON: http://127.0.0.1:8000/api/metrics

## Estructura

| Parte | Archivo | Responsabilidad |
|---|---|---|
| Service | `app/Services/ServerMetrics.php` | Lee los datos del sistema operativo (PHP + PowerShell/WMI) |
| Controlador | `app/Http/Controllers/ServerController.php` | Devuelve la vista o el JSON |
| Vista | `resources/views/servidor.blade.php` | Muestra las tarjetas y hace polling cada 5 segundos |
| Rutas | `routes/web.php`, `routes/api.php` | `/servidor` y `/api/metrics` |

*Capturas
<img width="1917" height="395" alt="Captura de pantalla 2026-11-05 120249" src="https://github.com/user-attachments/assets/1e6b710b-ebbf-4d9a-87e9-65fbaebcf802" />
<img width="305" height="427" alt="Captura de pantalla 2026-11-05 120243" src="https://github.com/user-attachments/assets/f89d4a2d-dbf6-4856-b46b-5d5db86d3a1d" />
<img width="1910" height="327" alt="Captura de pantalla 2026-11-05 123157" src="https://github.com/user-attachments/assets/7fb4e786-990f-4983-8acb-cd51a985e50a" />


## Respuestas a las preguntas

### 1. ¿Por qué la temperatura suele aparecer como "No disponible" en Windows?

Porque la clase de WMI `MSAcpi_ThermalZoneTemperature` (en el namespace `root/wmi`) **solo se puede leer con permisos de administrador**. Si la terminal o el servidor corren como usuario común, PowerShell devuelve "Acceso denegado", el `try/catch` del script captura el error y la temperatura queda en `null`. Por eso la vista muestra "No disponible".

Además, aun ejecutando como administrador, muchos equipos (sobre todo notebooks) **no informan este dato**: la temperatura sale de la zona térmica ACPI que expone la placa madre, no directamente del sensor del procesador. Si el fabricante no la implementa, la clase no devuelve nada, o devuelve un valor fijo que no cambia.

**Prueba realizada:** con la terminal normal la temperatura dio `null`. Al ejecutar como administrador el resultado fue: 68.1.

### 2. ¿Por qué el script de PowerShell nunca debe incluir datos que vengan del usuario?

Porque `shell_exec()` ejecuta comandos reales en el servidor con los permisos del proceso de PHP. Si se armara el script con datos de un formulario o de la URL, un atacante podría mandar texto que cierre el comando original y agregue el suyo. Por ejemplo, podría borrar archivos, leer el `.env` con las contraseñas o descargar y ejecutar malware. Este ataque se llama **inyección de comandos** (*command injection*).

Codificar el script en Base64 no protege contra esto. Solo evita problemas de comillas: si el dato malicioso ya forma parte del script, se codifica y se ejecuta igual. Por eso en este TP el script es **fijo** y no recibe ningún dato externo.

### 3. ¿Qué riesgo tiene dejar `/api/metrics` sin autenticación en un servidor real?

- **Filtración de información:** cualquiera puede ver el nombre del equipo, la versión exacta del sistema operativo y de PHP. Con eso un atacante puede buscar vulnerabilidades conocidas para esas versiones.
- **Ver el estado del servidor:** el uso de CPU, memoria y disco muestra cuándo el servidor está sobrecargado o casi lleno, lo que ayuda a planear un ataque.
- **Denegación de servicio (DoS):** cada pedido abre un proceso de PowerShell, que es costoso. Muchos pedidos simultáneos pueden saturar el servidor. El `throttle:60,1` limita los pedidos por IP, pero no frena un ataque desde muchas IP distintas.

La solución es protegerlo con autenticación, por ejemplo `->middleware('auth:sanctum')`, para que solo usuarios autorizados puedan consultarlo.

### 4. Con polling cada 5 segundos, ¿cuántos pedidos por minuto recibe el servidor si hay 50 personas mirando?

Cada persona hace un pedido cada 5 segundos, o sea **60 ÷ 5 = 12 pedidos por minuto**.

**50 personas × 12 = 600 pedidos por minuto**, que son 10 pedidos por segundo. Cada uno abre PowerShell, aunque los datos no hayan cambiado. Este es el problema que resuelven los WebSockets: el servidor lee los datos una sola vez y se los envía a todos los conectados.

### 5. Si una app solo necesita la temperatura, ¿qué datos recibe de más al llamar a `/api/metrics`?

Recibe **todos los demás campos**, que no necesita:

- `hostname`, `os` y `php`
- `disk` (`total_gb`, `free_gb`, `used_pct`)
- `memory` (`total_mb`, `available_mb`, `used_pct`)
- `cpu_pct`, `uptime_s` y `at`

Solo le sirve `cpu_temp_c`. Esto se llama **over-fetching**: con REST, el cliente recibe siempre la respuesta completa del recurso. Además, el servidor igual calcula todos los datos (ejecuta el script completo de PowerShell) aunque se use uno solo. GraphQL resuelve este problema porque el cliente pide exactamente los campos que necesita.
