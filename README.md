<a id="Presentación"></a>
# Proyecto-Monitoreo-Consumo-Energetico
Este proyecto utiliza una ESP32, una Raspberry Pi, entre otros componentes para su funcionamiento. Con el podrás medir la corriente y potencia de uno o más dispositivos conectados a un multi contacto.

<a id="Índice"></a>
## Índice

1. [Inicio del proyecto](#Presentación)

1. [Índice](#Índice)

1. [Materiales a utilizar](#Materiales)

1. [Requisitos para empezar](#Requisitos)

1. [Creacion de la base de datos](#Base_de_datos)

1. [Circuito](#Circuito)

1. [Código ESP32](#Codigo)

1. [Código Node-Red](#Node)

1. [Configuración Node-red](#ConfNode)

1. [Funcionamiento del proyecto](#Funcionamiento)

<a id="Materiales"></a>
## Materiales a utilizar                                    

* Raspberry Pi 4

* ESP32-WROOM-32

* Cable de datos tipo USB-A a USB-C

* 1 SCT013 30A/1V

* 1 Modulo Breakout TRRS Jack 3.5mm

* 1 Capacitor de 100μF a 25 o 35V 

* 2 Resistencias de 470kΩ

* 1 Resistencia de 330Ω

* 1 LED (cualquier color)

* Cables Jumper macho-hembra

* 1 o 2 Protoboard

* Relevador SRD-05VDC-SL-C

* 1 Transistor 2N2222A

* Un multi contacto

<a id="Requisitos"></a>
## Requisitos para empezar

* Saber la IP de nuestro Raspberry Pi en el formato *0.0.0.0*
 
* Tener previamente instalado la **IDE de Arduino**, **Node-red** y **Mysql** (*mariadb-server*) en tu Raspberry Pi.

* Se necesita de un usuario con todos los privilegios en **Mysql**.

* Tener conocimientos básicos de cómo programar en **C++** ya que este es el lenguaje utilizado por la **IDE de Arduino**.

[Regresar al indice](#Índice)

<a id="Base_de_datos"></a>
## Creacion de la base de datos

Para la creación de nuestra base de datos ocuparemos **Mysql** (*mariadb-server*), con el, almacenaremos las lecturas de corriente y potencia que lea nuestro **SCT013** en nuestra Raspberry Pi.

Para ello necesitaremos abrir una terminal en la **Raspberry Pi**, en ella iniciar **Mysql** con nuestro usuario y ejecutar los siguientes comandos:

```mysql
CREATE DATABASE consumo;
```

Y despues lo ocuparemos para crear nuestra tabla usando este comando:

```mysql
USE consumo;
```

Una vez destro de nuestra **DATABASE** crearemos nuestra tabla con el siguiente comando:

```mysql
CREATE TABLE ConsumoEnergetico ( id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY, fecha TIMESTAMP
DEFAULT CURRENT_TIMESTAMP, nombre CHAR(248) NOT NULL, irms FLOAT(4,2), pot FLOAT(6,3) );
```
Ya creada podemos usar estos comandos para verificar que funciona:
```mysql
INSERT INTO ConsumoEnergetico (nombre, irms, pot) VALUES ('usuario', 1.37, 0.065);

SELECT * FROM ConsumoEnergetico;
```

Y con esto tendremos nuestra base de datos ya creada.

[Regresar al indice](#Índice)

<a id="Circuito"></a>
## Circuito

Debemos tener en cuenta lo siguiente a la hora de armar nuestro circuito.

En el siguiente recuadro se muestra una tabla de que pines de la ESP32 van a que partes del circuito:

```
Pines ESP32----------Circuito
3v3------------------pata libre de resistencia 2 470kΩ
1GND-----------------negativo del capacitor
2GND-----------------pin emisor del transistor
3GND-----------------resistencia 330Ω
pin34----------------pin TIP del TRRS
pin33----------------LED wifi
```



<a id="Codigo"></a>
## Código ESP32

Para que funcione el codigo primero debemos acceder al **IDE de ARDUINO** en **File** y luego **Preferences** y agregar el siguiente link y dar ok: https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json

Ahora pasamos a **Tools** y **Board:**, seleccionamos **ESP32 Arduino** y buscamos **ESP32 Dev Module**, dentro de **Tools** pasamos a **Port** para verificar que nuestra **ESP32** esta conectada y lista para recibir codigo de la **Raspberry Pi**.

Despues nos vamos a **Tools** y **Manage Libraries** y buscamos **EmonLib** *by  OpenEnergyMonitor*, **PubSubClient** *by Nick O'Leary*, **Adafuit Unified Sensor** *by Adafruit* y losd instalamos. Una vez instalados podemos comenzar a cargar el siguiente codigo en nuestra ESP32.

```c++

/*

Envia corriente y potencia del SCT013 en JSON 
por MQTT con el ESP32-WROOM32 con configuracion no bloqueante

Creado por: Roberto Giordano Gonzalez Valencia
Fecha: 26 de julio de 2024

Este codigo esta basado en otro codigo que se conecta al DHT11 el cual fue
creado por Hugo Escalpelo el 6 de julio de 2023 el cual fue modificado para
este proyecto el cual mide corriente y potencia con el sensor SCT013.
El siguiente codigo es una modificacion del antes mencionado. 

*/

// Bibliotecas
#include <WiFi.h> // Biblioteca WiFi
#include <PubSubClient.h> // Biblioteca MQTT
#include "EmonLib.h" // Biblioteca para el sensor SCT013

// Datos para el internet y el mqtt
const char* ssid = "******";
const char* password = "******";
const char* mqtt_server = "***.***.*.**";

//esta variable es el voltaje que proporciona CFE en Mexico, este puede variar dependiendo de cada casa y se puede modificar
float voltajeRed = 127.0;

// Variables para envio de mensajes
unsigned long lastMsg = 0; // Contador de tiempo mensajes
#define MSG_BUFFER_SIZE  (50) // Buffer para enviï¿½o de mensajes
char msg[MSG_BUFFER_SIZE]; // Variable para conversion de mensaje

// Declaracion del LED e indicador
int statusLedPin = 33;  // led wifi Esta variable controla el led para saber si se conecto a internet la esp32
int RelevadorPin = 4; // Esta variable controla el relevador
bool statusLed = 0;// Bandera que me dice si el led esta encendido o apagado

// Variables para el manejo de tiempo no bloqueante
double timeLast, timeNow; // Variables para el control de tiempo no bloqueante
long lastReconnectAttempt = 0; // Variable para el conteo de tiempo entre intentos de reconexion
double WAIT_MSG = 2500;  // Espera de 2.5 segundos entre mensajes
double RECONNECT_WAIT = 5000; // Espera de 5 segundos entre conexiones

// Temas MQTT
const char* mqttMsg = "codigoIoT/esp32/msg";
const char* mqttCallback = "codigoIoT/esp32/callback";
const char* mqttDHT = "codigoIoT/esp32/SCT013/ConsumoEnergetico";

// iniciadores de librerias WiFi, MQTT y sensor SCT013
WiFiClient espClient; // iniciador para conexion a WiFi
PubSubClient client(espClient); // iniciador para conexion a MQTT
EnergyMonitor emon1; // Objeto que inicia el sensor SCT013

// Funcion para conectarse a Internet
void setup_wifi() {
  delay(10);
  
  // Mensajes de intento de conexion
  Serial.println();
  Serial.print("Conectando a: ");
  Serial.println(ssid);

  // Funciones de conexion
  WiFi.mode(WIFI_STA); // STA inicia el micro controlador como cliente, para accespoint se usa WIFI_AP
  WiFi.begin(ssid, password); // llama los valores ya declarados de red

  // Secuencia que hace parpadear el LED durante el intento de conexion. Logica Inversa
  while (WiFi.status() != WL_CONNECTED) {
    digitalWrite (statusLedPin, LOW); // Apagar LED
    delay(500); // Dado que es de suma importancia esperar a la conexion, debe usarse espera en vez de bloqueante
    digitalWrite (statusLedPin, HIGH); // Encender LED
    Serial.print(".");  // Indicador de progreso
    delay (5); // Espera asimetrica para dar un parpadeo al led
  }

  // Esta funcion mejora los resultados de las funciones aleatorias y toma como semilla el contador de tiempo
  randomSeed(micros());

  // Mensajes de confirmacion
  Serial.println("");
  Serial.println("Conectado a WiFi");
  Serial.println("Direccion IP: ");
  Serial.println(WiFi.localIP());
}

// Funcion Callback, Se ejecuta al recivir un mensaje
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Mensaje recibido en [");
  Serial.print(topic);
  Serial.print("] ");

  // Esta secuencia imprime y construye el mensaje recibido
  String messageTemp;
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]); // Imprime
    messageTemp += (char)payload[i]; // Construye el mensaje en una variable String
  }
  Serial.println();

  // Se comprueba que el mensaje se haya concatenado correctamente
  Serial.println();
  Serial.print ("Mensaje concatenado en una sola variable: ");
  Serial.println (messageTemp);

  //activa o desactiva el Relevador al recibir un true o false
  if(messageTemp == "true") {
    Serial.println("Led encendido");
    digitalWrite(RelevadorPin, HIGH);
  }
  else if(messageTemp == "false") {
    Serial.println("Led apagado");
    digitalWrite(RelevadorPin, LOW);
  }
}

// Funcion de reconexion. Devuelve un valor booleano para indicar exito o falla
boolean reconnect() {
  Serial.print("Intentando conexion MQTT...");
  // Generar un ID aleatorio
  String clientId = "ESP32Client-";
  clientId += String(random(0xffff), HEX);
  // Intentar conexion
  if (client.connect(clientId.c_str())) {
    // Una vez conectado, da una retroalimentacion
    client.publish(mqttMsg,"Conectado");
    // Funcion de suscripcion
    client.subscribe(mqttCallback);
  } else {
      // si da error
      Serial.print("Error rc=");
      Serial.print(client.state());
      Serial.println(" Intentando de nuevo");
      // Espera 1 segundo
      delay(1000);
    }
  return client.connected();
}

// Inicio del codigo
void setup() {

  // Inician las comunicaciones
  Serial.begin(115200); //baudios en los que trabaja la ESP32
  emon1.current(34, 7.5);//Aqui se declara el pin de entrada al cual le llegara la seï¿½al leida por el SCT013 y la constante de calibracion para la ESP32
  //(pin de entrada, constante)

  // Configuracion de pines
  pinMode (statusLedPin, OUTPUT);// Se configura el pin como salida 
  pinMode (RelevadorPin, OUTPUT);// Se configura el pin como salida
  digitalWrite (statusLedPin, LOW);// Se comienza con el LED apagado
  digitalWrite (RelevadorPin, LOW);// Se comienza con el Relevador apagado

  // Iniciar conexiones  
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);

 

  // Iniciar secuencias de tiempo
  timeLast = millis (); // Inicia el control de tiempo
  lastReconnectAttempt = timeLast; // Control de secuencias de tiempo
}

// Cuerpo del codigo
void loop() {
  
  double Irms = emon1.calcIrms(22500); //Aqui calcula la corriente en RMS y el valor dentro del "()" es la cantidad de muestras que se toman para hacer el calculo

  float potencia = Irms * voltajeRed; //Calculamos la potencia a partir de P=I*V dando el valor en W

  // Funcion principal de seguimiento de tiempo
  timeNow = millis();

  // Funcion de conexion y seguimiento
  if (!client.connected()) { // Se pregunta si no existe conexion
    if (timeNow - lastReconnectAttempt > RECONNECT_WAIT) { // Espera de reconexion
      // Intentar reconectar
      if (reconnect()) {
        lastReconnectAttempt = timeNow; // Actualizacion de seguimiento de tiempo
      }
    }
  } else {
    // Funcion de seguimiento
    client.loop();
  }
  
  // Enviar un mensaje cada WAIT_MSG
  if (timeNow - timeLast > WAIT_MSG && client.connected() == 1) { // Manda un mensaje por MQTT cada cinco segundos
    timeLast = timeNow; // Actualizacion de seguimiento de tiempo    

    //Se construye el string correspondiente al JSON que contiene 3 variables
    String json = "{\"id\":\"usuario\",\"irms\":"+String(Irms)+",\"pot\":"+String(potencia)+"}";
    Serial.println(json); // Se imprime en monitor solo para poder visualizar que el string esta correctamente creado
    int str_len = json.length() + 1;//Se calcula la longitud del string
    char char_array[str_len];//Se crea un arreglo de caracteres de dicha longitud
    json.toCharArray(char_array, str_len);//Se convierte el string a char array    
    client.publish(mqttDHT, char_array); // Esta es la funcion que envia los datos por MQTT, especifica el tema y el valor
  }// fin del if (timeNow - timeLast > wait)
}

```

[Regresar al indice](#Índice)

<a id="Telegram"></a>
## Bot Telegram

Para inciar debemos tener instalado previamente **Telegram**.

Abrimos **Telegram** y buscamos el la lupa **BotFather** y lo inciamos con el boton **Iniciar** o introduciendo lo siguiente : **/start**

Una vez iniciado intriducimos **/newbot** para crear nuestro bot al que le mandaremos instrucciones para controlar los pines de la **ESP32**.

Despues de mandar **/newbot** nos preguntara cual será el nombre de nuestro bot al cual le podremos poner cualquier nombre como el siguiente: **Bot Consumo Energetico**. Una vez mandado nos pedira otro nombre para el bot el cual usaremos en **Node-red** el cual puede ser el siguiente : **consumo_energetico_bot**.   
- **NOTA: Es importante que lleve "_" si el nombre lo quieres poner espaciado y que al final termine con *bot***.

Una vez que mandemos el nombre nos dara un mensaje en el cual nos dara en una parte algo parecido a **Use this token to access the HTTP API:** y ese sera el token que utilizaremos mas adelante en **Node-red**.

Ahora para que le puedan llegar los mensajes, ahi mismo en el **BotFather** mandamos un **/setprivacy** y elegimos nuestro bot. Despues de eso mandamos un **Disable** y con eso ya esta configurado el bot.

Despues salimos del **BotFather** y ahora buscamos **IDBot**, dentro de el mandamos **/getid** el cual sera nuestro chatId que nos servira para el **Node-red**.

[Regresar al indice](#Índice)

<a id="Node"></a>
## Código Node-Red

Primero que nada debemos de instalar lo siguiente. En **Node-red** nos dirijimos al menu de hamburgesa (tres lineas horizontales) y seleccionamos **Manage palette** en **install** y buscamos: **node-red-dashboard**, **node-red-node-mysql** y **node-red-contrib-telegrambot** y los instalamos.

Y con esto ya podremos importar el siguiente codigo de **Node-red** en formato **JSON**. Para ello accedemos al menu de Hamburgesa, **import** y pegas el siguiente codigo:

```javascrip
[
    {
        "id": "655c2b33c17c7d19",
        "type": "tab",
        "label": "Consumo Energetico",
        "disabled": false,
        "info": "",
        "env": []
    },
    {
        "id": "548780248dd8ced5",
        "type": "mqtt in",
        "z": "655c2b33c17c7d19",
        "name": "",
        "topic": "codigoIoT/esp32/SCT013/ConsumoEnergetico",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "cad3536bf5f60c36",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 210,
        "y": 40,
        "wires": [
            [
                "bb0d3c9816f5cb03"
            ]
        ]
    },
    {
        "id": "21175a601cfc7aca",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "Extraer Corriente",
        "func": "msg.topic=msg.payload.ID;\nmsg.payload=msg.payload.irms;\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 690,
        "y": 60,
        "wires": [
            [
                "f97434cc75aead59",
                "2197f02b9ddb7d28"
            ]
        ]
    },
    {
        "id": "5e5bd47e69ca5a3b",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "Extraer Potencia kW",
        "func": "msg.topic=msg.payload.ID;\nmsg.payload=(msg.payload.pot *0.001).toFixed(3);\nreturn msg;\n",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 720,
        "y": 160,
        "wires": [
            [
                "ea7b94b2520b530c",
                "4d3c2466af07428d"
            ]
        ]
    },
    {
        "id": "2197f02b9ddb7d28",
        "type": "ui_gauge",
        "z": "655c2b33c17c7d19",
        "name": "",
        "group": "08d40e027fb0f7f9",
        "order": 0,
        "width": 0,
        "height": 0,
        "gtype": "gage",
        "title": "Corriente. Actual",
        "label": "A",
        "format": "{{value}}",
        "min": 0,
        "max": "10",
        "colors": [
            "#20cdff",
            "#2ee600",
            "#ff0202"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 1050,
        "y": 80,
        "wires": []
    },
    {
        "id": "4d3c2466af07428d",
        "type": "ui_gauge",
        "z": "655c2b33c17c7d19",
        "name": "",
        "group": "f323906c191efaa9",
        "order": 0,
        "width": "0",
        "height": "0",
        "gtype": "donut",
        "title": "Potencia. Actual (kW)",
        "label": "kW",
        "format": "{{value}}",
        "min": 0,
        "max": "1",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 1080,
        "y": 180,
        "wires": []
    },
    {
        "id": "f97434cc75aead59",
        "type": "ui_chart",
        "z": "655c2b33c17c7d19",
        "name": "",
        "group": "b78994697d2872cb",
        "order": 0,
        "width": 0,
        "height": 0,
        "label": "Corriente en el tiempo (A)",
        "chartType": "line",
        "legend": "true",
        "xformat": "HH:mm:ss",
        "interpolate": "linear",
        "nodata": "Esperando datos",
        "dot": false,
        "ymin": "0",
        "ymax": "10",
        "removeOlder": "4",
        "removeOlderPoints": "",
        "removeOlderUnit": "3600",
        "cutout": 0,
        "useOneColor": false,
        "useUTC": false,
        "colors": [
            "#ff790b",
            "#31d0ff",
            "#006fbf",
            "#2ca02c",
            "#98df8a",
            "#d62728",
            "#ff9896",
            "#9467bd",
            "#c5b0d5"
        ],
        "outputs": 1,
        "useDifferentColor": false,
        "className": "",
        "x": 1050,
        "y": 40,
        "wires": [
            []
        ]
    },
    {
        "id": "37a1fe43fd83c5e3",
        "type": "mysql",
        "z": "655c2b33c17c7d19",
        "mydb": "",
        "name": "",
        "x": 1070,
        "y": 260,
        "wires": [
            []
        ]
    },
    {
        "id": "e8e029d70e3711a9",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "Generar consulta",
        "func": "//INSERT INTO ConsumoEnergetico (nombre, temperatura, humedad) VALUES (x, y, z);\n//Generar inicio del query\nmsg.topic = \"INSERT INTO ConsumoEnergetico (nombre, irms, pot) VALUES (\";\n//Concatenar identidad del usuario\nmsg.topic += \"'\" + msg.payload.ID + \"'\" + \", \";\n//Concatenar temperatura\nmsg.topic += msg.payload.irms + \", \";\n//Concatenar humedad\nmsg.topic += msg.payload.pot + \");\";\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 710,
        "y": 260,
        "wires": [
            [
                "37a1fe43fd83c5e3"
            ]
        ]
    },
    {
        "id": "039bc8f82abbc9b7",
        "type": "inject",
        "z": "655c2b33c17c7d19",
        "name": "",
        "props": [
            {
                "p": "topic",
                "vt": "str"
            }
        ],
        "repeat": "",
        "crontab": "",
        "once": false,
        "onceDelay": 0.1,
        "topic": "SELECT * FROM ConsumoEnergetico;",
        "x": 190,
        "y": 320,
        "wires": [
            [
                "2950ac1edd35506e"
            ]
        ]
    },
    {
        "id": "2950ac1edd35506e",
        "type": "mysql",
        "z": "655c2b33c17c7d19",
        "mydb": "",
        "name": "",
        "x": 630,
        "y": 320,
        "wires": [
            [
                "489b9f92b113b0a0"
            ]
        ]
    },
    {
        "id": "489b9f92b113b0a0",
        "type": "debug",
        "z": "655c2b33c17c7d19",
        "name": "resultado",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "",
        "statusType": "auto",
        "x": 1080,
        "y": 320,
        "wires": []
    },
    {
        "id": "ea7b94b2520b530c",
        "type": "ui_chart",
        "z": "655c2b33c17c7d19",
        "name": "",
        "group": "b78994697d2872cb",
        "order": 0,
        "width": 0,
        "height": 0,
        "label": "Potencia en el tiempo (kW)",
        "chartType": "line",
        "legend": "true",
        "xformat": "HH:mm:ss",
        "interpolate": "linear",
        "nodata": "Esperando datos",
        "dot": false,
        "ymin": "0",
        "ymax": "1",
        "removeOlder": "4",
        "removeOlderPoints": "",
        "removeOlderUnit": "3600",
        "cutout": 0,
        "useOneColor": false,
        "useUTC": false,
        "colors": [
            "#ff790b",
            "#31d0ff",
            "#006fbf",
            "#2ca02c",
            "#98df8a",
            "#d62728",
            "#ff9896",
            "#9467bd",
            "#c5b0d5"
        ],
        "outputs": 1,
        "useDifferentColor": false,
        "className": "",
        "x": 1060,
        "y": 140,
        "wires": [
            []
        ]
    },
    {
        "id": "bb0d3c9816f5cb03",
        "type": "json",
        "z": "655c2b33c17c7d19",
        "name": "",
        "property": "payload",
        "action": "obj",
        "pretty": false,
        "x": 450,
        "y": 160,
        "wires": [
            [
                "21175a601cfc7aca",
                "5e5bd47e69ca5a3b",
                "e8e029d70e3711a9",
                "1337acdee2b1a6b4"
            ]
        ]
    },
    {
        "id": "b71febdbc5dec7c8",
        "type": "mqtt out",
        "z": "655c2b33c17c7d19",
        "name": "",
        "topic": "codigoIoT/esp32/callback",
        "qos": "2",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "cad3536bf5f60c36",
        "x": 1030,
        "y": 400,
        "wires": []
    },
    {
        "id": "c8d0c95a258854ad",
        "type": "ui_switch",
        "z": "655c2b33c17c7d19",
        "name": "",
        "label": "Dispositivo",
        "tooltip": "",
        "group": "6ef270b50846b685",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "true",
        "onvalueType": "bool",
        "onicon": "",
        "oncolor": "",
        "offvalue": "false",
        "offvalueType": "bool",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 590,
        "y": 400,
        "wires": [
            [
                "ab1e085a061523b5",
                "b71febdbc5dec7c8"
            ]
        ]
    },
    {
        "id": "3408ee7655657425",
        "type": "telegram receiver",
        "z": "655c2b33c17c7d19",
        "name": "",
        "bot": "",
        "saveDataDir": "",
        "filterCommands": false,
        "x": 130,
        "y": 400,
        "wires": [
            [
                "5596149ef4749e34"
            ],
            [
                "5596149ef4749e34"
            ]
        ]
    },
    {
        "id": "5596149ef4749e34",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "MansajeInterpretado",
        "func": "if (msg.payload.content === \"Apagar el ventilador\"){\n    msg.payload = true;\n} else if (msg.payload.content === \"Prender el ventilador\"){\n    msg.payload = false;          \n}\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 380,
        "y": 400,
        "wires": [
            [
                "c8d0c95a258854ad"
            ]
        ]
    },
    {
        "id": "707bf51a66aa617d",
        "type": "telegram sender",
        "z": "655c2b33c17c7d19",
        "name": "",
        "bot": "",
        "haserroroutput": false,
        "outputs": 1,
        "x": 1050,
        "y": 540,
        "wires": [
            []
        ]
    },
    {
        "id": "ab1e085a061523b5",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "MandaMensaje1",
        "func": "if (msg.payload === true ){\n    msg.payload = {\n        \"chatId\": ingresa tu chatid en numero,\n        \"type\": \"message\",\n        \"content\": \"El ventilador se apago con exito!\\n -Manda \\\"Prender el ventilador\\\" para conectarlo nuevamente\"\n    }\n} else if (msg.payload === false){\n    msg.payload = {\n        \"chatId\": ingresa tu chatid en numero,\n        \"type\": \"message\",\n        \"content\": \"El ventilador se conecto nuevamente con exito!\\n -Manda \\\"Apagar el ventilador\\\" para apagarlo nuevamente\"\n    }\n}\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 18,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 770,
        "y": 480,
        "wires": [
            [
                "707bf51a66aa617d"
            ]
        ]
    },
    {
        "id": "e878f1e80d1c113a",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "MandaMensaje2",
        "func": "msg.payload = {\n    \"chatId\": ingresa tu chatid en numero,\n    \"type\": \"message\",\n    \"content\": \"Ya paso una hora de que se conecto el dispositivo.\"\n}\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 9,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 770,
        "y": 540,
        "wires": [
            [
                "707bf51a66aa617d",
                "1e31ecbf976948ce"
            ]
        ]
    },
    {
        "id": "1e31ecbf976948ce",
        "type": "delay",
        "z": "655c2b33c17c7d19",
        "name": "",
        "pauseType": "delay",
        "timeout": "5",
        "timeoutUnits": "seconds",
        "rate": "1",
        "nbRateUnits": "1",
        "rateUnits": "second",
        "randomFirst": "1",
        "randomLast": "5",
        "randomUnits": "seconds",
        "drop": false,
        "allowrate": false,
        "outputs": 1,
        "x": 580,
        "y": 600,
        "wires": [
            [
                "8d1078f1cb562c0e"
            ]
        ]
    },
    {
        "id": "8d1078f1cb562c0e",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "MandaMensaje3",
        "func": "msg.payload = {\n    \"chatId\": ingresa tu chatid en numero,\n    \"type\": \"message\",\n    \"content\": \"¿Qué desea hacer? \\n-Apagar el ventilador\\n-Dejar que siga recopilando datos (para ello no mande nada)\"\n}\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 9,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 770,
        "y": 600,
        "wires": [
            [
                "707bf51a66aa617d",
                "1ca5c2076986a80b"
            ]
        ]
    },
    {
        "id": "1ca5c2076986a80b",
        "type": "delay",
        "z": "655c2b33c17c7d19",
        "name": "",
        "pauseType": "delay",
        "timeout": "1",
        "timeoutUnits": "seconds",
        "rate": "1",
        "nbRateUnits": "1",
        "rateUnits": "second",
        "randomFirst": "1",
        "randomLast": "5",
        "randomUnits": "seconds",
        "drop": false,
        "allowrate": false,
        "outputs": 1,
        "x": 580,
        "y": 660,
        "wires": [
            [
                "0b4ecc3bd4fcfcd7"
            ]
        ]
    },
    {
        "id": "0b4ecc3bd4fcfcd7",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "MandaMensaje4",
        "func": "msg.payload = {\n    \"chatId\": ingresa tu chatid en numero,\n    \"type\": \"message\",\n    \"content\": \"O visita el siguiente link para revisar el estado de consumo:\\n-inserta el link de node-red ui\"\n}\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 9,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 770,
        "y": 660,
        "wires": [
            [
                "707bf51a66aa617d"
            ]
        ]
    },
    {
        "id": "1337acdee2b1a6b4",
        "type": "function",
        "z": "655c2b33c17c7d19",
        "name": "Extraer Potencia W",
        "func": "msg.topic=msg.payload.ID;\nmsg.payload=msg.payload.pot;\nreturn msg;\n",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 710,
        "y": 220,
        "wires": [
            [
                "94565075deb613d3"
            ]
        ]
    },
    {
        "id": "94565075deb613d3",
        "type": "ui_gauge",
        "z": "655c2b33c17c7d19",
        "name": "",
        "group": "f323906c191efaa9",
        "order": 0,
        "width": "0",
        "height": "0",
        "gtype": "donut",
        "title": "Potencia. Actual (W)",
        "label": "W",
        "format": "{{value}}",
        "min": 0,
        "max": "1000",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 1080,
        "y": 220,
        "wires": []
    },
    {
        "id": "732283b917fb1d58",
        "type": "inject",
        "z": "655c2b33c17c7d19",
        "name": "",
        "props": [
            {
                "p": "payload"
            },
            {
                "p": "topic",
                "vt": "str"
            }
        ],
        "repeat": "3600",
        "crontab": "",
        "once": false,
        "onceDelay": 0.1,
        "topic": "",
        "payload": "",
        "payloadType": "date",
        "x": 110,
        "y": 540,
        "wires": [
            [
                "e878f1e80d1c113a"
            ]
        ]
    },
    {
        "id": "cad3536bf5f60c36",
        "type": "mqtt-broker",
        "name": "Mosquitto Pi RGK",
        "broker": "192.168.1.73",
        "port": "1883",
        "clientid": "",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": "4",
        "keepalive": "60",
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthRetain": "false",
        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closeQos": "0",
        "closeRetain": "false",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willRetain": "false",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    },
    {
        "id": "08d40e027fb0f7f9",
        "type": "ui_group",
        "name": "Corriente",
        "tab": "7d055cfb531cd518",
        "order": 1,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "f323906c191efaa9",
        "type": "ui_group",
        "name": "Potencia",
        "tab": "7d055cfb531cd518",
        "order": 2,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "b78994697d2872cb",
        "type": "ui_group",
        "name": "Histórico",
        "tab": "7d055cfb531cd518",
        "order": 3,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "6ef270b50846b685",
        "type": "ui_group",
        "name": "Control Prender/Apagar Dispositivo",
        "tab": "7d055cfb531cd518",
        "order": 4,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "7d055cfb531cd518",
        "type": "ui_tab",
        "name": "Consumo Energetico ESP32",
        "icon": "wi-wu-mostlysunny",
        "order": 2,
        "disabled": false,
        "hidden": false
    }
]
```

[Regresar al indice](#Índice)

<a id="ConfNode"></a>
## Configuración del Node-red

Una vez importado el codigo a **Node-red**, accedemos a cada uno de los nodos **function** que llevan por nombre **MandaMensaje** 1, 2, ect. e ingresamos nuestro chatId en numero y el link del node-red ui en que que lo pide (solo en el 4 lo pide).

Ahora pasamos al nodo **Telegram receiver** damos en el simbolo "+" y añadimos el nombre de nuestro bot el tipo "**_bot**" y el token. Despues ingresamos nuestro usuario en **Users** y nuestro chatId en **ChatIds**. Pasamos ahora al nodo **Telegram sender** y elegimos el nombre de nuestro bot donde nos aparece **none** en **Bot**.

Despues configuramos uno nuestros nodos **mysql** dando en el simbolo de "+" y en **Host** agregamos *127.0.0.1*, en **port** ponemos *3306*, en **User** nuestro usuario de **mysql** que creamos en la **Raspberry Pi**, en **Password** ponemos la contraseña del usuario anterior y en **Database** insertamos el nombre de nuestra **DATABASE** la cual fue ***consumo***.
Posteriormente nos dirijimos al otro nodo **mysql** y elegimos el nombre de nuestra **DATABASE** donde dice **none** en **Database**.

<a id="Funcionamiento"></a>
## Funcionamiento del proyecto

Funcionamiento