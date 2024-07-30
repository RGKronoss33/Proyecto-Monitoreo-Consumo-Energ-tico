<a id="Presentación"></a>
# Proyecto-Monitoreo-Consumo-Energ-tico
Este proyecto utiliza una ESP32, una Raspberry Pi, entre otros componentes para su funcionamiento. Con el podrás medir la corriente y potencia de uno o más dispositivos conectados a un multi contacto.

<a id="Índice"></a>
## Índice

1. [Inicio del proyecto](#Presentación)

1. [Índice](#Índice)

1. [Materiales a utilizar](#Materiales)

1. [Requisitos para empezar](#Requisitos)

1. [Creacion de la base de datos](#Base_de_datos)

1. [Circuito]()

1. [Código]()

1. [Configuración Node-Red]()

1. [Funcionamiento del proyecto]()

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

```

python

def hello_world():

print("Hello, World!")

```