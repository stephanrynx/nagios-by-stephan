# Laboratorio de Docker: Despliegue de Nagios con Docker en Ubuntu 24.04

Este proyecto fue realizado en una instancia de laboratorio EC2  de AWS, el sistema operativo usado es Ubuntu 24.04 y para efectos de pruebas se liberaron los grupos de seguridad para poder conectarse desde cualquier red al servidor nagios. Esto ultimo puede ser omitido en caso de que realicen el laboratorio de forma local en una maquina virtual en sus computadores.

### Paso 1: Las bases antes de iniciar con este proyecto.

1.- Para iniciar este proyecto necesitamos actualizar nuestro sistema operativo:

```
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

2.- Tambien debemos de instalar los paquetes necesarios para isntalar Docker:

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
3.- Actualizar el índice de paquetes APT y instalar Docker Engine y Docker Compose:
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
``` 
4.- Verificar la instalación de Docker:

```
sudo docker run hello-world
```
Deberías ver un mensaje que indica que la instalación fue exitosa.

5.- Añadir tu usuario al grupo docker (opcional pero recomendado):
Esto te permite ejecutar comandos docker sin sudo. Cierra y vuelve a abrir tu sesión de terminal para que los cambios surtan efecto.
```
sudo usermod -aG docker ${USER}
# Cierra la terminal y vuelve a abrirla, o ejecuta 'newgrp docker'
```

### Paso 3: Creación del Directorio del Proyecto y Archivos.

Vamos a crear la estructura de tu proyecto Docker para Nagios.

1.-Crear el directorio del proyecto:
```
mkdir ~/nagios-proyecto
cd ~/nagios-proyecto
```
Puedes cambiar el '~' por la ruta donde vas a trabajar en tu proycto.




