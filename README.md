# Laboratorio de Docker: Despliegue de Nagios con Docker en Ubuntu 24.04

Este proyecto fue realizado en una instancia de laboratorio EC2  de AWS, el sistema operativo usado es Ubuntu 24.04 y para efectos de pruebas se liberaron los grupos de seguridad para poder conectarse desde cualquier red al servidor nagios. Esto ultimo puede ser omitido en caso de que realicen el laboratorio de forma local en una maquina virtual en sus computadores.

### Paso 1: Las bases antes de iniciar con este proyecto.

Para iniciar este proyecto necesitamos actualizar nuestro sistema operativo:

```
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

Tambien debemos de instalar los paquetes necesarios para isntalar Docker:

```
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
```

### Paso 2: Instalar Docker y Docker Compose

Vamos a instalar la última versión de Docker Engine y Docker Compose (v2) siguiendo las mejores prácticas.


1.- Añadir la clave GPG oficial de Docker
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

```
2.- Añadir el repositorio de Docker a APT sources:
```
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

