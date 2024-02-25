# Practica-04-01-daw

En esta practica se utilizará ansible para instalar una pila lamp junto a wordpress

## Variables

```yml
ip:
    frontend: 172.31.93.137
    backend: 172.31.87.196
db:
    name: prestashop_db
    user: prestashop_user
    password: prestashop_password

wordpress:
    title: practica-04-01
    admin: LuzSerranoDiaz
    password: password123
    email: diaz03luz@gmail.com

certbot:
    email: diaz03luz@gmail.com
    domain: ansible-daw.ddns.net
```

## Inventory

```
[backend]
172.31.87.196
#nodo1

[frontend]
172.31.93.137
#nodo2

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/vockey.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
```

## Templates

```
ServerSignature Off
ServerTokens Prod

<VirtualHost *:80>
        DocumentRoot /var/www/html
        DirectoryIndex index.php index.html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        <Directory "/var/www/html">
            AllowOverride All
        </Directory>
</VirtualHost>
```

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

### Deploy

```yml
---
- name: Playbook para hacer el deploy de la aplicación web Wordpress
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: decargar wp-cli, moverlo a bin y darle permisos
      ansible.builtin.get_url:
        url: https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
        dest: /usr/local/bin/wp
        mode: a+x

    - name: borrar 
      shell: /bin/rm -rf /var/www/html*
```
Se descarga la herramiento wp-cli, se le da permisos y se borran versiones anteriores de WP
```yml
    - name: descargar el codigo de wordpress
      command:
        wp core download \
        --locale=es_ES \
        --path=/var/www/html \
        --allow-root
```
Se borran las versiones previas de WordPress y se instala una version nueva con estos parametros:
* `--locale=es_ES` : especifica el idioma.
* `--path=/var/www/html` : especifica el directorio de la descarga. 
* `--allow-root` : para poder ejecutar el comando como sudo.
```yml
    - name: Set up wp-config
      command:        
        wp config create \
        --dbname={{ db.name }} \
        --dbuser={{ db.user }} \
        --dbpass={{ db.password }} \
        --dbhost={{ ip.backend }} \
        --path=/var/www/html \
        --allow-root
```
Con este comando se crea el archivo de configuración, se automatiza con estos parametros:
* `--dbname` : se le especifica el nombre de la bases de datos.
* `--dbuser` : se le especifica el nombre del usuario de la base de datos.
* `--dbpass` : se le especifica la contraseña del usuario de la base de datos.
* `--dbhost` : se le especifica host de la base de datos.
```yml
    - name: install wp
      command:
        wp core install \
        --url={{ certbot.domain }} \
        --title="{{ wordpress.title }}" \
        --admin_user={{ wordpress.admin }} \
        --admin_password={{ wordpress.password }} \
        --admin_email={{ wordpress.email }} \
        --path=/var/www/html \
        --allow-root  
```
Con este comando se completa la instalación de wordpress y se automatiza con estos parametros:
* `--url` : se especifica el dominio del sitio de WordPress.
* `--title` : se especifica el titulo del sitio WordPress.
* `--admin-user` : se especifica el usuario administrador.
* `--admin-password` : se especifica la contraseña del usuario administrador.
* `--admin-email` : se especifica el email del usuario administrador.
```
    - name: install plugin
      command:
        wp plugin install wps-hide-login --path=/var/www/html --allow-root

    - name: PermaLinks
      command: wp --path=/var/www/html plugin install permalink-manager --activate --allow-root

    - name: PermaLinks config
      command: wp --path=/var/www/html option update permalink_structure '/%postname%/' --allow-root
```
Se instala los plugins `wps-hide-login` y `permalink-manager`.
Se cambia la estructura del permalink a `/%postname%/`

### Main.yml
```yml
---
- import_playbook: playbooks/install_lamp_frontend.yml
- import_playbook: playbooks/install_lamp_backend.yml
- import_playbook: playbooks/setup_letsencrypt_https.yml
- import_playbook: playbooks/deploy.yml
```
Ejecuta cada playbook en este orden.



