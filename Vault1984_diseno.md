# Vault 1984: documento de diseño

> «El Gran Hermano te vigila.» Una bóveda contra él.

## 1. Qué es

**Vault 1984** es la evolución del [Ardutype](https://github.com/h4rryhaller/ardutype): un gestor de contraseñas que las **teclea** como un teclado USB y que además lleva un **disco cifrado** en una microSD. Se desbloquea en el propio aparato (joystick y pantalla), y el mismo desbloqueo abre las contraseñas y el disco.

- **Por cable, sin radio** (decisión del usuario, 2026-11-15): nada de wifi ni Bluetooth, como el Ardutype. Las ideas con radio quedan en la sección 10.
- **Proyecto para aprender**, pero pensado para usarse en la práctica.
- **Hereda todo el Ardutype:** bóveda de contraseñas con el ATECC608B limitando los intentos del PIN, clave de recuperación de 8 palabras, guiones, sincronización con KeePassXC, pomodoro, mantener sesión, `subir.sh` y las copias de seguridad.
- **Lo nuevo:** el disco cifrado, una pantalla más grande, y el **arranque seguro** del RP2350, que cierra el punto más grave de la revisión de seguridad del Ardutype (firmware no autorizado).

El nombre: el refugio numerado de *Fallout* («Vault 101», «Vault 111») con el número de la novela de Orwell.

## 2. Componentes (propuesta, 2026-11-15)

| Pieza | Elección | Por qué |
|---|---|---|
| Placa | **Waveshare RP2350-Zero** (~23 × 18 mm, USB-C, LED RGB WS2812 en la placa) | Más pequeña que la KB2040 (descatalogada), con LED propio y pines de sobra. Comprar **2-3**: el arranque seguro se prueba grabando la OTP, sin vuelta atrás, en una de sacrificio. Si se puede, revisión del chip **A4** (la A2 tiene la errata E9 con los *pull-down* internos; usamos *pull-up*, pero mejor evitarla). Alternativas: Pico 2 (más grande, micro-USB), XIAO RP2350 (justa de pines) |
| Pantalla | **OLED de 1,3" 128 × 64, I2C** (SH1106 o SSD1306) | El doble de alta que la del Ardutype (4 filas de menú en vez de 2) con la misma librería (U8g2): cambio mínimo de código. Una TFT a color obligaría a rehacer el dibujo (pregunta abierta) |
| Almacenamiento | **Módulo microSD de 3,3 V, sin regulador ni adaptador de niveles** (p. ej. el breakout microSD «3V only» de Adafruit), por **SPI** | Pequeño. El SPI da más velocidad de la que el USB puede aprovechar |
| Seguridad | **ATECC608B** en su propio bus I2C con pull-ups de 4,7 kΩ | Como en el Ardutype (quedan chips de sobra). Limita los intentos del PIN con un contador que no se puede rebobinar |
| Entrada | **Joystick de 5 direcciones**, el mismo | |
| Sonido | **Piezo pasivo pequeño** (9-12 mm) con **100 Ω** | Como el del Ardutype, más pequeño |
| LED | El WS2812 de la placa | |

**Pines necesarios:** pantalla 2 (I2C1), ATECC 2 (I2C0), joystick 5, zumbador 1, microSD 4 (SPI0) + 1 opcional (detección de tarjeta) = **14-15**. La asignación concreta, al tener la placa y su pinout.

**Lecciones del Ardutype que se mantienen:** el OLED y el ATECC en **buses I2C separados** (compartiendo bus, el tráfico de la pantalla despertaba al chip a mitad de una orden); pull-ups externas de 4,7 kΩ en el bus del ATECC (las internas no bastaron); aislar las patas del chip.

## 3. El disco cifrado

### Cómo funciona

- El aparato se presenta por USB como **teclado + ratón + puerto serie + disco** (*mass storage*, TinyUSB).
- **Bloqueado:** el PC ve un lector **sin tarjeta** (unidad sin medio).
- **Desbloqueado:** aparece el disco. Cada sector de 512 bytes que pide el PC se lee de la microSD y se descifra al vuelo con **AES-XTS** (el modo de BitLocker, LUKS y VeraCrypt, con el número de sector como *tweak*); al escribir, al revés. La microSD solo guarda datos cifrados.
- El PC formatea el disco virtual (FAT32 o exFAT) como cualquier pendrive: funciona en Linux, Windows, macOS o un televisor **sin instalar nada**.
- **Clave del disco:** aleatoria (TRNG del RP2350 y del ATECC), de 512 bits (AES-256-XTS usa dos claves), envuelta con la misma clave maestra que la bóveda de contraseñas.

### Límites conocidos

- **Velocidad: ~1 MB/s como mucho.** El RP2350 solo tiene USB *Full Speed* (12 Mbit/s) y no tiene AES por hardware. Bien para documentos, claves y fotos sueltas; no para vídeo. Hay que medirlo en la fase 3.
- **XTS no autentica:** quien manipule la microSD puede estropear sectores sin que se note (como en todo cifrado de disco). Sí impide leerlos.
- **Mientras el disco está abierto, el PC ve los datos en claro**, como con cualquier disco cifrado.

### Ideas para el almacenamiento (2026-11-16)

Propuesta del usuario, con la valoración de esfuerzo:

1. **Modo solo lectura** (para ordenadores no fiables). *Esfuerzo bajo.* TinyUSB pregunta al firmware si la unidad admite escritura (`tud_msc_is_writable_cb`) y el firmware rechaza las escrituras: lo hace cumplir el aparato, no la buena voluntad del PC. Protege la **integridad** (nada se escribe en el disco), no la **confidencialidad**: un PC no fiable puede copiar todo lo que se abra. Ventaja de serie: el PIN se mete en el aparato, así que un *keylogger* del PC no lo ve.
2. **Varias zonas en la microSD: utilidades sin cifrar + documentos cifrados.** *Esfuerzo medio, encaja con las fases 3-4.* En vez de una tabla de particiones, el aparato presenta **varias unidades lógicas (LUN)** de TinyUSB, cada una sobre un rango de sectores de la tarjeta: el PC ve dos discos. La de utilidades está siempre presente (también bloqueado); la cifrada aparece al desbloquear. Cada una se puede hacer de solo lectura por separado.
3. **Dos zonas cifradas: diario y archivo** (aclarado por el usuario, 2026-11-16). No se trata de ocultar nada ante una coacción, sino de **no exponer lo importante en el uso diario**. *Esfuerzo bajo-medio sobre el punto 2:* es otra LUN cifrada.
   - **Diario:** se abre con el desbloqueo normal; documentos de poco riesgo.
   - **Archivo:** copia de la documentación personal (escrituras, hipoteca, copias de documentos). Cerrado por defecto; se abre a propósito desde el menú, con un **segundo PIN**, y aparece como un disco más.
   - **Solo lectura por defecto**, con escritura solo si se pide al abrirlo: un *ransomware* o un borrado accidental en el PC no lo alcanza.
   - **Se cierra solo** tras unos minutos sin uso, o al bloquear.
   - **Claves distintas** para cada zona; el segundo PIN con su propio slot y contador de intentos en el ATECC.
   - Sin pretensión de negación: las zonas pueden tener tamaño fijo y verse en el menú. La negación creíble (volumen oculto, PIN de coacción) queda descartada de momento: exige rellenar la tarjeta con azar, un señuelo verosímil y usado, y aun así se delata comparando dos volcados de la tarjeta.
4. **Arrancar un PC con imágenes de la zona de utilidades.**
   - **Ventoy** en la zona de utilidades: se copian las ISO y Ventoy da el menú de arranque. No hace falta emular nada. Es lo recomendado.
   - **Emular un CD-ROM USB** (una ISO como unidad de CD, como hacen IODD o DriveDroid): posible con TinyUSB, porque una LUN puede presentarse como CD y responder a sus órdenes SCSI (sectores de 2048 bytes, TOC). *Esfuerzo medio-alto*; solo si Ventoy no basta. Ventaja: la imagen es de solo lectura por naturaleza.
   - **Límite serio: la velocidad.** A ~1 MB/s, una ISO de 1 GB tarda unos 17 minutos en leerse. Vale para sistemas pequeños (TinyCore, Alpine, herramientas de rescate pequeñas); no para una distribución completa en vivo. Para eso, mejor un pendrive normal.

## 4. Seguridad

Parte de la revisión de seguridad del Ardutype (2026-11-15). Lo que cambia:

1. **Firmware no autorizado → arranque seguro del RP2350.** Solo arranca firmware firmado con nuestra clave (su resumen, grabado en la OTP); se puede desactivar la depuración (SWD) y restringir el arranque por USB; detectores de *glitch*. Riesgo: la OTP no tiene vuelta atrás; perder la clave de firma deja la placa sin poder actualizarse. Hay que guardar la clave de firma como la clave de recuperación.
2. **Escucha del bus del ATECC:** sigue igual que en el Ardutype (quien ponga un analizador en el bus mientras se desbloquea ve el valor que, con un volcado de la flash, abre la bóveda). Estudiar si el RP2350 puede cifrar ese tráfico con la *IO protection key* del ATECC608B, y cubrir el chip con resina en la versión final.
3. **Desatendido y desbloqueado lo da todo:** ahora también el disco. Bloqueo automático corto, y pensar qué pasa con el disco montado (pregunta abierta).
4. **Sin auditoría:** el cifrado del disco se compone a mano (AES de BearSSL en modo XTS); probarlo contra los vectores de prueba oficiales del IEEE 1619.

## 5. Firmware: qué se reutiliza

El core es el mismo (arduino-pico de Earle Philhower, que ya soporta el RP2350), así que casi todo el código del Ardutype pasa tal cual: `boveda`, `cripto`, `atecc`, `almacen`, `admin`, `guion`, `teclado`, `pomodoro`, `mantener`, `zumbador`, `led`, `joystick`, `utf8`, y la herramienta de sincronización. Cambian `pantalla` y `ui` (128 × 64: 4 filas) y la comprobación del Flash Size de `config.h` (otra placa, otra memoria). Nuevos: `disco` (TinyUSB *mass storage* + AES-XTS) y `sd` (acceso a la microSD por SPI).

## 6. Fases

1. **Prototipo en protoboard:** placa, OLED de 1,3", joystick, ATECC608B, piezo y microSD. Sketch de prueba de cada pieza.
2. **Portar el Ardutype** al RP2350 y a la pantalla de 4 filas. Que funcione igual que el Ardutype.
3. **Disco USB sin cifrar:** el PC ve la microSD tal cual. Aprender TinyUSB, comprobar que convive con teclado, ratón y puerto serie, y **medir la velocidad**.
4. **Disco cifrado:** AES-XTS por sector (con los vectores de prueba del IEEE 1619), clave envuelta en la bóveda; bloqueado = sin tarjeta.
5. **Arranque seguro**, primero en una placa de sacrificio: firmar con `picotool`, grabar la OTP, desactivar la depuración.
6. **PCB y carcasa.**

## 7. Preguntas abiertas

1. **Pantalla:** ¿OLED monocromo de 128 × 64 (propuesta) o TFT a color?
2. **¿Dónde va la clave del disco?** (a) Solo en la bóveda de la flash interna: la microSD sola no sirve de nada, pero si se pierde el aparato se pierden los datos (salvo copia con `respaldo.sh`). (b) También en una cabecera de la microSD, envuelta con la clave de recuperación: los datos se recuperan con otro Vault 1984, a cambio de que la tarjeta lleve un ataque posible contra la clave de recuperación (84 bits con PBKDF2: no se rompe por fuerza bruta).
3. **Bloqueo con el disco montado:** si el aparato se bloquea solo y el disco desaparece de golpe, el PC puede perder lo que tenga pendiente de escribir. ¿No bloquear mientras haya escrituras recientes? ¿Avisar antes? ¿Expulsar de forma ordenada?
4. **Disco y USB en el core:** el core trae `FatFSUSB` (expone un FatFS como disco USB); para cifrar hace falta trabajar con sectores en crudo (las funciones `tud_msc_*` de TinyUSB). Comprobar cómo hacerlo sin chocar con el core.
5. **Arranque seguro con arduino-pico:** flujo para firmar (`picotool seal --sign`) los binarios que genera el core, y cómo encaja con `subir.sh`.
6. **Copias de seguridad del disco:** ¿copiar la microSD cifrada tal cual (imagen) basta? Importa sobre todo para la zona de archivo (sección 3, idea 3): el aparato no puede ser el único sitio donde estén esos documentos.
7. **Hora para TOTP (9.6):** ¿la da el PC al conectar o un reloj propio con pila (DS3231)?

## 8. Compras

- 2 × **Waveshare RP2350-Zero** (2026-11-15), sin pines soldados (llega antes y se le sueldan dos tiras para la protoboard; la -M es la misma con los pines ya soldados): USB-C **hembra** y 4 MB de flash. Sin wifi ni Bluetooth (los anuncios mezclan datos de la Raspberry Pi Zero 2 W). Una para trabajar y otra de sacrificio para el arranque seguro. Los anuncios no dicen la revisión del chip (A2/A4): no afecta, usamos *pull-up*.
  - Descartada de momento la **RP2350-Zero-CM** (USB-C **macho**, se enchufa al PC como un pendrive, 2 MB de flash): pulsar el joystick con el aparato clavado en el puerto hace palanca sobre los conectores, y solo encaja en puertos USB-C. Se puede valorar para la fase 6 si se quiere forma de pendrive.
- 1 × OLED 1,3" 128 × 64 I2C (SH1106 o SSD1306).
- 1 × breakout microSD de 3,3 V sin regulador.
- 1 × piezo pasivo de 9-12 mm.
- 1 × microSD de prueba (sin datos que importen).
- Ya disponibles: ATECC608B (y adaptador SOIC-8), joystick, resistencias de 100 Ω y 4,7 kΩ, condensador de 100 nF.

## 9. Navaja suiza: funciones extra

La idea (2026-11-16): que el aparato sea una navaja suiza de privacidad y seguridad, además de bóveda y disco. Ya existen pomodoro y mantener sesión; estas son las siguientes. Ninguna necesita radio, y casi todas se pueden probar antes en el Ardutype.

### 9.1 Aleatoriedad

Propuesta del usuario:

| Función | Opciones | Ejemplo |
|---|---|---|
| Lanzar moneda | número de monedas | 5 → 0, X, X, 0, 0 |
| Lanzar dados | número de dados | 2 → 5, 4 |
| Generar contraseña | longitud; conjuntos a-z, A-Z, 0-9, símbolos; excluir parecidos (i, l, 1, O, 0) | |
| Contraseña Diceware | número de palabras | |
| Generar números | desde, hasta, cuántos, permitir repetidos | 1-99, 6 → 43, 58, 7, 44, 90, 23 |
| Generar color | ruleta en el LED y el código en pantalla (#022D5B), primero rápida y frenando poco a poco | |
| Formatos fijos | alfanumérico 32, token 64 (a-z0-9), hexadecimal 32, UUID | |

Siempre con la opción de **teclear el resultado**.

Notas de diseño:

- **La fuente de azar es lo que importa.** Nada de `random()` sembrado con `millis()`: usar el **generador de números aleatorios del ATECC608B** (orden *Random*, 32 bytes) y el **TRNG del RP2350**, mezclados con SHA-256. En el Ardutype (RP2040, sin TRNG de verdad) basta el del ATECC.
- **Sin sesgo de módulo:** para elegir entre N valores, descartar y repetir (*rejection sampling*) en vez de `azar % N`. Un dado con `% 6` sale sesgado.
- **Mostrar la entropía** (bits) de cada contraseña generada, para elegir la longitud con criterio. Referencias: Diceware 12,9 bits por palabra (6 palabras ≈ 77 bits); alfanumérico 32 ≈ 190 bits; hexadecimal 32 = 128 bits; UUID v4 = 122 bits.
- **Diceware necesita una lista de palabras** en la flash (la grande de la EFF: 7776 palabras, ~60 KB, licencia CC BY). Si se usa una en español, **sin tildes ni ñ**, para que se teclee igual con cualquier distribución de teclado. Elegir la lista es una pregunta abierta.
- **Teclear evita el portapapeles** (sin historial ni otros programas leyéndolo): es un punto a favor frente a copiar y pegar.
- **Riesgo: generar una contraseña y perderla.** Si se teclea en un formulario de alta y no se guarda en KeePassXC, se pierde la cuenta. El resultado se queda en pantalla hasta que se descarta a propósito, y luego se borra de la RAM. (Guardarla directamente en la bóveda queda para más adelante: la sincronización va de KeePassXC al aparato, no al revés.)
- **Pantalla:** a 21 caracteres por fila, un token de 64 ocupa 4 filas y un UUID 2. Se ve, pero justo.
- **Meter números con el joystick** es pesado: incrementos con aceleración al mantener pulsado, valores por defecto razonables y recordar los últimos usados.
- No hace falta desbloquear: no toca secretos guardados.

### 9.2 Notificaciones y confirmaciones

La idea (2026-11-16): que el aparato avise (pitido y LED) y pregunte **aceptar o denegar**, desde cualquier sitio del PC: scripts de bash y Claude Code cuando pide permiso y el usuario está en otro escritorio. Patrón conocido: la confirmación en un aparato aparte, como una cartera hardware o el toque de una YubiKey.

- **Protocolo** por el puerto serie USB, una orden por línea: el PC manda `PREGUNTA <id>|<origen>|<texto>`, el aparato pita, parpadea y lo enseña; el joystick elige y responde `RESPUESTA <id> SI|NO`.
- **Script `preguntar "texto"`**: sale con 0 si se acepta, 1 si se deniega y 2 si se acaba el tiempo o no hay aparato. Usa `flock` sobre el puerto, que solo admite un programa a la vez (también lo usa la sincronización).
- **Claude Code:** sus *hooks* permiten responder a la petición de permiso desde un comando (si no responde, se pregunta en la terminal como siempre) y avisar cuando espera respuesta o termina. Comprobar el formato exacto antes de implementarlo.
- **Reglas:**
  - Es comodidad, no seguridad: un programa malicioso en el PC se salta el aparato.
  - Sin respuesta a tiempo = sin respuesta, nunca aceptar.
  - El texto completo debe poder leerse (desplazar con el joystick) antes de aceptar.
  - Bloqueado: avisa, se puede denegar, pero aceptar exige desbloquear con el PIN.
- **Orden:** 1) solo avisar; 2) preguntar y responder con `preguntar`; 3) responder a los permisos de Claude Code. Se puede hacer primero en el Ardutype, que ya tiene puerto serie, buzzer, LED, pantalla y joystick.

