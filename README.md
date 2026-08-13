# Telemetría SATCOM

# Configuración de Telemetría SATCOM para Drones y ESP32

Esta documentación detalla el proceso completo para establecer un enlace de telemetría redundante utilizando una placa ESP32, el controlador de vuelo Cube y una conexión satelital mediante Starlink.  

## **Etapa 1: Verificación de la Placa y Enlace Local**

Antes de instalar el firmware definitivo, es fundamental validar que el hardware de la placa ESP32 funciona correctamente y es capaz de conectarse a la red inalámbrica. Para ello, se compila un código de prueba básico que conecta el microcontrolador a una red Wi-Fi. 

### Código de Validación de Red

El siguiente script en C++ configura la ESP32 en modo Cliente y abre un socket de prueba. Recuerda reemplazar los valores de red por los tuyos: 

```cpp
/*
Telemetría SATCOM
Paso 1.1: Verificación de la placa y enlace local MAVLink-over-IP
Autor: Dylan Cobos
*/

#include <WiFi.h>

// Reemplaza con las credenciales de tu red Wi-Fi local de prueba
const char* ssid = "NOMBRE_DE_TU_RED";
const char* password = "CLAVE_DE_TU_RED";

void setup() {
    // Iniciamos el puerto serie a la misma velocidad estándar que usaremos con ArduPilot
    Serial.begin(115200);
    delay(1000);
    Serial.println();
    Serial.println("===========================");
    Serial.println(" INICIANDO PROTOCOLO DE RED ");
    Serial.println("===========================");
    
    Serial.print("[*] Intentando enlazar con la red: ");
    Serial.println(ssid);
    
    // Forzamos a la placa a trabajar solo como Cliente
    WiFi.mode(WIFI_STA);
    WiFi.begin(ssid, password);
    
    // Bucle de espera hasta obtener respuesta del router
    int intentos = 0;
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
        intentos++;
        if (intentos > 30) { 
            // Si pasa de 15 segundos sin conectar, avisamos del error
            Serial.println("\n[!] ERROR: No se pudo conectar. Revisa el SSID o la contraseña.");
            return;
        }
    }
    
    Serial.println("\n\n[+] ¡ENLACE ESTABLECIDO CON ÉXITO!");
    Serial.print(" -> Dirección IP local asignada: ");
    Serial.println(WiFi.localIP());
    Serial.print(" -> Potencia de recepción (RSSI): ");
    Serial.print(WiFi.RSSI());
    Serial.println(" dBm");
    Serial.println("===========================");
    Serial.println("[i] El microcontrolador está listo para el test de Ping.");
}

void loop() {
    // En esta etapa de verificación, el procesador solo mantiene el socket abierto
    delay(1000);
}
```

### **Resultados esperados**

Una vez establecida la conexión, se debe realizar un *ping* desde una computadora en la misma red hacia la IP asignada a la ESP32 para comprobar la integridad de la conectividad.

## Etapa 2: Conexión de Hardware (Cube - ESP32)

Confirmado el funcionamiento lógico de la placa, el siguiente paso es realizar el cableado para enlazar el puerto de telemetría (`TELEM 1` o `TELEM 2`) del controlador Cube con los puertos seriales y de alimentación de la ESP32.

La configuración de pines debe realizarse de la siguiente manera:

- **PIN 1 (VCC 5V)** del Cube se conecta al **PIN 5V ( VIN)** de la ESP32 (cable rojo).
- **PIN 2 (TX)** del Cube se conecta al **GPIO 16 (RX2)** de la ESP32 (cable verde).
- **PIN 3 (RX)** del Cube se conecta al **GPIO 17 (TX2)** de la ESP32 (cable azul).
- **PIN 6 (GND)** del Cube se conecta al **PIN GND** de la ESP32 (cable negro).

### ¿Por qué no se conectan los pines CTS y RTS?

En un puerto de telemetría estándar de 6 pines, los conectores número 4 y 5 corresponden a las líneas de control de flujo por hardware CTS y RTS. En esta implementación, se ha decidido omitir su conexión por las siguientes razones:

- MAVLink es un protocolo de comunicación altamente eficiente que, a las velocidades de transmisión utilizadas funciona de manera robusta de forma asíncrona utilizando únicamente las líneas de transmisión (TX) y recepción (RX). No requiere obligatoriamente una validación física de listo para enviar/recibir
- Como se verá en etapas posteriores, el firmware de DroneBridge instalado en la ESP32 se configura intencionalmente con los parámetros `UART RTS GPIO` y `UART CTS GPIO` en `0`. Esto le indica al microcontrolador que desactive la espera de señales físicas de control de flujo

## Etapa 3: Preparación y Flasheo del Firmware

