# Despliegue de Nagios Core con Docker en Ubuntu 24.04
## Este repositorio contiene los archivos necesarios para desplegar una instancia de Nagios Core en un contenedor Docker utilizando Ubuntu 24.04 como imagen base. La guía detalla el proceso completo, desde la preparación del sistema anfitrión hasta la configuración y acceso a la interfaz web de Nagios.

## Contenido del Repositorio
`Dockerfile`: Contiene las instrucciones para construir la imagen Docker de Nagios Core.
`start_nagios.sh`: Un script auxiliar para iniciar los servicios de Apache y Nagios dentro del contenedor.

### Requisitos Previos
Antes de proceder, el usuario debe asegurarse de tener un sistema Ubuntu 24.04 operativo y acceso con privilegios sudo.

### 1. Preparación del Sistema Operativo Anfitrión (Ubuntu 24.04)
Se recomienda actualizar el sistema anfitrión antes de instalar Docker.

### Actualizar el sistema:

Bash

sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
Instalar paquetes necesarios para Docker:

Bash

sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
### 2. Instalación de Docker y Docker Compose
Se procede con la instalación de la última versión de Docker Engine y Docker Compose (v2) siguiendo las recomendaciones oficiales.

### Añadir la clave GPG oficial de Docker:

Bash

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
Añadir el repositorio de Docker a APT sources:

Bash

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
Actualizar el índice de paquetes APT e instalar Docker Engine y Docker Compose:

    sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verificar la instalación de Docker:

Bash

sudo docker run hello-world
Se espera un mensaje que confirme la instalación exitosa.

Añadir el usuario actual al grupo docker (opcional pero recomendado):
Esto permite ejecutar comandos docker sin el prefijo sudo. El usuario debe cerrar y volver a abrir su sesión de terminal para que los cambios surtan efecto.

Bash

sudo usermod -aG docker ${USER}
# Cierra la terminal y vuelve a abrirla, o ejecuta 'newgrp docker'
3. Creación del Directorio del Proyecto y Archivos
Se crea la estructura del proyecto Docker para Nagios.

Crear el directorio del proyecto:

Bash

mkdir ~/nagios-proyecto
cd ~/nagios-proyecto
Crear el archivo Dockerfile:
El usuario debe crear el archivo Dockerfile dentro del directorio ~/nagios-proyecto y pegar el siguiente contenido:

Dockerfile

# Dockerfile para Nagios Core en Ubuntu 24.04

# Paso 1: Se elige la imagen base de Ubuntu 24.04.
FROM ubuntu:24.04

# Paso 2: Se establecen variables de entorno para evitar preguntas interactivas durante la instalación,
# lo cual es crucial para construcciones automatizadas de Docker.
ENV DEBIAN_FRONTEND=noninteractive