### 9.3 Desbloqueo con gesto («código Konami»)

La idea (2026-11-16): desbloquear con una secuencia de movimientos del joystick en vez de elegir cifras en pantalla, para que el PIN no se vea.

**Decisión del usuario (2026-11-16): se mantiene el PIN de cifras** (cómodo y bien resuelto). El gesto queda como idea apuntada, como mucho opcional.

- **Es otra forma de escribir el PIN, no otro mecanismo:** la secuencia se convierte en texto (`AAbbIDID…`) y entra por el mismo camino que el PIN de cifras (KDF + ATECC con su contador de intentos). La seguridad la sigue poniendo el ATECC.
- **Longitud:** con 4 direcciones, cada movimiento da 2 bits; **10 movimientos = 4¹⁰ ≈ 10⁶**, lo mismo que el PIN de 6 cifras actual. El código Konami (↑↑↓↓←→←→ + 2) mide justo eso.
- **El centro confirma**; una pulsación larga del centro borra y empieza de nuevo. La izquierda deja de servir para volver atrás, porque es parte del código.
- **Pantalla:** solo un punto por movimiento, sin dirección. Mismo pitido corto para todos, sin colores por dirección en el LED.
- **Más rápido:** 10 movimientos frente a unas 20-25 pulsaciones del PIN de cifras (subir y bajar cada cifra y confirmarla).
- **Límite:** quien mire la mano puede ver los movimientos, aunque cuesta más que leer una cifra en pantalla y se puede tapar con la otra mano.
- **Riesgo práctico:** un rebote o una diagonal mal hecha cuenta como fallo y gasta un intento del ATECC. Probar bien el antirrebote y que solo cuente al volver el joystick al centro.
- Se puede probar antes en el Ardutype; allí cambiar de PIN a gesto es «Cambiar PIN» (vuelve a envolver la clave, no borra la bóveda), pero está en producción: hacer antes `respaldo.sh`.