En esta etapa comenzaremos con la implementación de bibliotecas de código abierto para garantizar un control redundante y profesional al momento de operar el cuadricóptero. Para ello, instalaremos el firmware de DroneBridge en la placa ESP32.

### Descarga de Herramientas y Firmware

Para comenzar, es necesario descargar los recursos de software:

#### Herramienta de Flasheo (Espressif):

Descarga la herramienta Flash Download Tools desde el sitio web oficial de Espressif.

https://www.espressif.com/en/support/download/other-tools

Descomprime la carpeta y ejecuta el archivo `flash_download_tool_xxxx.exe` 

En la ventana inicial que aparecerá, configura los parámetros de la siguiente manera:

- **ChipType:** `ESP32`
- **WorkMode:** `Develop`
- **LoadMode:** `UART`

Haz Clic en OK

#### Firmware DroneBridge:

Descarga el firmware desde los releases en su repositorio Oficial.

**Nota de estabilidad:** Se recomienda descargar la versión **v2.1.0 stable** (ubicada en la sección *Assets* como un archivo `.zip`) para garantizar el mejor rendimiento del proyecto.

https://github.com/DroneBridge/ESP32/releases

Al descomprimir el archivo, notarás que existen carpetas para diferentes versiones de placas. Para este proyecto, abre la carpeta llamada `esp32`

Dentro de este directorio, encontrarás los archivos binarios (`.bin`) requeridos y un archivo de texto con las direcciones de memoria necesarias para el flasheo.

#### Configuración del Entorno de Flasheo

Dentro de la interfaz principal del Flash Download Tool, debes seleccionar los archivos binarios y asignarles su dirección correspondiente. Configúralos en el siguiente orden:

| Fila | Archivo Binario | Dirección de Memoria |
| --- | --- | --- |
| 1 | bootloader.bin | 0x1000 |
| 2 | partition-table.bin | 0x8000 |
| 3 | db_esp32.bin | 0x10000 |
| 4 | www.bin | 0x190000 |

Marca las 4 casillas pequeñas en el extremo izquierdo de cada fila. Deben ponerse de color verde; de lo contrario, el programa ignorará los archivos al momento de grabar. 

En la sección inferior del programa (`SPIFlashConfig`), ajusta la configuración del puerto y la placa:

- **SPI SPEED:** `40MHz`
- **SPI MODE:** `DIO`
- **FLASH SIZE:** `32Mbit`
- **COM:** Selecciona el puerto COM correspondiente a tu ESP32.
- **BAUD:** `115200`

#### El Flasheo Definitivo

Con los parámetros configurados, procede a grabar la placa:

1. Mantén presionado el botón **BOOT** (o **IO0**) de tu placa y no lo sueltes.
2. Haz clic en el botón **START** ubicado en la esquina inferior izquierda del programa.
3. En cuanto veas la palabra **"SYNC"** en el panel o que la barra verde de progreso comience a avanzar, puedes soltar el botón de la placa.
4. Espera a que la barra llegue al 100% y el cuadro verde de arriba a la derecha indique **FINISH**.
5. Una vez finalizado, desconecta el cable USB y vuélvelo a conectar para reiniciar el microcontrolador con su nuevo firmware.

#### **Verificación**

Si el proceso fue exitoso, al buscar en las redes Wi-Fi de tu computadora debería aparecer una nueva red llamada **DroneBridge ESP32**.

## Etapa 4: Transición a Modo Cliente en DroneBridge

Dentro de la interfaz web de DroneBridge, cambiaremos el rol de la placa para que actúe como un dispositivo más dentro de tu red satelital:

1. Ve a la sección de configuración de Wi-Fi. 
2. Cambia el modo de operación de **Access Point** a **Wi-Fi Client.**
3. Ingresa las credenciales de tu red
    - **SSID:** `NOMBRE_DE_TU_RED_SATELITAL`
    - **Password:** `CLAVE_DE_TU_RED_SATELITAL`
4. En la sección *Serial*, ajusta los pines que configuramos en la Etapa 2
    - **UART TX GPIO:** Cambia el valor por `17`
    - **UART RX GPIO:** Cambia el valor por `16`
5. Guarda los cambios y reinicia el dispositivo.

A partir de este momento, el hardware dejará de emitir su propia red y se enlazará automáticamente al Wi-Fi del dron cada vez que reciba energía.

## Etapa 5: Creación del Servidor VPS en la Nube

Para enrutar la telemetría desde cualquier parte del mundo hacia la estación de control, utilizaremos una máquina virtual. Microsoft ofrece el programa **Azure for Students**, el cual es ideal porque no requiere ingresar ninguna tarjeta de crédito para arrancar.

### Despliegue de la Máquina Virtual

