# Nagios Docker Setup

Este repositorio contiene un Dockerfile para construir una imagen de Docker que instala y configura Nagios Core desde cero en una base de Ubuntu 20.04.

## Requisitos

Antes de comenzar, asegúrate de tener Docker instalado en tu sistema. Puedes descargar Docker desde [aquí](https://docs.docker.com/get-docker/).

## Construir la Imagen

Sigue los pasos a continuación para construir la imagen de Docker con Nagios:

1. Clona este repositorio o crea un archivo `Dockerfile` en un nuevo directorio.

2. Copia el contenido del siguiente Dockerfile en tu archivo `Dockerfile`:

    ```Dockerfile
    # Utiliza la imagen base de Ubuntu
    FROM ubuntu:20.04

    # Establece las variables de entorno para evitar la interacción durante la instalación
    ENV DEBIAN_FRONTEND=noninteractive

    # Actualiza el sistema y instala las dependencias necesarias
    RUN apt-get update && \
        apt-get install -y wget build-essential unzip apache2 php libapache2-mod-php libgd-dev libperl-dev libssl-dev daemon iputils-ping

    # Crea el usuario y grupo de Nagios antes de la instalación
    RUN useradd nagios && \
        groupadd nagcmd && \
        usermod -aG nagcmd nagios && \
        usermod -aG nagcmd www-data

    # Descarga Nagios Core
    RUN wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.5.2.tar.gz && \
        tar -zxvf nagios-4.5.2.tar.gz

    # Instala Nagios Core
    RUN cd nagios-4.5.2 && \
        ./configure --with-command-group=nagcmd > configure.log 2>&1 || { cat configure.log; exit 1; } && \
        make all > make_all.log 2>&1 || { cat make_all.log; exit 1; } && \
        make install > make_install.log 2>&1 || { cat make_install.log; exit 1; } && \
        make install-init && \
        make install-config && \
        make install-commandmode && \
        make install-webconf

    # Instala los plugins de Nagios
    RUN wget https://nagios-plugins.org/download/nagios-plugins-2.3.3.tar.gz && \
        tar -zxvf nagios-plugins-2.3.3.tar.gz && \
        cd nagios-plugins-2.3.3 && \
        ./configure --with-nagios-user=nagios --with-nagios-group=nagios > configure_plugins.log 2>&1 || { cat configure_plugins.log; exit 1; } && \
        make > make_plugins.log 2>&1 || { cat make_plugins.log; exit 1; } && \
        make install > make_install_plugins.log 2>&1 || { cat make_install_plugins.log; exit 1; }

    # Configura Nagios
    RUN htpasswd -b -c /usr/local/nagios/etc/htpasswd.users nagiosadmin nagios && \
        sed -i 's/^check_external_commands=0/check_external_commands=1/' /usr/local/nagios/etc/nagios.cfg

    # Configura Apache
    RUN a2enmod cgi && \
        echo "ServerName localhost" >> /etc/apache2/apache2.conf

    # Agrega un script de inicio para Nagios y Apache
    RUN echo '#!/bin/bash\n\
    trap "exit" SIGINT SIGTERM\n\
    service apache2 start\n\
    /usr/local/nagios/bin/nagios /usr/local/nagios/etc/nagios.cfg\n\
    tail -f /usr/local/nagios/var/nagios.log' > /start.sh && \
        chmod +x /start.sh

    # Exponer los puertos necesarios
    EXPOSE 80 443

    # Define el comando por defecto para el contenedor
    CMD ["/start.sh"]
    ```

3. Navega al directorio donde se encuentra tu archivo `Dockerfile` y construye la imagen de Docker con el siguiente comando:

    ```bash
    docker build -t strife_prueba2:strife_nagios .
    ```

## Ejecutar el Contenedor

Una vez que la imagen esté construida, puedes ejecutar el contenedor utilizando el siguiente comando:

```bash
docker run --name nagios4 -p 0.0.0.0:8888:80 strife_prueba2:strife_nagios