### 9.4 Teclado y ratón de emergencia

La idea (2026-11-16): usar el aparato como teclado o ratón cuando no hay otro (BIOS, un PC sin teclado, un servidor).

- **Modo teclado:** el joystick son las flechas y el centro es Intro; un menú con teclas sueltas: Esc, Tab, Supr, F1-F12 (entrar en la BIOS o el menú de arranque), Ctrl+Alt+Supr, Ctrl+Alt+F1…F7 (cambiar de consola en Linux), Alt+PetSis+REISUB (reiniciar con orden un Linux colgado).
- **Modo ratón:** el joystick mueve el puntero con aceleración; centro = clic, centro largo = clic derecho.
- **Salir del modo:** pulsación larga del centro (las direcciones están ocupadas).
- **BIOS:** muchas BIOS solo entienden teclados en *boot protocol*, y el aparato es un dispositivo compuesto (teclado + ratón + serie + disco). Comprobar en la BIOS real; si falla, ver si TinyUSB puede declarar el teclado como *boot keyboard*. Se puede probar ya con el Ardutype.
- No hace falta desbloquear: no toca secretos (cualquier teclado puede hacer lo mismo).

### 9.5 Interruptor del hombre muerto

La idea (2026-11-16): si el usuario no da señales cada cierto tiempo, se lanza una acción en el PC (bloquear, hibernar).

