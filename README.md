# Practica-04-01-daw

En esta practica se utilizará ansible para instalar una pila lamp junto a wordpress

## Playbooks

### Install_lamp_backend

```yml
---
- name: Playbook para instalar la pila LAMP en el Backend
  hosts: backend
  become: yes

  vars_files:
    - ../vars/variables.yml
```
Archivo con las variables del sistema.
```yml

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Instalar el sistema gestor de bases de datos MySQL
      apt:
        name: mysql-server
        state: present

    - name: Instalamos el gestor de paquetes de Python pip3
      apt: 
        name: python3-pip
        state: present

    - name: Instalamos el módulo de pymysql
      pip:
        name: pymysql
        state: present
```
Los pasos hasta ahora son autodescriptivos gracias a sus nombres.
```yml

    - name: Crear una base de datos
      mysql_db:
        name: "{{ db.name }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 

    - name: Crear el usuario de la base de datos
      mysql_user:         
        name: "{{ db.user }}"
        password: "{{ db.password }}"
        priv: "{{ db.name }}.*:ALL"
        host: "%"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
```
Creamos una base de datos con el nombre dado en el archivo de variables junto a un usuario utilizando los datos del archivo .env dandole todos los permisos y permitiendole conectarse desde cualquier conexión.
```yml

    - name: Configuramos MySQL para permitir conexiones desde cualquier interfaz
      replace:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: 127.0.0.1
        replace: 0.0.0.0

    - name: Reiniciamos el servicio de base de datos
      service:
        name: mysql
        state: restarted
```
Se configura con el archivo mysqld.conf y se reinicia el servicio.

### Install_lamp_frontend

```yml
---
- name: Playbook para instalar la pila LAMP en el FrontEnd
  hosts: frontend
  become: yes

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Instalar el servidor web Apache
      apt:
        name: apache2
        state: present

    - name: Instalar PHP y los módulos necesarios
      apt: 
        name:
          - php
          - php-mysql
          - libapache2-mod-php
          - php-bcmath
          - php-curl
          - php-gd
          - php-imagick
          - php-intl
          - php-memcached
          - php-mbstring
          - php-dom
          - php-zip
          - php-cli
        state: present
```
Se instala apache y php con todos sus modulos.
```yml

    - name: Modificamos el valor max_input_vars de PHP
      replace: 
        path: /etc/php/8.1/apache2/php.ini
        regexp: ;max_input_vars = 1000
        replace: max_input_vars = 5000

    - name: Modificamos el valor de memory_limit de PHP
      replace: 
        path: /etc/php/8.1/apache2/php.ini
        regexp: memory_limit = 128M
        replace: memory_limit = 256M

    - name: Modificamos el valor de post_max_size de PHP
      replace: 
        path: /etc/php/8.1/apache2/php.ini
        regexp: post_max_size = 8M
        replace: post_max_size = 128M

    - name: Modificamos el valor de upload_max_filesize de PHP
      replace:
        path: /etc/php/8.1/apache2/php.ini
        regexp: upload_max_filesize = 2M
        replace: upload_max_filesize = 128M

```
Configuramos php, aumentando el número maximo de variables, memoria, el tamaño maximo de una peticion post y el tamaño maximo de archivos que se pueden subir.
```yml

    - name: Copiar el archivo de configuración de Apache
      copy:
        src: ../templates/000-default.conf
        dest: /etc/apache2/sites-available/
        mode: 0755

    - name: Habilitar el módulo rewrite de Apache
      apache2_module:
        name: rewrite
        state: present
```
Se confirgura apache y se habilita el módulo rewrite.
```

    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
```
### Setup_letsencrypt_https.yml
```yml
---
- name: Playbook para configurar HTTPS
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Desinstalar instalaciones previas de Certbot
      apt:
        name: certbot
        state: absent

    - name: Instalar Certbot con snap
      # command: snap install --classic certbot
      snap:
        name: certbot
        classic: yes
        state: present

    - name: Crear un alias para el comando certbot
      command: ln -s -f /snap/bin/certbot /usr/bin/certbot
```
Desinstalamos versiones anterioras, instalamos certbot clasico y se crea un alias para cerbot para poder usarlo en bash. 
```yml
    - name: Solicitar y configurar certificado SSL/TLS a Let's Encrypt con certbot
      command:
        certbot --apache \
        -m {{ certbot.email }} \
        --agree-tos \
        --no-eff-email \
        --non-interactive \
        -d {{ certbot.domain }}
```
Con el comando `certbot --apache` realizamos el certificado y con estos siguientes parametros automatizamos el proceso:
* `-m $CERTIFICATE_EMAIL` : indicamos la direccion de correo que en este caso es `demo@demo`
* `--agree-tos` : indica que aceptamos los terminos de uso
* `--no-eff-email` : indica que no queremos compartir nuestro email con la 'Electronic Frontier Foundation'
* `-d $CERTIFICATE_DOMAIN` : indica el dominio, que en nuestro caso es 'practica-15.ddns.net', el dominio conseguido con el servicio de 'no-ip'
* `--non-interactive` : indica que no solicite ningún tipo de dato de teclado.

