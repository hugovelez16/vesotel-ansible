# Configuración de Software y Orquestación con Ansible

Este repositorio contiene la configuración de software y aprovisionamiento interno de los servidores creados por Terraform en Proxmox.

---

## ¿Qué es Ansible y para qué sirve?

**Ansible** es una herramienta de gestión de configuración y orquestación de sistemas de código abierto. Su propósito es automatizar la instalación de paquetes, la configuración de archivos de sistema, la gestión de usuarios, el despliegue de aplicaciones y la seguridad de los servidores.

### Características Clave:

1. **Agentless (Sin agentes)**: A diferencia de otras herramientas (como Puppet o Chef), Ansible no requiere instalar ningún programa "agente" dentro de los servidores destino. Se conecta utilizando la infraestructura estándar de **SSH** (o WinRM en Windows), ejecuta los comandos necesarios a través de módulos temporales de Python y luego se desconecta sin dejar rastro.
2. **Declarativo e Idempotente**: Describes el estado en el que quieres que esté tu servidor (ej: *"El paquete docker-ce debe estar instalado"*). Si vuelves a ejecutar el playbook y el paquete ya está instalado, Ansible no hace nada (devuelve `OK`). Esto hace que sea extremadamente seguro ejecutar los playbooks múltiples veces.
3. **Sintaxis Simple en YAML**: Las configuraciones se definen en archivos YAML legibles llamados **Playbooks**.

---

## Conceptos Clave de este Repositorio

1. **Inventario (`hosts`)**: Es el listado de tus servidores físicos o virtuales agrupados por su rol (base de datos, aplicaciones, gateway, almacenamiento), con sus correspondientes IPs privadas y los usuarios de conexión SSH.
2. **Playbook (`playbook.yml`)**: El archivo principal que orquesta la ejecución. Contiene una lista de "jugadas" (Plays). Cada jugada asocia un grupo de servidores (de tu inventario) con una lista de tareas.
3. **Tareas (Tasks) y Módulos**: Cada tarea ejecuta un **Módulo** de Ansible. Por ejemplo:
   - `apt`: Para instalar y actualizar paquetes usando el gestor de paquetes de Debian/Ubuntu.
   - `shell`: Para ejecutar comandos directamente en la shell del servidor.
   - `get_url`: Para descargar archivos de internet de forma segura.

---

## Inventario del Clúster (`hosts`)
El inventario define las IPs privadas estáticas que Terraform asignó a las máquinas:

* **[gateway] (`10.110.110.2`)**: El contenedor LXC que actuará como proxy y balanceador de carga externo usando **Caddy**.
* **[apps] (`10.110.110.3`)**: La máquina virtual Ubuntu principal que ejecutará Docker Engine y liderará el clúster de **Docker Swarm**.
* **[database] (`10.120.120.10`)**: El LXC de datos aislado que albergará **PostgreSQL 16** y **Redis**.
* **[storage] (`10.110.110.4`)**: El LXC de almacenamiento privado que correrá **MinIO S3**.

---

## Playbook de Automatización (`playbook.yml`)

El playbook realiza las siguientes tareas de forma secuencial:

1. **En el Gateway**: Actualiza repositorios e instala `caddy` y `curl`.
2. **En la VM de Aplicaciones**: Instala los repositorios oficiales de Docker, descarga Docker Engine e inicializa el modo **Docker Swarm** apuntando a su IP privada.
3. **En la Base de Datos**: Instala Postgres 16 y Redis Server.
4. **En el Almacenamiento**: Descarga e instala de forma limpia el servidor **MinIO S3** a través de su paquete oficial `.deb`.

---

## Instrucciones de Ejecución

Sigue estos pasos en tu servidor de desarrollo para ejecutar la configuración:

### 1. Comprobar Conectividad (Ping)
Ansible utilizará tus llaves SSH locales para intentar conectar con cada host del inventario. Comprueba que responda en verde (`SUCCESS`):
```bash
ansible all -m ping -i hosts
```

### 2. Ejecutar la configuración completa
Para ejecutar el playbook y aplicar todas las instalaciones:
```bash
ansible-playbook -i hosts playbook.yml
```

### 3. Verificar el estado
Una vez finalizado, puedes conectar por SSH a cualquiera de los servidores para verificar que los servicios estén activos, por ejemplo en la base de datos:
```bash
ssh root@10.120.120.10 "systemctl status postgresql"
```