- **La señal se da en el aparato** (pulsar el centro), no en el PC. Diferencia con el bloqueo por inactividad del escritorio (xss-lock): a ese lo reinicia cualquiera que toque el PC, y también el propio «mantener sesión». Este solo lo reinicia quien tenga el aparato. Útil precisamente con «mantener sesión» activado.
- **Aviso antes de actuar:** pitido y LED un minuto antes; una pulsación lo aplaza.
- **Cómo actúa, de más sencillo a más completo:**
  1. **Un atajo de teclado** (p. ej. Super+L o uno propio de sxhkd que hiberne). Sin programa en el PC. Mejor un atajo que teclear un comando: escribir texto a ciegas en la ventana que tenga el foco es peligroso.
  2. **Una orden por el puerto serie** a un pequeño servicio del PC (el mismo de las notificaciones, 9.2), que ejecuta el script que se configure y confirma que lo ha hecho.
- **Hibernar protege más que bloquear solo si el disco está cifrado** (LUKS, también la swap): al hibernar, la clave del disco sale de la RAM.
- **Variante relacionada (tipo BusKill):** si se desenchufa el aparato, el PC se bloquea. Es una regla de udev al desconectar `239a:8106`; no necesita firmware.
- Se puede hacer ya en el Ardutype (la opción 1).

