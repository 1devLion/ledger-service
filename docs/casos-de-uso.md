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
  - [CU-05: Consultar saldo](#cu-05-consultar-saldo)
  - [CU-06: Consultar destinatario](#cu-06-consultar-destinatario)
  - [CU-07: Transferir](#cu-07-transferir)
  - [CU-08: Historial (en proceso)](#cu-08-historial-en-proceso)
  - [CU-09: Detalle de transferencia (en proceso)](#cu-09-detalle-de-transferencia-en-proceso)
  - [CU-10: Evaluar políticas de control (límites y anti-fraude)](#cu-10-evaluar-políticas-de-control-límites-y-anti-fraude)
  - [CU-11: Revisar operación (en proceso)](#cu-11-revisar-operación-en-proceso)

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
- **A2. Reintentos máximos (paso 2):** el e-mail o la IP superó el límite de intentos fallidos en la ventana. Retorna código 429 con el mensaje `Se ha excedido el límite de intentos posibles para ingresar. Intente de nuevo más tarde` y la cabecera `Retry-After`.
- **A3. Credenciales inválidas (paso 4):** el e-mail no existe o la contraseña ingresada no es correcta respecto al e-mail presentado. Retorna código 401 y el mensaje `El e-mail o la contraseña no son correctos`, y el intento se registra como fallido, sumando 1 al contador de IP y e-mail.
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

## CU-05: Consultar saldo

- **Actor principal:** usuario autenticado.
- **Objetivo:** que el usuario pueda consultar el saldo de su cuenta.
- **Disparador:** el usuario envía `GET /balance` con el token de autenticación y opcionalmente, la moneda como parámetro `(?currency=ARS)`.

**Precondiciones**

- El usuario está autenticado.

**Flujo principal**

1. El sistema valida que el token sea válido.
2. Si se eligió moneda el sistema valida que sea correcta.
3. El sistema valida que el usuario tenga una cuenta en esa moneda.
4. El sistema retorna código 200 con el saldo de la cuenta, la moneda, fecha y hora de la consulta (UTC) y el estado de la cuenta.

**Flujos alternativos**

- **A1. Token expirado o inválido (paso 1):** retorna código 401.
- **A2. Moneda inválida (paso 2, sólo si se seleccionó moneda):** retorna código 400.
- **A3. Cuenta no existente para esa moneda (paso 3, sólo si se seleccionó moneda):** retorna código 404.
- **A4. Sin moneda y sin cuentas (paso 3):** retorna código 200 con lista vacía.
- **A5. Falla técnica a mitad del proceso:** no se muestra el saldo y retorna código 500 con un id de seguimiento.

**Postcondiciones**

- **Éxito:** se devolvió el saldo; no se modificó ningún dato.
- **Fallo:** no se devolvió el saldo, no se modificó ningún dato.

**Reglas de negocio involucradas**

- El saldo refleja solo operaciones confirmadas hasta el momento de la consulta, sin incluir las que están en curso, y no se sirve desde una caché desactualizada
- El dueño sale del token, no del pedido. Un usuario solo puede ver sus propias cuentas.
- El monto se devuelve con 2 decimales, y no se registran saldos en los logs.
- La respuesta del GET siempre es una lista de saldos, aunque tenga un solo elemento.

---

## CU-06: Consultar destinatario

- **Actor principal:** usuario autenticado.
- **Objetivo:** consultar datos básicos del titular de una cuenta, por alias o número de cuenta, para confirmar a quién se va a enviar el dinero.
- **Disparador:** el usuario envía `GET /accounts/lookup?identifier={alias_or_accountNumber}`, con su token de autenticación.

**Precondiciones**

- El usuario está autenticado.

**Flujo principal**

1. El sistema valida que el token sea válido.
2. El sistema valida que no se haya excedido el límite de consultas.
3. El sistema valida que el formato sea válido (número de cuenta y alias).
4. El sistema valida que la cuenta exista o no este inactiva.
5. El sistema retorna código 200 con nombre, inicial del apellido, alias y moneda.

**Flujos alternativos**

- **A1. Token expirado o inválido (paso 1):** retorna código 401.
- **A2. Límite de consultas excedido (paso 2):** retorna código 429 y la cabecera `Retry-After`.
- **A3. Formato de consulta inválido (paso 3):** retorna código 400.
- **A3. Cuenta inactiva o inexistente (paso 4):** retorna código 404 con un mensaje `La cuenta ingresada no existe`.
- **A4. Falla técnica a mitad del proceso:** no se muestran datos y retorna código 500 con un id de seguimiento.

**Postcondiciones**

- **Éxito:** se devolvieron exitosamente los datos del titular de la cuenta: nombre, apellido, alias y moneda.
- **Fallo:** no se devolvió ningún dato. No se modificó ningún dato.

**Reglas de negocio involucradas**

- El apellido del titular de la cuenta se devuelve abreviado. Sólo se muestra la inicial, p. ej., `Salvattore -> S.`. Si el apellido es compuesto se muestra sólo la primera inicial, p. ej., `Di Tella -> D.`
- El límite de consultas es de 10 cada 30 minutos para usuario (con el id que sale del token) y 20 cada 30 minutos por IP. Cuenta toda consulta que llegue a la búsqueda, exista o no la cuenta. Se define por usuario. Los pedidos mal formados (400) no suman al contador, los bloqueados no extienden el bloqueo.
- Si el titular de la cuenta presenta varios nombres, se muestra sólo el primero.
- El número de cuenta debe tener 22 dígitos números, el alias 3 palabras de 6 a 20 carácteres.
- El alias se normaliza antes de realizar la consulta (sin espacios y en minúsculas).
- No se devuelve nunca id interno, e-mail ni datos personales.
- Consultar usando número de cuenta devuelve también el alias y viceversa.
- Las consultas se registran con un id de seguimiento, sin guardar datos personales, para detectar patrones de abuso.
- Si el identificador es de una cuenta del propio usuario, se devuelve igual.

---

## CU-07: Transferir

- **Actor principal:** usuario autenticado.
- **Objetivo:** mover dinero desde su cuenta a la cuenta de otro usuario.
- **Disparador:** el usuario envía `POST /transfers` con su token de autenticación, el alias o número de cuenta del titular de la cuenta destino, id de operación, moneda, monto y descripción.

**Precondiciones**

- El usuario está autenticado.

**Flujo principal**

1. El sistema valida que el token sea válido.
2. El sistema valida el formato del pedido (campos obligatorios, monto mayor a 0 con 2 decimales como máximo).
3. El sistema valida que la clave de idempotencia no fue usada antes.
4. El sistema valida la existencia y el estado de la cuenta de origen y destino.
5. El sistema valida que la cuenta origen sea diferente a la cuenta destino.
6. El sistema valida que cuenta de origen y la de destino coincidan en la moneda.
7. El sistema inicia la transacción y bloquea ambas cuentas, siempre en el mismo orden (por id) para evitar bloqueos cruzados.
8. El sistema evalúa la operación con las políticas de control (ver CU-10) y obtiene la decisión: aprobada o en revisión.
9. El sistema verifica que el saldo de la cuenta origen cubre el monto.
10. El sistema registra un débito en la cuenta origen y un crédito en la cuenta destino (dos asientos que suman cero).
11. El sistema actualiza ambos saldos.
12. El sistema guarda la transferencia con estado `COMPLETED`, junto con la clave de idempotencia y el resultado.
13. El sistema confirma la transacción.
14. El sistema retorna código 201 con el comprobante, id, monto, moneda, estado, origen (nombre y apellido), destino (nombre e inicial del apellido), fecha y descripción.

**Flujos alternativos**

- **A1. Token inválido o expirado (paso 1):** retorna código 401.
- **A2. Formato del pedido inválido (paso 2):** retorna código 400 con el detalle del campo.
- **A3. Clave ya usada, mismo contenido (paso 3):** no se vuelve a transferir; retorna código 200 con el comprobante original o 202 con estado `UNDER_REVIEW` si lo estuviese.
- **A4. Clave ya usada, contenido distinto (paso 3):** responde 409 por conflicto de idempotencia.
- **A5. Cuenta inexistente o inactiva (paso 4):** retorna código 422.
- **A6. Cuenta origen igual a cuenta destino (paso 5):** retorna código 400.
- **A7. Moneda inválida (paso 6):** el sistema no realiza la transacción y retorna código 422.
- **A8. Operación en revisión (paso 8):** la política de control deja la operación en estado `UNDER_REVIEW` (ver CU-10). El sistema guarda la transferencia como `UNDER_REVIEW`, sin débito ni crédito y asociada a la clave de idempotencia, confirma esa transacción y retorna el código que defina CU-10 para el motivo de la revisión.
- **A9. Saldo insuficiente (paso 9):** se revierte la transacción y retorna código 422. No se escribe ningún asiento.
- **A10. Pedido simultáneo con la misma clave:** solo uno se procesa; el otro recibe el resultado del primero.
- **A11. Falla técnica a mitad del proceso:** se revierte todo, no queda nada a medias y responde 500 con un id de seguimiento, sin exponer detalles internos.

**Postcondiciones**

- **Éxito:** los saldos cambiaron en el monto exacto, existen dos asientos nuevos, la transferencia queda guardada como COMPLETED y la clave de idempotencia queda asociada a su resultado. Se registra el evento para publicar a Kafka.
- **Pendiente de revisión:** la transferencia queda guardada como `UNDER_REVIEW`, sin débito/crédito, y asociada a la clave de idempotencia. Ver CU-10 y CU-11.
- **Fallo:** nada cambió en saldos, asientos ni transferencias.

**Reglas de negocio involucradas:**

- El monto debe estar en `BigDecimal`, mayor a 0, máximo 2 decimales, si son más de 2 decimales, se rechaza.
- La descripción es un campo opcional y puede tener un máximo de 200 caracteres.
- Los asientos son inmutables.
- No se permiten operaciones donde el destino sea igual a la cuenta origen.
- Al existir pedidos simultáneos sólo se procesa el primero en llegar, el resto recibe el resultado del primero. Lo garantiza una restricción `UNIQUE` en la base.
- La cuenta origen no puede transferir un monto más grande al saldo disponible que posee.
- No se permiten transferencias entre cuentas con monedas distintas.
- Las reglas de límites, anti-fraude y revisión no se definen acá: están en CU-10. Este caso de uso solo ejecuta la evaluación y actúa según su decisión.

---

## CU-08: Historial (en proceso)

---

## CU-09: Detalle de transferencia (en proceso)

---

## CU-10: Evaluar políticas de control (límites y anti-fraude)

- **Actor principal:** sistema (se ejecuta dentro de CU-07).
- **Objetivo:** decidir si una operación se aprueba o queda en revisión, según los límites y el anti-fraude.
- **Disparador:** CU-07 invoca la evaluación con la cuenta origen, la cuenta destino, el monto y la moneda de la operación.

**Precondiciones**

- La cuenta origen está bloqueada dentro de una transacción en curso (para que los totales sean consistentes).

**Flujo principal**

1. El sistema valida que las operaciones del usuario no sean sospechosas (anti-fraude).
2. El sistema valida que las operaciones diarias totales no pasen la restricción de $50.000.000 ARS diarios.
3. El sistema valida que la operación individual no pase los $2.500.000 ARS permitidos.
4. El sistema devuelve a CU-07 la decisión: aprobada.

**Flujos alternativos**

- **A1. Bloqueo por operaciones sospechosas (paso 1):** si el sistema detecta un posible intento de fraude, CU-07 retorna código 400 y se pone el estado de la cuenta y la transferencia `UNDER_REVIEW`.
- **A2. Operaciones superan el monto máximo diario total permitido (paso 2):** el sistema deja en estado `UNDER_REVIEW` la transacción y CU-07 retorna código 202.
- **A3. Operación individual supera el monto máximo individual permitido (paso 3):** el sistema deja en estado `UNDER_REVIEW` la transacción y CU-07 retorna código 202.

**Postcondiciones**

- **Aprobada:** no se modifica ningún dato; CU-07 continúa con la verificación de saldo.
- **En revisión:** la operación y/o la cuenta entran en estado `UNDER_REVIEW`, sin débito/crédito. Todas las acciones futuras de débito/crédito entran en estado `UNDER_REVIEW`. Un auditor decide después sobre la cuenta y las operaciones pendientes de aprobación (ver CU-11).

**Reglas de negocio involucradas:**

- Si una transacción queda en estado `UNDER_REVIEW` y la cuenta no está en estado `UNDER_REVIEW`, el usuario puede seguir operando, pero esa transacción cuenta en la suma al total diario. Se aprueba si el total sin contar esa transacción no excede el límite, o el auditor puede autorizar el exceso.
- Si una transferencia está en estado `UNDER_REVIEW`, el monto total de esa transferencia sea débito o crédito es retenido y no se verá reflejado en la cuenta origen ni en la cuenta destino.
- Existe un sistema anti-fraude que detecta patrones sospechosos de operaciones y bloquea las transacciones indefinidamente. Las sospechas pueden surgir de, p. ej., varias transferencias a la misma cuenta destino en poco tiempo; una operación de un monto inusual para la cuenta del usuario, etc. La cuenta pasa a estar en estado `UNDER_REVIEW`, las operaciones sospechosas y futuras quedan registradas y en estado `UNDER_REVIEW` y la cuenta la deberá revisar un auditor el cual podrá decidir si es suspendida, elevada a autoridades financieras o queda en estado `ACTIVE`.
- Si el sistema anti-fraude pausa una cuenta, hasta que la cuenta no entre en estado `ACTIVE` ninguna operación puede cambiar de estado.
- Si una operación individual supera el monto de $2.500.000 ARS, esta entra en estado `UNDER_REVIEW` a revisar por un auditor.
- Si el monto total de las operaciones diarias de la cuenta origen supera el máximo permitido ($50.000.000 ARS) esta transacción entra en estado `UNDER_REVIEW`, donde un auditor debe decidir si aprobar esta transferencia, o informarla a las autoridades financieras para obtener una respuesta antes de avanzar en ella. Se toma 'diario' como las últimas 24 horas.
- Si la operación excede el monto por operación individual de $2.500.000 y el monto diario total de $50.000.000, sólo se sigue el flujo del monto diario total.

---

## CU-11: Revisar operación (en proceso)

---
