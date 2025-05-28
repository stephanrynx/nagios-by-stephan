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
Puedes cambiar el '~' por la ruta donde vas a trabajar en tu proyecto.

2.- Crear el archivo `Dockerfile`:
Usa vim, nano o tu editor de texto favorito para crear el archivo Dockerfile dentro de ~/nagios-proyecto.

```
vim Dockerfile
```
Pega lo siguiente en el Dockerfile.

```
# Dockerfile para Nagios Core en Ubuntu 24.04

# Paso 1: Elegir la imagen base
FROM ubuntu:24.04

# Paso 2: Establecer variables de entorno para evitar preguntas interactivas durante la instalación
ENV DEBIAN_FRONTEND=noninteractive

# Paso 3: Actualizar la lista de paquetes e instalar dependencias necesarias
# Incluimos Apache2, PHP, librerías de desarrollo, y utilidades de compilación (build-essential)
# para compilar Nagios Core y los Plugins.
# 'adduser' es para gestionar usuarios y grupos.
# 'ca-certificates' es crucial para verificar certificados SSL al descargar de HTTPS.
# '--no-install-recommends' reduce el tamaño de la imagen final.
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

# Paso 4: Crear usuarios y grupos de Nagios
# Estos usuarios y grupos son necesarios para el funcionamiento de Nagios.
RUN adduser --system --no-create-home --shell /bin/false --group nagios && \
    adduser --system --no-create-home --shell /bin/false --group nagcmd && \
    usermod -a -G nagcmd nagios && \
    usermod -a -G nagcmd www-data

# Paso 5: Descargar y compilar Nagios Core
# Definimos la versión de Nagios Core que queremos instalar y el directorio de instalación.
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
    a2enmod rewrite cgi # Habilita módulos de Apache necesarios

# Paso 6: Descargar y compilar Nagios Plugins
# Definimos la versión de Nagios Plugins y lo compilamos para el mismo prefijo de Nagios Core.
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

# Paso 7: Configuración de la autenticación de Apache para la interfaz web de Nagios
# Creamos el directorio 'etc' si no existe, luego creamos el archivo de usuarios htpasswd.
WORKDIR ${NAGIOS_HOME}
RUN mkdir -p ${NAGIOS_HOME}/etc && \
    htpasswd -bc ${NAGIOS_HOME}/etc/htpasswd.users dylan dylan
# NOTA: Las credenciales por defecto son 'dylan' / 'dylan'. CAMBIA ESTO EN UN ENTORNO DE PRODUCCIÓN.

# Paso 8: Configurar ServerName para Apache (suprimir la advertencia de FQDN)
# Esto evita la advertencia "Could not reliably determine the server's fully qualified domain name".
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

# Paso 9: Configurar permisos de usuario en cgi.cfg de Nagios
# Esto es CRÍTICO para que el usuario 'dylan' pueda ver los hosts y servicios en la interfaz web.
RUN sed -i 's/^[[:space:]]*authorized_for_all_hosts=.*/authorized_for_all_hosts=dylan,nagiosadmin/' ${NAGIOS_HOME}/etc/cgi.cfg && \
    sed -i 's/^[[:space:]]*authorized_for_all_services=.*/authorized_for_all_services=dylan,nagiosadmin/' ${NAGIOS_HOME}/etc/cgi.cfg

# Paso 10: Configuración de Nagios Core (modificar nagios.cfg)
# 'daemon_mode=0' para que Nagios no se ejecute como un demonio separado, sino como un proceso en primer plano.
# 'check_external_commands=1' y 'command_check_interval' para permitir la comunicación con la interfaz web.
RUN sed -i 's/^[[:space:]]*daemon_mode=1/daemon_mode=0/' ${NAGIOS_HOME}/etc/nagios.cfg && \
    sed -i 's/^[[:space:]]*check_external_commands=0/check_external_commands=1/' ${NAGIOS_HOME}/etc/nagios.cfg && \
    sed -i 's/^[[:space:]]*command_check_interval=1/command_check_interval=3/' ${NAGIOS_HOME}/etc/nagios.cfg

# Paso 11: Limpieza final
# Eliminamos los archivos de instalación temporales para reducir el tamaño final de la imagen.
RUN rm -rf /tmp/*

# Paso 12: Exponer el puerto
# Indica que el contenedor escuchará en el puerto 80 (puerto estándar de HTTP).
EXPOSE 80

# Paso 13: Script de inicio del contenedor
# Copiamos el script que manejará el inicio de Apache y Nagios.
COPY start_nagios.sh /usr/local/bin/start_nagios
RUN chmod +x /usr/local/bin/start_nagios

# Comando final que se ejecuta al iniciar el contenedor
# Este comando inicia nuestro script, que a su vez inicia Apache y Nagios.
CMD ["/usr/local/bin/start_nagios"]
```


### 3. Crear el archivo start_nagios.sh:
Crea el archivo start_nagios.sh en el mismo directorio ~/nagios-proyecto.
