1. Entra a la página oficial de Azure for Students y haz clic en **Start free**
2. Inicia sesión directamente con tu correo institucional de la EPN y tu contraseña habitual. El sistema validará automáticamente el dominio académico, te asignará el panel de control y depositará los créditos.
3. Dirígete a **Máquinas virtuales** > **Crear** > **Máquina virtual**
4. En la pestaña **Basics (Datos básicos)**, llena los campos de la siguiente manera:
    - **Subscription:** `Azure for Students`
    - **Resource group:** Haz clic en "Create new" y asígnale un nombre.
    - **Virtual machine name:** `Servidor-Telemetria`
    - **Region:** `north central us`
    - **Opciones de disponibilidad:** Sin redundancia de la infraestructura necesaria
    - **Tipo de seguridad:** Máquinas virtuales de inicio seguro
    - **Image:** `Ubuntu Server 24.04 LTS - x64 Gen2`
    - **VM architecture:** `x64`
    - **Tamaño:** `Standard_B2ats_v2`
    - **Nombre de usuario:** Define el usuario con el que accederás a la terminal del servidor.
5. Deja el resto en predeterminado, haz clic en Revisar y crear, y descarga las claves de acceso (.pem). Es importante descargar estas claves, ya que sin ellas no tendrás acceso a la máquina virtual.

### Apertura de Puertos (Reglas ACL)

Por seguridad, Azure bloquea todo el tráfico entrante a excepción del puerto administrativo. Debemos habilitar el canal para el dron. En el menú lateral izquierdo de tu máquina virtual dentro del portal de Azure, ve a **Configuración de red** > **Crear ACL del puerto** > **Agregar regla de puerto de entrada**:

#### Regla 1: Puerto del Dron

- **Intervalos de puertos de destino:** `14550` (es el puerto estándar por donde viajan los datos MAVLink).
- **Protocolo:** `UDP` si lo dejas en TCP o Cualquiera, la telemetría se va a atascar.
- **Acción:** `Permitir`.
- **Nombre:** Ponle algo identificable como `Telemetria_Dron`.

#### Regla 2: Puerto de la Estación en Tierra

- **Intervalos de puertos de destino:** `5760`.
- **Protocolo:** `TCP`
- **Acción:** `Permitir`.
- **Prioridad:** Puedes dejar el número que Azure te sugiera automáticamente seguramente será `320`.
- **Nombre:** `QGC_Conexion`.

Haz clic en **Agregar** y espera unos segundos a que Azure confirme que la regla se guardó.

## **Etapa 6: Configuración Interna del Servidor (MAVLink-Router)**

### Acceso mediante SSH

Abre el Símbolo del sistema en tu computadora y usa el comando `cd` para moverte a la carpeta donde guardaste tu archivo `.pem`.

Si estás en Windows, ajusta los permisos de la llave ejecutando estos comandos:

```bash
icacls Servidor-Telemetria_key.pem /inheritance:r
icacls Servidor-Telemetria_key.pem /grant:r "%username%":"(R)"
```

Una vez en la carpeta correcta, ejecuta este comando usando tu IP:

```bash
ssh -i "nombre_del_archivo.pem" usuario@ip_publica_del_servidor
```

### Preparación y Compilación del Sistema

Actualizamos el sistema base: 

```bash
sudo apt update && sudo apt upgrade -y
```

Instalamos las herramientas de compilación:

```bash
sudo apt install git meson ninja-build pkg-config gcc g++ systemd -y
```

Para poder compilar correctamente se hizo un *swap* de RAM para tener 2GB en nuestra máquina virtual debido a que viene con solo 1GB. Copia y pega cada línea por separado:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Descargamos el código del enrutador y compilamos:

```bash
git clone https://github.com/mavlink-router/mavlink-router.git
cd mavlink-router
git submodule update --init --recursive
meson setup build
ninja -C build
sudo ninja -C build install
```

### Creación de la Configuración de Red

Generamos la carpeta para el enrutador y abrimos el archivo de configuración:

```bash
sudo mkdir -p /etc/mavlink-router
sudo nano /etc/mavlink-router/main.conf
```

Se abrirá el editor en tu terminal. Pega exactamente este bloque de texto ahí dentro:

```bash
[General]
TcpServerPort=5760
ReportStats=false
MavlinkDialect=common
Log=/var/log/mavlink-router

[UdpEndpoint UDP_In]
Mode=server
Address=0.0.0.0
Port=14550
```

Para guardar los cambios en ese editor:

Presiona `Ctrl + O` y luego `Enter`.

Presiona `Ctrl + X`.

### Encender el Puente de Telemetría

Ejecuta los siguientes comandos para iniciar el servicio y asegurarte de que arranque automáticamente:

```bash
sudo systemctl enable mavlink-router.service
sudo systemctl start mavlink-router.service
sudo systemctl status mavlink-router.service
```