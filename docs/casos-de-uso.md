# Casos de uso

> Estado: en desarrollo. CU-01 a CU-04 redactados; CU-05 a CU-09 en proceso.

## Índice

- [Casos de uso](#casos-de-uso)
  - [Índice](#índice)
  - [Glosario](#glosario)
  - [CU-01: Registrar usuario](#cu-01-registrar-usuario)
  - [CU-02: Abrir cuenta](#cu-02-abrir-cuenta)
  - [CU-03: Iniciar sesión](#cu-03-iniciar-sesión)
  - [CU-04: Recibir ingreso externo](#cu-04-recibir-ingreso-externo)
  - [CU-05: Consultar saldo (en proceso)](#cu-05-consultar-saldo-en-proceso)
  - [CU-06: Consultar destinatario (en proceso)](#cu-06-consultar-destinatario-en-proceso)
  - [CU-07: Transferir (en proceso)](#cu-07-transferir-en-proceso)
  - [CU-08: Historial (en proceso)](#cu-08-historial-en-proceso)
  - [CU-09: Detalle de transferencia (en proceso)](#cu-09-detalle-de-transferencia-en-proceso)

## Glosario

- **Usuario:** persona registrada en el sistema.
- **Visitante:** persona que todavía no se registró.
- **Cuenta:** cuenta de un usuario en una moneda (ARS o USD), con saldo, alias y número de cuenta externo.
- **Número de cuenta externo:** identificador de 22 dígitos numéricos (equivalente a un CBU/CVU) para recibir dinero desde afuera.
- **Alias:** identificador legible de la cuenta, formado por 3 palabras separadas por un punto.
- **Proveedor externo:** sistema ajeno (banco, billetera o red de cobranza) que envía ingresos a las cuentas. Se autentica con clave de API y firma.
- **Cuenta de liquidación:** cuenta de sistema, una por proveedor, desde la que se debitan los ingresos externos.
- **Ingreso externo:** dinero que llega a una cuenta desde un proveedor externo.
- **Asiento:** registro contable inmutable de un débito o un crédito sobre una cuenta.
- **Id de operación:** identificador único que envía el proveedor y se usa como clave de idempotencia.

---

## CU-01: Registrar usuario

- **Actor principal:** visitante no registrado.
- **Objetivo:** tener un usuario registrado.
- **Disparador:** `POST /users` con nombre, apellido, e-mail, fecha de nacimiento, contraseña y términos aceptados.

**Precondiciones**

- El visitante no está registrado.

**Flujo principal**

1. El sistema valida que los datos ingresados son correctos (campos obligatorios, nombres y apellidos sin números ni caracteres especiales (exceptuando tildes, espacios, apóstrofes, guiones y la ñ), formato del e-mail, mayor o igual a 18 años, contraseña segura, términos aceptados).
2. El sistema verifica que el e-mail no esté registrado.
3. El sistema guarda el usuario con la contraseña hasheada.
4. El sistema retorna código 201.

**Flujos alternativos**

- **A1. Datos inválidos (paso 1):** los campos obligatorios no son correctos. Campos vacíos, nombres y apellidos con números/caracteres especiales (exceptuando tildes, espacios, apóstrofes, guiones y la ñ), formato de e-mail incorrecto, minoría de edad, contraseña no segura, términos no aceptados. Retorna código 400.
- **A2. E-mail duplicado (paso 2):** el e-mail ya existe en el sistema. Retorna código 409.
- **A3. Falla técnica a mitad del proceso:** no se guarda ningún dato en el sistema. Se revierte todo y retorna código 500.

**Postcondiciones**

- **Éxito:** el usuario existe, sin cuenta.
- **Fallo:** el usuario no existe.

**Reglas de negocio involucradas**

- La contraseña debe tener un largo mínimo de 10 caracteres, un máximo de 72 bytes e incluir al menos una mayúscula y un carácter especial.
- Contraseña guardada con hash usando BCrypt, no se muestra en responses ni logs.
- El e-mail se normaliza (sin espacios y en minúsculas) antes de compararlo.
- Si llegan dos registros simultáneos con el mismo e-mail, sólo uno se crea (el primero en llegar). Lo garantiza una restricción UNIQUE en la base y el otro recibe código 409.
- El código 201 devuelve id, nombre y e-mail.

---

## CU-02: Abrir cuenta

- **Actor principal:** usuario existente.
- **Objetivo:** tener una cuenta en una moneda.
- **Disparador:** `POST /accounts` con la moneda.

**Precondiciones**

- El usuario está registrado y autenticado.

**Flujo principal**

1. El sistema valida que esté autenticado.
2. El sistema valida que la moneda sea correcta.
3. El sistema valida que no tenga ya una cuenta con esa moneda.
4. El sistema crea la cuenta con saldo en 0, un alias autogenerado, número de cuenta externo (CBU/CVU) y estado activa.
5. El sistema retorna código 201, con id, moneda, saldo, alias, número de cuenta externo y estado.

**Flujos alternativos**

- **A1. Usuario no autenticado (paso 1):** el usuario no está autenticado. Retorna código 401.
- **A2. Moneda inválida (paso 2):** la moneda no es válida. Retorna código 400.
- **A3. Cuenta duplicada (paso 3):** ya existe una cuenta con esa moneda. Retorna código 409.
- **A4. Falla técnica a mitad del proceso:** se revierte todo, la cuenta no es creada y retorna código 500.

**Postcondiciones**

- **Éxito:** existe una cuenta nueva asociada al usuario, con saldo 0, moneda definida, alias y número de cuenta generados y estado activa.
- **Fallo:** no se crea la cuenta.

**Reglas de negocio involucradas**

- Monedas permitidas: ARS, USD.
- Una cuenta por moneda por usuario, garantizado con `UNIQUE (usuario, moneda)`. Si llegan dos pedidos simultáneos, el último en llegar recibe código 409.
- El dueño sale del token, no del cuerpo del pedido. El cliente no manda el id del usuario; lo toma el servidor del JWT.
- Si el token está vencido o es inválido, retorna código 401.
- El número de cuenta externo está compuesto exactamente por 22 dígitos numéricos. El sistema genera el número y verifica que no exista previamente en el sistema. Se garantiza la unicidad con una restricción UNIQUE en la base; si falla, se intenta otro valor.
- El alias es autogenerado con un máximo de 20 caracteres y un mínimo de 6. Debe contener palabras aleatorias de un diccionario, 3 palabras separadas por un punto, ej. `saco.molinete.acecho`. Se valida que no exista previamente en el sistema. Se garantiza la unicidad con una restricción UNIQUE en la base; si falla, se intenta otro valor.
- El alias mantiene el formato en minúsculas por default. Se permiten números, pero no caracteres especiales a excepción del `.`.

---

## CU-03: Iniciar sesión

- **Actor principal:** usuario no autenticado.
- **Objetivo:** autenticar al usuario para que pueda entrar a su cuenta.
- **Disparador:** el usuario envía `POST /auth/login` con sus datos de log-in (e-mail, contraseña).

**Flujo principal**

1. El sistema valida el formato del pedido (campos obligatorios, formato de e-mail).
2. El sistema valida que el acceso no esté bloqueado por intentos.
3. El sistema busca el usuario por e-mail normalizado (sin espacios y en minúsculas).
4. El sistema compara la contraseña con el hash o hace una comparación falsa si el usuario no existe.
5. El sistema genera un token de acceso.
6. El sistema retorna código 200, token de acceso tipo Bearer y tiempo de expiración.

**Flujos alternativos**

- **A1. Pedido mal formado (paso 1):** campos obligatorios mal formados. Retorna código 400.
- **A2. Reintentos máximos (paso 2):** el e-mail o la IP superó el límite de intentos fallidos en la ventana. Retorna código 429 con el mensaje "Se ha excedido el límite de intentos posibles para ingresar. Intente de nuevo más tarde" y la cabecera `Retry-After`.
- **A3. Credenciales inválidas (paso 4):** el e-mail no existe o la contraseña ingresada no es correcta respecto al e-mail presentado. Retorna código 401 y el mensaje "El e-mail o la contraseña no son correctos", y el intento se registra como fallido, sumando 1 al contador de IP y e-mail.
- **A4. Falla técnica en el proceso:** el token de acceso no se puede generar o algo en el proceso falla. El usuario no se loguea y retorna código 500.

**Postcondiciones**

- **Éxito:** se emitió un token de acceso válido para el usuario y se reinicia el contador de intentos del e-mail.
- **Fallo:** no se emitió ningún token.

**Reglas de negocio involucradas**

- Si existía un token no expirado, se crea uno nuevo y el anterior se mantiene hasta que expira.
- Tanto si el e-mail no existe como si la contraseña es incorrecta, retorna el mismo código (401), el mismo mensaje y un tiempo de respuesta comparable, para evitar dar información sobre la base de datos.
- Cada intento fallido suma al contador del e-mail y de la IP, aunque el e-mail no exista. Al llegar a 6 fallidos por e-mail o a 20 por IP dentro de 15 minutos, se bloquean nuevos intentos durante 15 minutos y retorna 429. Retorna 429 durante el bloqueo aun si los datos son correctos.
- No se suman al contador ni extienden el tiempo de bloqueo (si lo tiene) los pedidos mal formados (400), las fallas técnicas (500) y los intentos bloqueados.
- El sistema hace una comparación de BCrypt falsa cuando el usuario no existe, a fin de evitar que agentes malintencionados comparen tiempos entre el caso de e-mail existente y no existente.
- No se registran contraseñas ni tokens.
- Se registran intentos fallidos con un id de seguimiento.
- Un log-in exitoso reinicia el contador de e-mail.
- El token expira a los 15 minutos.

---

## CU-04: Recibir ingreso externo

- **Actor principal:** proveedor externo.
- **Objetivo:** que el proveedor ingrese un monto a una cuenta existente.
- **Disparador:** el proveedor envía `POST /external-deposits`, autenticado con una clave de API y firma, con un id de operación único, monto a enviar, moneda y número de cuenta externo.

**Precondiciones**

- El proveedor está registrado y tiene una clave de API vigente y secreto de firma.

**Flujo principal**

1. El sistema valida que la clave de API y la firma sean válidas.
2. El sistema valida que el monto sea mayor a 0 y que la moneda sea soportada.
3. El sistema verifica que el proveedor no superó el límite de pedidos por minuto (máximo 10 por minuto por proveedor).
4. El sistema verifica si el id de operación ya fue procesado.
5. El sistema busca la cuenta destino por su número de cuenta externo.
6. El sistema valida que la cuenta está activa y que la moneda coincide.
7. El sistema verifica que el saldo de la cuenta de liquidación cubre el monto.
8. El sistema registra un débito en la cuenta de liquidación del proveedor y un crédito en la cuenta destinataria (dos asientos que suman cero) y actualiza ambos saldos.
9. El sistema guarda el ingreso externo como COMPLETADO.
10. El sistema retorna código 201 junto con el id de operación dado por el proveedor (usado como clave de idempotencia), el id de ingreso generado por el sistema, monto, moneda, fecha, cuenta destino y estado.

**Flujos alternativos**

- **A1. Clave de API o firma inválidas (paso 1):** retorna código 401.
- **A2. Monto menor o igual a 0 o moneda no soportada (paso 2):** monto menor o igual a 0, moneda no soportada o campos faltantes. Retorna código 400.
- **A3. Límite de pedidos alcanzado (paso 3):** retorna código 429 y la cabecera `Retry-After`.
- **A4. Mismo id con el mismo contenido (paso 4):** retorna código 200 y devuelve el resultado original.
- **A5. Mismo id con contenido distinto (paso 4):** retorna código 409.
- **A6. Cuenta inexistente (paso 5):** retorna código 404.
- **A7. Cuenta inactiva o la moneda no coincide (paso 6):** retorna código 422.
- **A8. Saldo de la cuenta de liquidación insuficiente (paso 7):** retorna código 422.
- **A9. Falla técnica a mitad del proceso:** se revierte todo y retorna código 500 con un id de seguimiento.

**Postcondiciones**

- **Éxito:** los saldos cambiaron en el monto exacto, existen dos asientos nuevos, el ingreso externo queda guardado como COMPLETADO y la clave de idempotencia queda asociada a su resultado.
- **Fallo:** nada cambia en saldos, asientos ni ingresos.

**Reglas de negocio involucradas**

- La clave de API es única y tiene un tiempo de expiración de 30 días.
- Cada pedido lleva un timestamp y una firma calculada sobre método, ruta, timestamp y cuerpo; se rechaza si el timestamp se desvía más de unos 5 minutos.
- Si falla la clave o la firma, retorna código 401 con el mismo mensaje, sin aclarar cuál falló.
- Si el id de operación es el mismo con contenido distinto, retorna código 409.
- La clave de API se guarda hasheada y se muestra una sola vez al crearla.
- La cuenta de liquidación y la cuenta destino se bloquean siempre en el mismo orden (por id) para evitar bloqueos cruzados.
- Se garantiza la unicidad por id de operación con una restricción `UNIQUE (proveedor, id de operación)` en la base. Si llegan dos pedidos iguales al mismo tiempo, sólo uno se procesa.
- El secreto de firma se guarda cifrado y no se registra en logs.

---

## CU-05: Consultar saldo (en proceso)

## CU-06: Consultar destinatario (en proceso)

## CU-07: Transferir (en proceso)

## CU-08: Historial (en proceso)

## CU-09: Detalle de transferencia (en proceso)