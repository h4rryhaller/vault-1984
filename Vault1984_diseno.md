# Vault 1984: documento de diseño

> «El Gran Hermano te vigila.» Una bóveda contra él.

## 1. Qué es

**Vault 1984** es la evolución del [Ardutype](https://github.com/h4rryhaller/ardutype): un gestor de contraseñas que las **teclea** como un teclado USB y que además lleva un **disco cifrado** en una microSD. Se desbloquea en el propio aparato (joystick y pantalla), y el mismo desbloqueo abre las contraseñas y el disco.

- **Por cable, sin radio** (decisión del usuario, 2026-11-15): nada de wifi ni Bluetooth, como el Ardutype. Las ideas con radio quedan en la sección 9.
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
6. **Copias de seguridad del disco:** ¿copiar la microSD cifrada tal cual (imagen) basta?

## 8. Compras

- 2-3 × Waveshare RP2350-Zero (revisión A4 si se puede).
- 1 × OLED 1,3" 128 × 64 I2C (SH1106 o SSD1306).
- 1 × breakout microSD de 3,3 V sin regulador.
- 1 × piezo pasivo de 9-12 mm.
- 1 × microSD de prueba (sin datos que importen).
- Ya disponibles: ATECC608B (y adaptador SOIC-8), joystick, resistencias de 100 Ω y 4,7 kΩ, condensador de 100 nF.

## 9. Ideas para más adelante (con radio)

Descartadas de momento (2026-11-15). Si algún día se usara un ESP32-S3, con la radio apagada por defecto y activada a propósito: teclado Bluetooth (móvil, tablet, televisor sin cables), bloqueo por proximidad (al alejarse el móvil bloquea el PC y el aparato), hora por NTP para generar códigos TOTP, copias automáticas de lo cifrado a un NAS, actualizaciones por wifi firmadas.