### 9.6 Códigos TOTP

La idea (2026-11-16): generar los códigos de 6 cifras de la verificación en dos pasos y teclearlos.

- **Cómo funciona:** HMAC-SHA1 del secreto de la cuenta con la hora (contador cada 30 s); el resultado se recorta a 6 cifras. Poco código.
- **Hace falta la hora exacta (±30 s), y es la decisión importante** (pregunta abierta 7):
  - **Que la dé el PC** al conectar, por el puerto serie. Sin piezas nuevas, pero un PC malicioso podría poner una hora futura, pedir códigos y usarlos más tarde.
  - **Un reloj propio con pila** (DS3231, I2C, muy preciso): el aparato no depende del PC y puede rechazar saltos de hora sospechosos. Una pieza más, en el bus del OLED.
- **Los secretos vienen de KeePassXC**, que ya guarda TOTP en las entradas: la sincronización los traería con las contraseñas.
- **Matiz:** con la contraseña y el TOTP en el mismo aparato (o en la misma base de KeePassXC), los dos factores pasan a ser uno. Cómodo, pero menos seguro que tenerlos separados.

### 9.7 FIDO2 / llave de seguridad

La idea (2026-11-16): que el aparato funcione como una llave de seguridad (tipo YubiKey) para webs y SSH.