# Paso 3: Se actualiza la lista de paquetes y se instalan las dependencias necesarias.
# Se incluye Apache2, PHP, librerías de desarrollo, y utilidades de compilación (build-essential)
# para compilar Nagios Core y los Plugins. 'adduser' es para gestionar usuarios y grupos,
# y 'ca-certificates' es crucial para verificar certificados SSL al descargar de HTTPS.
# La opción '--no-install-recommends' se utiliza para reducir el tamaño final de la imagen.
RUN apt-get update && \
    apt-get install -y \
    adduser \
    ca-certificates \
    apache2 \
    php \
    libapache2-mod-php \
    build-essential \
    unzip \
    wget \
    libgd-dev \
    libjpeg-dev \
    libpng-dev \
    libperl-dev \
    libssl-dev \
    fping \
    --no-install-recommends && \
    rm -rf /var/lib/apt/lists/*

# Paso 4: Se crean los usuarios y grupos dedicados que Nagios necesita para su operación.
RUN adduser --system --no-create-home --shell /bin/false --group nagios && \
    adduser --system --no-create-home --shell /bin/false --group nagcmd && \
    usermod -a -G nagcmd nagios && \
    usermod -a -G nagcmd www-data

# Paso 5: Se descarga y compila Nagios Core.
# Se definen la versión de Nagios Core y el directorio de instalación base.
ENV NAGIOS_VERSION="4.5.9"
ENV NAGIOS_HOME="/opt/nagios"

WORKDIR /tmp
RUN wget -O nagioscore.tar.gz "https://github.com/NagiosEnterprises/nagioscore/releases/download/nagios-${NAGIOS_VERSION}/nagios-${NAGIOS_VERSION}.tar.gz" && \
    tar xzf nagioscore.tar.gz && \
    cd nagios-${NAGIOS_VERSION} && \
    ./configure --prefix=${NAGIOS_HOME} \
                --with-nagios-group=nagios \
                --with-command-group=nagcmd \
                --with-httpd-conf=/etc/apache2/sites-enabled \
                --with-mail=/usr/bin/sendmail \
                --enable-event-broker && \
    make all && \
    make install && \
    make install-init && \
    make install-commandmode && \
    make install-webconf && \
    make install-exfoliation && \
    make install-config && \
    a2enmod rewrite cgi # Se habilitan los módulos de Apache necesarios

# Paso 6: Se descarga y compila Nagios Plugins.
# Se define la versión de los plugins y se compilan para el mismo prefijo de Nagios Core.
ENV NAGIOS_PLUGINS_VERSION="2.4.12"

WORKDIR /tmp
RUN wget -O nagios-plugins.tar.gz "https://github.com/nagios-plugins/nagios-plugins/releases/download/release-${NAGIOS_PLUGINS_VERSION}/nagios-plugins-${NAGIOS_PLUGINS_VERSION}.tar.gz" && \
    tar xzf nagios-plugins.tar.gz && \
    cd nagios-plugins-${NAGIOS_PLUGINS_VERSION} && \
    ./configure --prefix=${NAGIOS_HOME} \
                --with-nagios-user=nagios \
                --with-nagios-group=nagios && \
    make && \
    make install

# Paso 7: Configuración de la autenticación de Apache para la interfaz web de Nagios.
# Se crea el directorio 'etc' si no existe, luego se crea el archivo de usuarios htpasswd.
WORKDIR ${NAGIOS_HOME}
RUN mkdir -p ${NAGIOS_HOME}/etc && \
    htpasswd -bc ${NAGIOS_HOME}/etc/htpasswd.users dylan dylan
# NOTA: Las credenciales por defecto son 'dylan' / 'dylan'. Se recomienda cambiar esto en un entorno de producción.

# Paso 8: Se configura la directiva ServerName para Apache.
# Esto suprime la advertencia "Could not reliably determine the server's fully qualified domain name".
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

# Paso 9: Se configuran los permisos de usuario en cgi.cfg de Nagios.
# Esto es CRÍTICO para que el usuario 'dylan' pueda ver los hosts y servicios en la interfaz web.
RUN sed -i 's/^[[:space:]]*authorized_for_all_hosts=.*/authorized_for_all_hosts=dylan,nagiosadmin/' ${NAGIOS_HOME}/etc/cgi.cfg && \
    sed -i 's/^[[:space:]]*authorized_for_all_services=.*/authorized_for_all_services=dylan,nagiosadmin/' ${NAGIOS_HOME}/etc/cgi.cfg

# Paso 10: Configuración de Nagios Core (modificación de nagios.cfg).
# 'daemon_mode=0' se establece para que Nagios no se ejecute como un demonio separado,
# sino como un proceso en primer plano, adecuado para contenedores.
# 'check_external_commands=1' y 'command_check_interval' se habilitan para permitir
# la comunicación con la interfaz web.
RUN sed -i 's/^[[:space:]]*daemon_mode=1/daemon_mode=0/' ${NAGIOS_HOME}/etc/nagios.cfg && \
    sed -i 's/^[[:space:]]*check_external_commands=0/check_external_commands=1/' ${NAGIOS_HOME}/etc/nagios.cfg && \
    sed -i 's/^[[:space:]]*command_check_interval=1/command_check_interval=3/' ${NAGIOS_HOME}/etc/nagios.cfg

