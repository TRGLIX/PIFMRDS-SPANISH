# PIFMRDS-SPANISH
Pi-FM-RDS
=========


## Transmisor FM-RDS usando la Raspberry Pi

Este programa genera una modulación FM, con datos RDS (Radio Data System) generados en tiempo real. Puede incluir audio monofónico o estereofónico.

Está basado en el transmisor FM creado por Oliver Mattos y Oskar Weigl, y posteriormente adaptado para usar DMA por [Richard Hirst](https://github.com/richardghirst). Christophe Jacquet lo adaptó y añadió el generador y modulador de datos RDS. El transmisor usa el generador PWM de la Raspberry Pi para producir señales VHF.

Es compatible tanto con la Raspberry Pi 1 (la original) como con Raspberry Pi 2, 3, 4 y Zero.

![](doc/vfd_display.jpg)

PiFmRds ha sido desarrollado solo para experimentación. No es un centro multimedia, y no está pensado para transmitir música a tu equipo de sonido. Consulta la [advertencia legal](#advertencia-y-aviso).

## ¿Cómo usarlo?

Pi-FM-RDS se ha probado en Raspberry Pi OS Lite (la versión sin entorno de escritorio). Debería poder ejecutarse en otras distribuciones, pero el autor no ofrece garantías ni soporte. Última versión de Raspberry Pi OS probada con éxito: 13.1 (basada en Debian Trixie).

Dependencias:

* Biblioteca `sndfile`, provista por el paquete `libsndfile1-dev` en Raspberry Pi OS y otras distribuciones tipo Debian.
* Controlador de Linux `rpi-mailbox`, por lo que necesitas un kernel de Linux compilado después de aproximadamente agosto de 2015.

Importante: los binarios compilados para un modelo de Raspberry Pi y una arquitectura ARM no son compatibles con otros modelos y arquitecturas. Siempre recompila al cambiar de modelo, ¡así que no te saltes el paso `make clean` en las instrucciones a continuación!

Estas deberían ser las instrucciones completas asumiendo una instalación limpia de Raspberry Pi OS Lite:

```bash
sudo apt install git libsndfile1-dev
git clone https://github.com/ChristopheJacquet/PiFmRds.git
cd PiFmRds/src
make clean
make
```

Importante: si `make` informa algún error, entonces no se genera el ejecutable `pi_fm_rds` (y viceversa). Por tanto, cualquier error debe corregirse antes de continuar. `make` puede fallar si falta alguna biblioteca requerida (ver más arriba), o por un bug en una distribución más nueva/específica. En ese caso, por favor reporta un bug.

Si `make` no informa errores (es decir, se genera el ejecutable `pi_fm_rds`), entonces puedes ejecutar simplemente:

```
sudo ./pi_fm_rds
```

Esto generará una transmisión FM en 107.9 MHz, con el nombre de emisora (PS), radiotexto (RT) y código PI por defecto, sin audio. La señal de radiofrecuencia se emite por el GPIO 4 (pin 7 en el conector P1).


Puedes añadir audio mono o estéreo referenciando un archivo de audio de la siguiente forma:

```
sudo ./pi_fm_rds -audio sound.wav
```

Para probar audio estereofónico, puedes usar el archivo `stereo_44100.wav` incluido.

La sintaxis más general para ejecutar Pi-FM-RDS es la siguiente:

```
pi_fm_rds [-freq freq] [-audio file] [-ppm ppm_error] [-pi pi_code] [-ps ps_text] [-rt rt_text]
```

Todos los argumentos son opcionales:

* `-freq` especifica la frecuencia portadora (en MHz). Ejemplo: `-freq 107.9`.
* `-audio` especifica un archivo de audio para reproducir como audio. La tasa de muestreo no importa: Pi-FM-RDS la remuestrea y filtra. Si se proporciona un archivo estéreo, Pi-FM-RDS generará una señal FM-Stereo. Ejemplo: `-audio sound.wav`. Los formatos soportados dependen de `libsndfile`. Esto incluye WAV y Ogg/Vorbis (entre otros) pero no MP3. Especifica `-` como nombre de archivo para leer audio desde la entrada estándar (útil para canalizar audio hacia Pi-FM-RDS, ver más abajo).
* `-pi` especifica el código PI de la transmisión RDS. 4 dígitos hexadecimales. Ejemplo: `-pi FFFF`.
* `-ps` especifica el nombre de la emisora (Program Service name, PS) de la transmisión RDS. Límite: 8 caracteres. Ejemplo: `-ps RASP-PI`.
* `-rt` especifica el radiotexto (RT) a transmitir. Límite: 64 caracteres. Ejemplo: `-rt 'Hello, world!'`.
* `-ctl` especifica una tubería con nombre (FIFO) para usar como canal de control para cambiar PS y RT en tiempo de ejecución (ver más abajo).
* `-ppm` especifica el error del oscilador de tu Raspberry Pi en partes por millón (ppm), ver más abajo.

Por defecto el PS cambia de un lado a otro entre `Pi-FmRds` y un número de secuencia, empezando en `00000000`. El PS cambia aproximadamente una vez por segundo.


### Calibrado del reloj (solo si tienes dificultades)

La norma RDS establece que el error para la subportadora de 57 kHz debe ser menor que ± 6 Hz, es decir, menos de 105 ppm (partes por millón). El oscilador de la Raspberry Pi puede presentar un error superior a esta cifra. Aquí es donde entra el parámetro `-ppm`: especificas el error de tu Pi y Pi-FM-RDS ajusta los divisores del reloj en consecuencia.

En la práctica, he observado que Pi-FM-RDS funciona correctamente incluso sin usar el parámetro `-ppm`. Supongo que los receptores son más tolerantes que lo indicado en la especificación RDS.

Una forma de medir el error en ppm es reproducir el archivo `pulses.wav`: reproducirá un pulso durante exactamente 1 segundo, luego 1 segundo de silencio, y así sucesivamente. Graba la salida de audio desde una radio con una buena tarjeta de sonido. Supongamos que muestras a 44.1 kHz. Mide 10 intervalos. Usando [Audacity](https://www.audacityteam.org/) por ejemplo, determina el número de muestras de esos 10 intervalos: en ausencia de error de reloj debería ser 441,000 muestras. Con mi Pi encontré 441,132 muestras. Por tanto, mi error en ppm es (441132-441000)/441000 * 1e6 = 299 ppm, asumiendo **que mi dispositivo de muestreo (tarjeta de sonido) no tenga error de reloj...**


### Canalizar audio hacia Pi-FM-RDS

Si usas el argumento `-audio -`, Pi-FM-RDS lee audio desde la entrada estándar. Esto permite canalizar la salida de un programa hacia Pi-FM-RDS. Por ejemplo, esto puede usarse para leer archivos MP3 usando Sox:

```
sox -t mp3 http://www.linuxvoice.com/episodes/lv_s02e01.mp3 -t wav -  | sudo ./pi_fm_rds -audio -
```

O para canalizar la entrada AUX de una tarjeta de sonido hacia Pi-FM-RDS:

```
sudo arecord -fS16_LE -r 44100 -Dplughw:1,0 -c 2 -  | sudo ./pi_fm_rds -audio -
```


### Cambiar PS, RT y TA en tiempo de ejecución

Puedes controlar PS, RT y TA (bandera de Anuncio de Tráfico — Traffic Announcement) en tiempo de ejecución usando una tubería con nombre (FIFO). Para ello ejecuta Pi-FM-RDS con el argumento `-ctl`.

Ejemplo:

```
mkfifo rds_ctl
sudo ./pi_fm_rds -ctl rds_ctl
```

En ese punto, Pi-FM-RDS espera hasta que otro programa abra la tubería con nombre en modo escritura (por ejemplo `cat >rds_ctl` en el ejemplo abajo) antes de empezar a transmitir.

Puedes usar la tubería para enviar “comandos” que cambien PS, RT y TA. Por ejemplo, en otra terminal:

```
cat >rds_ctl
PS MyText
RT A text to be sent as radiotext
TA ON
PS OtherTxt
TA OFF
...
```

> [!TIP]
> El programa que abre la tubería en modo escritura puede iniciarse después de Pi-FM-RDS (como en el ejemplo anterior) o antes (en cuyo caso Pi-FM-RDS no tiene que esperar al inicio).

Cada línea debe comenzar con `PS`, `RT` o `TA`, seguido de un carácter espacio y el valor deseado. Cualquier otra línea se ignora silenciosamente. `TA ON` activa la bandera de Anuncio de Tráfico, cualquier otro valor la desactiva.


### Caracteres no ASCII

Puedes usar el juego completo de caracteres soportados por el protocolo RDS. Pi-FM-RDS decodifica las cadenas de entrada según las variables de localización (locale) del sistema. A comienzos de 2024, Raspberry Pi OS usa por defecto UTF-8 y la variable `LANG` está establecida en `en_GB.UTF-8`. Con esta configuración, debería funcionar sin cambios.

Si no funciona, observa el primer mensaje que imprime Pi-FM-RDS. Debería ser algo sensato, por ejemplo:

```
Locale set to en_GB.UTF-8.
```

Si no coincide con tu configuración, o si la locale aparece como `(null)`, entonces tus variables de locale no están configuradas correctamente y Pi-FM-RDS no podrá trabajar con caracteres no ASCII.


### Compilar en distribuciones con distintos ABIs de coma flotante

El makefile usa `-mfloat-abi=hard`, que es apropiado para Raspberry Pi OS. Otras distribuciones pueden requerir valores distintos, concretamente `soft` o `softfp`.


## Advertencia y aviso

PiFmRds es un programa **experimental**, diseñado **solo para experimentación**. En modo alguno está pensado para convertirse en un *centro multimedia* personal ni en una herramienta para operar una *emisora de radio*, ni siquiera para emitir sonido a tu propio equipo de música.

En la mayoría de los países, transmitir ondas de radio sin una licencia estatal que cubra las modalidades de transmisión específicas (frecuencia, potencia, ancho de banda, etc.) es **ilegal**.

Por tanto, conecta siempre una línea de transmisión apantallada desde la Raspberry Pi directamente a un receptor de radio, de modo que **no** se emitan ondas de radio. Nunca uses una antena.

Incluso si eres un radioaficionado con licencia, usar PiFmRds para transmitir en frecuencias de radioaficionados sin ningún filtrado entre la Raspberry Pi y una antena es muy probablemente ilegal porque la portadora cuadrada contiene muchas armonías; por tanto, los requisitos de ancho de banda probablemente no se cumplen.

No me puedo responsabilizar por el uso indebido de tu propia Raspberry Pi. Cualquier experimento se realiza bajo tu propia responsabilidad.


## Pruebas

Pi-FM-RDS se probó con éxito con todos mis dispositivos compatibles con RDS, a saber:

* un despertador Sony ICF-C20RDS de 1995,
* un receptor portátil Sangean PR-D1 de 1998, y un ATS-305 de 1999,
* un teléfono móvil Samsung Galaxy S2 de 2011,
* un equipo HiFi Philips MBD7020 de 2012,
* un dongle USB Silicon Labs [USBFMRADIO-RD](http://www.silabs.com/products/mcu/Pages/USBFMRadioRD.aspx), que emplea un chip Si4701, y usando mi programa [RDS Surveyor](http://rds-surveyor.sourceforge.net/),
* un “PCear Fm Radio”, un clon chino del anterior, nuevamente usando RDS Surveyor.

La recepción funciona perfectamente con todos los dispositivos anteriores. RDS Surveyor no reporta errores de grupo.

![](doc/galaxy_s2.jpg)


### Uso de CPU

El uso de CPU es el siguiente:

* sin audio: 9%
* con audio mono: 33%
* con audio estéreo: 40%

El uso de CPU aumenta drásticamente al añadir audio porque el programa debe remuestrear la (posible) tasa de muestreo de entrada a 228 kHz, su frecuencia interna de operación. Para ello aplica un filtro FIR, lo cual es costoso.


## Diseño

El generador de datos RDS se encuentra en el archivo `rds.c`.

El generador RDS crea cíclicamente cuatro grupos 0A (para transmitir PS) y un grupo 2A (para transmitir RT). Además, cada minuto inserta un grupo 4A (para transmitir CT, hora de reloj). `get_rds_group` genera un grupo e usa `crc` para calcular la CRC.

Para obtener muestras de datos RDS, llama a `get_rds_samples`. Esta función llama a `get_rds_group`, codifica diferencialmente la señal y genera un símbolo biphase conformado. Los símbolos biphase sucesivos se solapan: las muestras se suman de modo que el resultado equivale a aplicar el filtro conformador (un filtro root-raised-cosine — RRC — especificado en la norma RDS) a una secuencia de pulsos codificados en Manchester.

El símbolo biphase conformado se genera una vez por todas mediante un programa Python llamado `generate_waveforms.py` que usa [Pydemod](https://github.com/ChristopheJacquet/Pydemod), uno de mis otros proyectos de radio software. Este programa Python genera un arreglo llamado `waveform_biphase` que resulta de aplicar el filtro RRC a un par de impulsos positivo-negativo. Nota: la salida de `generate_waveforms.py`, dos archivos llamados `waveforms.c` y `waveforms.h`, están incluidos en el repositorio Git, por lo que no necesitas ejecutar el script Python tú mismo para compilar Pi-FM-RDS.

Internamente, el programa muestrea todas las señales a 228 kHz, cuatro veces la subportadora RDS de 57 kHz.

La señal multiplex FM (señal base) se genera en `fm_mpx.c`. Este archivo gestiona el remuestreo del archivo de audio de entrada a 228 kHz y la generación del multiplex: la señal no modulada izquierda+derecha (limitada a 15 kHz), posiblemente el piloto estéreo a 19 kHz, posiblemente la señal izquierda-derecha modulada en amplitud sobre 38 kHz (portadora suprimida) y la señal RDS desde `rds.c`. El remuestreo se realiza usando un hold de orden cero seguido de un filtro FIR pasa-bajo de orden 60. El filtro es una sinc muestreada ventana por una ventana de Hamming. Los coeficientes del filtro se generan al inicio de la ejecución de forma que el filtro corte frecuencias por encima del mínimo entre:
* la frecuencia de Nyquist del archivo de audio de entrada (la mitad de la tasa de muestreo) para evitar aliasing,
* 15 kHz, la banda de paso de los canales left+right y left-right, según las normas de radiodifusión FM.

Las muestras son reproducidas por `pi_fm_rds.c` que está adaptado del [PiFmDma](https://github.com/richardghirst/PiBits/tree/master/PiFmDma) de Richard Hirst. El programa se modificó para soportar una tasa de muestreo de exactamente 228 kHz.


### Referencias

* EN 50067, Especificación del sistema de datos de radio (RDS) para radiodifusión sonora VHF/FM en el rango de frecuencias 87.5 a 108.0 MHz.


## Historial

* 2024-02-21: manejo correcto de caracteres no ASCII.
* 2015-09-05: soporte para Raspberry Pi 2 y modelos posteriores
* 2014-11-01: soporte para alternar la bandera TA (Anuncio de Tráfico) en tiempo de ejecución
* 2014-10-19: corrección de bug (detener limpiamente el motor DMA cuando el archivo especificado no existe, o no es posible leer desde stdin)
* 2014-08-04: corrección de bug (ppm ahora usa floats)
* 2014-06-22: generación de CT (hora del reloj), correcciones de bugs
* 2014-05-04: posibilidad de cambiar PS y RT en tiempo de ejecución
* 2014-04-28: soporte para canalizar datos de archivos de audio a la entrada estándar de Pi-FM-RDS
* 2014-04-14: nueva versión que soporta cualquier tasa de muestreo para la entrada de audio y que puede generar una señal FM-Stereo adecuada si se proporciona un archivo de entrada estereofónico
* 2014-04-06: lanzamiento inicial, que solo soportaba archivos de audio monofónicos de 228 kHz

--------

© [Christophe Jacquet](https://jacquet.xyz/en/) (F8FTK), 2014-2025. Publicado bajo la GNU GPL v3.
