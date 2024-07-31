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

1. [Código](#Codigo)

1. [Configuración Node-Red](#Node)

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

<a id="Requisitos"></a>
## Requisitos para empezar

* Tener previamente instalado la **IDE de Arduino**, **Node-red** y **Mysql** (*mariadb-server*) en tu Raspberry Pi.

* Se necesita de un usuario con todos los privilegios en **Mysql**.

* Tener conocimientos básicos de cómo programar en **C++** ya que este es el lenguaje utilizado por la **IDE de Arduino**.

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

<a id="Circuito"></a>
## Circuito

Circuito

<a id="Codigo"></a>
## Codigo

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
int statusLedPin = 33;  // Esta variable controla el led para saber si se conecto a internet la esp32
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

<a id="Telegram"></a>
## Bot Telegram

Para inciar debemos tener instalado previamente **Telegram**.

Abrimos **Telegram** y buscamos el la lupa **BotFather** y lo inciamos con el boton **Iniciar** o introduciendo lo siguiente : **/start**

Una vez iniciado intriducimos **/newbot** para crear nuestro bot al que le mandaremos instrucciones para controlar los pines de la **ESP32**.

Despues de mandar **/newbot** nos preguntara cual será el nombre de nuestro bot al cual le podremos poner cualquier nombre como el siguiente: **Bot Consumo Energetico**. Una vez mandado nos pedira otro nombre para el bot el cual usaremos en **Node-red** el cual puede ser el siguiente : **consumo_energetico_bot**.
Una vez que mandemos el nombre nos dara un mensaje en el cual nos dara en una parte algo parecido a **Use this token to access the HTTP API:** y ese sera el token que utilizaremos mas adelante en **Node-red**.   
- **NOTA: Es importante que lleve "_" si el nombre lo quieres poner espaciado y que al final termine con *bot***.

Ahora para que le puedan llegar los mensajes, ahi mismo en el **BotFather** mandamos un **/setprivacy** y elegimos nuestro bot. Despues de eso mandamos un **Disable** y con eso ya esta configurado el bot.

Despues salimos del **BotFather** y ahora buscamos **IDBot**, dentro de el mandamos **/getid** el cual sera nuestro chatId que nos servira para el **Node-red**.


<a id="Node"></a>
## Configuración Node-Red

Aqui viene la configuracion del **Node-red** en formato **JSON** para que dentro de tu **Node-red** presiones en el menu de Hamburgesa, import y pegues este codigo:

```javascrip

```

<a id="Funcionamiento"></a>
## Funcionamiento del proyecto

Funcionamiento