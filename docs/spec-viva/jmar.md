## Purpose

La capacidad de cuentas y acceso permite a una persona crear una cuenta, autenticarse y mantener una sesión con token para consultar su perfil en FlowSync. También impide que las pantallas privadas se muestren sin una sesión que el backend reconozca.

## Requirements

### Requirement 1: Registro de cuenta

El sistema SHALL permitir el registro mediante `POST /api/v1/auth/signup` con `email`, `password`, `passwordConfirmation` y un `fullName` anulable. SHALL validar que el email tenga formato válido, no supere 254 caracteres y sea único; que la contraseña tenga entre 8 y 32 caracteres; y que su confirmación coincida. Al crear la cuenta, SHALL persistir el usuario, proteger la contraseña mediante el modelo de autenticación, crear un token de acceso opaco y responder con `{ data: { user, token } }`.

La representación `user` SHALL exponer solamente `id`, `fullName`, `email`, `createdAt`, `updatedAt` e `initials`; nunca la contraseña. `initials` SHALL derivarse del nombre completo o, si no existe, de la parte anterior a `@` del email.

En pantalla, la ruta pública `/register` SHALL mostrar el formulario «Crea tu cuenta», con nombre completo opcional, email, contraseña y repetición de contraseña. SHALL comprobar localmente que ambas contraseñas coincidan antes de enviar, indicar que la contraseña debe tener entre 8 y 32 caracteres, deshabilitar el botón mientras envía y mostrar los errores de validación junto al campo correspondiente. Un email duplicado SHALL informar que ya está registrado e invitar a iniciar sesión.

## Scenario: Registro correcto

- **WHEN** una persona envía en `/register` un email no registrado, una contraseña válida y su misma confirmación, con o sin nombre completo.
- **THEN** la API crea el usuario y devuelve su perfil seguro y un token bajo `data`; el frontend guarda el token, establece la sesión autenticada y navega a `/profile` por la protección de rutas públicas.

## Scenario: Registro inválido

- **WHEN** una persona intenta registrarse con campos inválidos, un email ya usado o contraseñas diferentes.
- **THEN** no se crea la cuenta; los errores 422 de VineJS se traducen al castellano y se muestran en los campos, y la discrepancia de contraseñas se comunica localmente sin hacer una petición.

### Requirement 2: Inicio de sesión

El sistema SHALL permitir el inicio de sesión mediante `POST /api/v1/auth/login` con email y contraseña. SHALL validar el formato de email y que se suministre una contraseña; si las credenciales son correctas, SHALL verificar la contraseña con el modelo `User`, emitir un token de acceso opaco y responder con `{ data: { user, token } }` usando la misma representación segura del usuario que el registro.

En pantalla, la ruta pública `/login` SHALL mostrar «Inicia sesión», campos de email y contraseña, el enlace a registro y el botón «Entrar». SHALL impedir envíos duplicados mientras la petición está en curso. Para credenciales incorrectas SHALL mostrar «El email o la contraseña no son correctos.»; para errores de validación SHALL mostrar los mensajes traducidos junto a los campos, y para fallos de conexión o servidor SHALL mostrar un aviso general.

## Scenario: Inicio de sesión correcto

- **WHEN** una persona envía credenciales válidas desde `/login`.
- **THEN** recibe un usuario seguro y un token de `POST /api/v1/auth/login`; la aplicación almacena el token en `localStorage` bajo `flowsync.token`, marca la sesión como autenticada y redirige a `/profile`.

## Scenario: Inicio de sesión rechazado

- **WHEN** una persona envía un email o una contraseña que no corresponden a una cuenta.
- **THEN** el backend rechaza las credenciales y la pantalla mantiene el formulario, reactiva el botón y presenta el mensaje de error sin crear sesión.

### Requirement 3: Sesión autenticada con token

El sistema SHALL autenticar las rutas privadas mediante el guard `api`, que lee tokens de acceso Bearer asociados al modelo `User`. El middleware global SHALL forzar respuestas JSON y ejecutar una comprobación silenciosa de autenticación sin bloquear rutas públicas; las rutas privadas SHALL ejecutar además el middleware `auth`, que rechaza solicitudes sin token válido.

La aplicación web SHALL adjuntar `Authorization: Bearer <token>` a las llamadas autenticadas. Al arrancar con `flowsync.token` almacenado, SHALL mantener el estado `loading` y consultar `GET /api/v1/account/profile` antes de considerar válida la sesión. Si recibe un 401, SHALL borrar token y usuario y pasar a anónimo; ante un fallo de red o servidor, SHALL conservar el token almacenado, mostrar el motivo en acceso y tratar la sesión actual como anónima hasta un nuevo arranque o inicio de sesión.

## Scenario: Restauración de una sesión válida

- **WHEN** se recarga la aplicación y existe un token que `GET /api/v1/account/profile` reconoce.
- **THEN** se muestra un indicador de carga hasta recibir el perfil, se restaura el usuario en memoria y se permite entrar a las rutas privadas.

## Scenario: Token caducado o inválido

- **WHEN** la restauración del perfil devuelve 401 para el token guardado.
- **THEN** la aplicación elimina `flowsync.token`, cambia a estado anónimo y muestra en el acceso que la sesión ha caducado y debe iniciarse de nuevo.

### Requirement 4: Protección y redirección de rutas

El sistema SHALL proteger `/profile`: mientras se resuelve una sesión persistida SHALL mostrar una carga a pantalla completa, y sin sesión autenticada SHALL redirigir a `/login`. SHALL tratar `/login` y `/register` como rutas solo públicas y redirigir desde ellas a `/profile` cuando ya hay sesión. Cualquier URL no reconocida SHALL redirigir a `/profile`, desde donde la protección decidirá si corresponde mostrar perfil o acceso.

## Scenario: Visita a una ruta privada sin sesión

- **WHEN** una persona anónima abre `/profile` o llega a una URL desconocida.
- **THEN** el guard de rutas no renderiza el contenido privado y la persona termina en `/login`.

## Scenario: Visita a acceso con sesión

- **WHEN** una persona autenticada abre `/login` o `/register`.
- **THEN** el guard de rutas públicas evita mostrar el formulario y la redirige a `/profile`.

### Requirement 5: Perfil y cierre de sesión

El sistema SHALL exponer `GET /api/v1/account/profile` únicamente con autenticación y responder con `{ data: user }`. La pantalla `/profile` SHALL mostrar las iniciales, el nombre —o «Sin nombre»—, el email y la fecha de creación localizada en español, junto con la acción «Cerrar sesión».

El sistema SHALL aceptar `POST /api/v1/account/logout` únicamente con autenticación. SHALL eliminar del almacén de tokens el token actual cuando exista y responder `{ message: "Logged out successfully" }` sin el envoltorio `data`. Al cerrar sesión, la aplicación SHALL eliminar de inmediato el token y usuario locales, volver al estado anónimo y permitir que el guard redirija a `/login`; la revocación remota se intentará después sin impedir el cierre local si falla.

## Scenario: Consulta de perfil autenticada

- **WHEN** una persona con token Bearer válido solicita `/api/v1/account/profile` o llega a `/profile`.
- **THEN** la API entrega solo los datos transformados del usuario y la pantalla presenta su tarjeta de perfil con la fecha «Miembro desde».

## Scenario: Cierre de sesión

- **WHEN** una persona pulsa «Cerrar sesión» en su perfil.
- **THEN** el botón indica «Cerrando sesión…», la aplicación limpia su sesión local, solicita la revocación del token al backend y el guard la redirige a `/login` incluso si esa solicitud remota no puede completarse.
