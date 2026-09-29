# Registro de Usuarios — Laravel 13 + Brevo SMTP + Colas

Trabajo Práctico de **Programación IV** — Tecnicatura Universitaria en Programación (UTN).

Sistema de registro de usuarios en Laravel 13 con validaciones avanzadas, envío de correo de
bienvenida a través del servicio SMTP de Brevo, y procesamiento asíncrono del envío mediante
Colas (Queues).

## Qué hace

1. Formulario de registro (`/register`) con validaciones de nombre, email (único, formato
   RFC/DNS) y contraseña (mínimo 8 caracteres, confirmada).
2. Al registrarse, el usuario se guarda en la base de datos con la contraseña hasheada.
3. Se despacha un **Job** (`SendWelcomeEmailJob`) que envía un correo HTML de bienvenida a
   través de Brevo SMTP, sin bloquear la respuesta al usuario.
4. Un **worker** de Artisan procesa esa cola en segundo plano.

## Stack

- Laravel 13
- SQLite (base de datos de desarrollo)
- Brevo (SMTP) para el envío de correos
- Colas con driver `database`

## Estructura relevante

```
app/Http/Controllers/RegisterController.php   # Validaciones, alta de usuario, dispatch del Job
app/Jobs/SendWelcomeEmailJob.php              # Job encolado que arma y envía el mail
app/Mail/WelcomeUserMail.php                  # Mailable: asunto + vista del correo
resources/views/auth/register.blade.php       # Formulario de registro
resources/views/emails/welcome.blade.php      # Plantilla HTML del correo de bienvenida
routes/web.php                                # Rutas GET/POST de /register
```

## Cómo correrlo localmente

1. Clonar el repo e instalar dependencias:

   ```bash
   composer install
   ```

2. Copiar `.env.example` a `.env` y completar:

   - Credenciales de base de datos (por defecto usa SQLite; crear el archivo con
     `type nul > database/database.sqlite` en Windows o `touch database/database.sqlite`).
   - Credenciales SMTP de Brevo (`MAIL_USERNAME`, `MAIL_PASSWORD` con la **SMTP key** de 64
     caracteres, `MAIL_FROM_ADDRESS` con el remitente verificado en Brevo).

3. Generar la clave de la aplicación y correr las migraciones:

   ```bash
   php artisan key:generate
   php artisan migrate
   ```

4. Levantar el servidor:

   ```bash
   php artisan serve
   ```

5. En otra terminal, levantar el worker de colas (necesario para que se procesen y envíen los
   correos de bienvenida):

   ```bash
   php artisan queue:work
   ```

6. Ir a `http://127.0.0.1:8000/register` y probar el formulario.

## Sobre las Colas

El envío del correo de bienvenida no es crítico para completar el registro, así que se saca
del ciclo de request/response principal: el controlador despacha `SendWelcomeEmailJob` y
responde de inmediato al usuario, mientras el worker procesa el envío real en segundo plano.
Si el worker no está corriendo, el Job queda pendiente en la tabla `jobs` hasta que se prenda.