- **Encaja muy bien con el hardware:** FIDO2 firma con ECDSA P-256, que el **ATECC608B hace por hardware** con claves que nunca salen del chip; la «presencia del usuario» es pulsar el centro del joystick.
- **Esfuerzo alto:** otra interfaz HID (CTAPHID), mensajes CBOR, gestión de credenciales, PIN de FIDO2. Hay un proyecto libre que lo hace en RP2040/RP2350 (**pico-fido**) como referencia.
- **Camino de aprendizaje:** primero **U2F** (CTAP1, el protocolo antiguo: solo segundo factor, mucho más sencillo y aceptado por casi todas las webs), luego FIDO2.
- **Riesgos:** una llave casera no está certificada ni auditada; y si es la única llave de una cuenta y se pierde el aparato, se pierde la cuenta: registrar siempre una segunda llave o códigos de respaldo.

### 9.8 Claves SSH y certificados

La idea (2026-11-16): llevar e introducir claves SSH o certificados.

- **Teclear una clave privada** la deja en el PC (en pantalla, historial, disco): solo para preparar un equipo propio, nunca en uno ajeno.
- **Teclear la clave pública** (para `authorized_keys`) no tiene riesgo y es útil.
- **Llevarlas como archivos en el disco cifrado** (zonas de diario o archivo, sección 3, idea 3): sin trabajo extra una vez hecho el disco.
- **Lo mejor, gratis con FIDO2:** OpenSSH admite claves `ecdsa-sk` / `ed25519-sk` respaldadas por una llave FIDO2. La clave privada no sale del aparato y cada conexión pide pulsar el centro. Es la forma correcta de «inyectar» SSH.

### 9.9 Prioridades propuestas (2026-11-16)

Primero lo que se puede probar ya en el Ardutype y cuesta poco: notificaciones (9.2, avisar), interruptor del hombre muerto con atajo (9.5), teclado de emergencia (9.4), aleatoriedad (9.1). Después, con la Vault 1984: disco cifrado, TOTP con reloj propio, U2F → FIDO2 (y con él SSH).

## 10. Ideas para más adelante (con radio)

Descartadas de momento (2026-11-15). Si algún día se usara un ESP32-S3, con la radio apagada por defecto y activada a propósito: teclado Bluetooth (móvil, tablet, televisor sin cables), bloqueo por proximidad (al alejarse el móvil bloquea el PC y el aparato), hora por NTP para generar códigos TOTP, copias automáticas de lo cifrado a un NAS, actualizaciones por wifi firmadas.
