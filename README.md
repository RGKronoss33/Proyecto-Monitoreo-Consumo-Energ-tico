# Proyecto-Monitoreo-Consumo-Energ-tico
Este proyecto utiliza una ESP32, una Raspberry Pi, entre otros componentes para su funcionamiento. Con el podrás medir la corriente y potencia de uno o más dispositivos conectados a un multi contacto.

## Índice

1. Inicio del proyecto

1. Índice

1. Materiales a utilizar

1. Requisitos para empezar

1. Creacion de la base de datos

1. Circuito

1. Código

1. Configuración Node-Red

1. Funcionamiento del proyecto

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

## Requisitos para empezar

* Tener previamente instalado la **IDE de Arduino**, **Node-red** y **Mysql** (*mariadb-server*) en tu Raspberry Pi.

* Tener conocimientos básicos de cómo programar en **C++** ya que este es el lenguaje utilizado por la **IDE de Arduino**.

## Creacion de la base de datos

Para la creación de nuestra base de datos ocuparemos **Mysql** (*mariadb-server*), con el almacenaremos las lecturas de corriente y potencia que lea nuestro **SCT013** en nuestra Raspberry Pi.