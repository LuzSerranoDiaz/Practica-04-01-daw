# Practica-04-01-daw

En esta practica se utilizará ansible para instalar una pila lamp junto a wordpress

## Playbooks

# Install_lamp_backend

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
```

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

## Install_lamp_frontend