# Paso 11: Limpieza final.
# Se eliminan los archivos de instalación temporales para reducir el tamaño final de la imagen.
RUN rm -rf /tmp/*

# Paso 12: Se expone el puerto 80 del contenedor.
# Esto indica que el contenedor está diseñado para escuchar en el puerto 80 (HTTP estándar).
EXPOSE 80

# Paso 13: Script de inicio del contenedor.
# Se copia el script `start_nagios.sh` que manejará el inicio de Apache y Nagios.
COPY start_nagios.sh /usr/local/bin/start_nagios
RUN chmod +x /usr/local/bin/start_nagios

# Se define el comando final que se ejecuta al iniciar el contenedor.
# Este comando inicia el script `start_nagios.sh`, el cual a su vez inicia Apache y Nagios.
CMD ["/usr/local/bin/start_nagios"]
Crear el archivo start_nagios.sh:
El usuario debe crear el archivo start_nagios.sh en el mismo directorio ~/nagios-proyecto y pegar el siguiente contenido:

Bash

#!/bin/bash

# Se inicia Apache en primer plano (-D FOREGROUND) para que Docker lo considere el proceso principal
# y para evitar que el contenedor se cierre prematuramente.
/usr/sbin/apache2ctl -D FOREGROUND &

# Se inicia Nagios Core en segundo plano.
# La opción -d indica que se ejecute como un proceso demonio, aunque en el Dockerfile se configure
# daemon_mode=0, el parámetro -d sigue siendo necesario para iniciar el proceso de Nagios.
/opt/nagios/bin/nagios -d /opt/nagios/etc/nagios.cfg

# Se mantiene el contenedor en ejecución.
# `tail -f` sigue los logs de Nagios y Apache, asegurando que el proceso principal del contenedor
# se mantenga activo indefinidamente.
tail -f /opt/nagios/var/nagios.log /var/log/apache2/error.log
4. Construir la Imagen Docker
Desde el directorio ~/nagios-proyecto en la terminal, el usuario procede a construir la imagen Docker.

Bash

docker build -t dy-ca-nagios-ubuntu:v1.0 .
docker build: Comando para construir una imagen.
-t dy-ca-nagios-ubuntu:v1.0: Asigna un nombre (dy-ca-nagios-ubuntu) y una etiqueta/versión (v1.0) a la imagen.
.: Indica que el Dockerfile se encuentra en el directorio actual.
Este proceso puede tomar tiempo, ya que implica la descarga de la imagen base de Ubuntu, la instalación de dependencias y la compilación de Nagios Core y sus plugins.

5. Ejecutar el Contenedor Docker
Una vez que la imagen se haya construido con éxito, el usuario puede ejecutarla.

Ejecutar el contenedor:

Bash

docker run -d -p 8080:80 dy-ca-nagios-ubuntu:v1.0
docker run: Comando para iniciar un contenedor a partir de una imagen.
-d: Ejecuta el contenedor en modo "detached" (segundo plano), liberando la terminal del usuario.
-p 8080:80: Mapea el puerto 8080 de la máquina anfitriona al puerto 80 del contenedor. Esto permite acceder a Nagios a través del puerto 8080 desde un navegador externo.
dy-ca-nagios-ubuntu:v1.0: Nombre y etiqueta de la imagen construida.
Verificar que el contenedor se esté ejecutando:

Bash

docker ps
Se espera que el contenedor esté listado, mostrando los puertos mapeados (e.g., 0.0.0.0:8080->80/tcp).

6. Acceder a la Interfaz Web de Nagios
Abrir un navegador web.

Ingresar la URL de Nagios:

Si el usuario está en la misma máquina donde ejecutó Docker:

http://localhost:8080/nagios
Si el usuario está en una instancia de EC2 o VM remota:
Se necesita la IP pública de la instancia. Es fundamental que el grupo de seguridad (EC2) o firewall (VM) permita el tráfico TCP entrante en el puerto 8080.

http://[TU_IP_PUBLICA_DE_EC2]:8080/nagios
Iniciar sesión:
Cuando se solicite, el usuario debe ingresar las credenciales configuradas en el Dockerfile (Paso 7):

Usuario: dylan
Contraseña: dylan
Se espera que la interfaz de Nagios Core sea visible y completamente funcional.

7. Comandos Útiles de Docker
A continuación, se listan algunos comandos de Docker que pueden ser de utilidad:

Ver logs de un contenedor:

Bash

docker logs [ID_DEL_CONTENEDOR_O_NOMBRE]
(El ID o nombre del contenedor se puede obtener con docker ps).

Detener un contenedor:

Bash

docker stop [ID_DEL_CONTENEDOR_O_NOMBRE]
Eliminar un contenedor (debe estar detenido previamente):

Bash

docker rm [ID_DEL_CONTENEDOR_O_NOMBRE]
Eliminar una imagen:

Bash

docker rmi dy-ca-nagios-ubuntu:v1.0