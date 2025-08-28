# Manual de Implementación y Arquitectura

**Versión 3.0**
**Fecha de Elaboración:** 27 de agosto de 2025

---

## 1. Introducción

Este documento es una guía técnica y precisa para la implementación del sistema de monitoreo. Está dirigido a ingenieros de sistemas, administradores de TI y personal técnico cualificado. El manual cubre la instalación desde cero en un entorno sin conexión a internet, la configuración de todos sus componentes y la resolución de problemas comunes.

## 2. Requisitos del Sistema

Asegúrese de que el entorno de producción cumple con los siguientes requisitos para una implementación exitosa.

### 2.1. Requisitos de Hardware (por cada nodo del clúster)

*   **CPU:** 4 cores (mínimo)
*   **RAM:** 8 GB (mínimo), 16 GB (recomendado)
*   **Almacenamiento:**
    *   Disco 1 (SO Proxmox): 120 GB SSD (mínimo)
    *   Disco 2 (Datos VMs): 500 GB SSD o superior (recomendado)
*   **Red:** 2 x 1 Gbps NIC (mínimo, una para gestión y una para tráfico de VMs/clúster)

### 2.2. Requisitos de Software

*   **Sistema Operativo Base:** Proxmox VE 8.x (se instala desde cero).
*   **Componentes de la Aplicación:**
    *   Docker Engine
    *   Docker Compose
    *   Python 3.9+
    *   PostgreSQL
    (Nota: Todos estos componentes se instalan y configuran automáticamente a través de los scripts de despliegue).

### 2.3. Requisitos de Red

*   Subred local con direccionamiento IP estático disponible para los nodos y la VM de la aplicación.
*   Acceso SSH entre los nodos del clúster.
*   Puertos requeridos:
    *   `8006` (TCP): para la interfaz web de Proxmox.
    *   `22` (TCP): para SSH.
    *   `8000` (TCP): para el acceso al dashboard de monitoreo (en la IP de la VM).

## 3. Arquitectura del Sistema

### 3.1. Visión General y Componentes

La arquitectura se basa en un clúster de alta disponibilidad (HA) de dos nodos Proxmox. La aplicación de monitoreo se ejecuta dentro de una máquina virtual (VM) que puede conmutar entre los nodos en caso de fallo.

`![Arquitectura General](uml/arquitectura.puml)`

*   **Clúster Proxmox:** Dos servidores físicos que actúan como uno solo, proporcionando redundancia.
*   **VM de Aplicación:** Una máquina virtual Debian que aloja la aplicación en contenedores Docker.
*   **Contenedores Docker:**
    *   `web`: La aplicación Django que sirve el dashboard.
    *   `db`: La base de datos PostgreSQL.
    *   `data_collector`: El script que recopila métricas de Proxmox.
*   **Alta Disponibilidad (HA):** Si el nodo activo falla, la VM se inicia automáticamente en el nodo pasivo.
*   **Replicación:** El disco de la VM se copia periódicamente al nodo pasivo para minimizar la pérdida de datos.

### 3.2. Flujo de Datos

`![Flujo de Despliegue](uml/despliegue.puml)`

1.  El **Colector de Datos** (en un contenedor) se conecta a la API de Proxmox.
2.  Recolecta métricas de los nodos, VMs y contenedores.
3.  Envía los datos a la **API de la aplicación web** (otro contenedor).
4.  La aplicación procesa y almacena los datos en la **base de datos PostgreSQL** (tercer contenedor).
5.  El **Dashboard** (servido por la app web) consulta la base de datos para mostrar la información al usuario.

## 4. Procedimiento de Instalación

Este procedimiento detalla la instalación completa en un entorno offline.

### Fase 1: Preparación de Medios (Entorno Online)

Esta fase se realiza en un equipo con acceso a internet para preparar todos los componentes necesarios.

1.  **Creación de ISOs de Proxmox Automatizados:**
    El primer paso es generar las imágenes de instalación desatendida para cada nodo del clúster.
    *   **Navegar al Directorio del Instalador:**
        ```bash
        cd tesis/instalador
        ```
    *   **Iniciar Contenedor de Build:** Se utiliza un contenedor para asegurar un entorno de compilación consistente.
        ```bash
        docker run --platform linux/amd64 -it --rm -v "$(pwd)":/project debian:12-slim /bin/bash
        ```
    *   **Ejecutar Script de Creación dentro del Contenedor:** Este bloque de comandos automatiza la generación de ambos ISOs.
        ```bash
        set -e
        apt-get update && apt-get install -y curl gpg xorriso
        curl -fsSL https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg | gpg --dearmor -o /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg
        echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
        apt-get update && apt-get install -y proxmox-auto-install-assistant
        cd /project
        
        echo "--- Creando ISO para Nodo 1 ---"
        proxmox-auto-install-assistant prepare-iso ./proxmox/proxmox-ve_8.4-1.iso --fetch-from iso --answer-file ./proxmox/archivos_config/answer-node1.toml
        mv ./proxmox/proxmox-ve_8.4-1-auto-from-iso.iso proxmox-node1-automated.iso

        echo "--- Creando ISO para Nodo 2 ---"
        proxmox-auto-install-assistant prepare-iso ./proxmox/proxmox-ve_8.4-1.iso --fetch-from iso --answer-file ./proxmox/archivos_config/answer-node2.toml
        mv ./proxmox/proxmox-ve_8.4-1-auto-from-iso.iso proxmox-node2-automated.iso
        
        echo "--- Proceso completado ---"
        exit
        ```
    *   **Resultado:** Encontrarás `proxmox-node1-automated.iso` y `proxmox-node2-automated.iso` en la carpeta `instalador`.

2.  **Creación de USB de Instalación:**
    *   Use el script `create_installer_usb.sh` para "flashear" cada ISO en un dispositivo USB. Siga las instrucciones del asistente en la terminal para seleccionar el archivo ISO y el disco USB de destino.

3.  **Preparación del Pendrive de Despliegue:**
    *   Copie la carpeta `despliegue` completa desde la raíz del proyecto a un segundo pendrive. Este dispositivo contendrá todos los artefactos necesarios para la configuración del clúster y la aplicación en el entorno offline.

### Fase 2: Instalación Física de Nodos

1.  **Instalar Nodo 1 y Nodo 2:** Arranque cada servidor físico con su respectivo USB de instalación. El proceso es totalmente automático.
2.  **Verificar Conectividad:** Asegúrese de que ambos nodos están en red y accesibles a través de sus interfaces web (`https://IP_NODO:8006`).

### Fase 3: Orquestación del Clúster y Aplicación

1.  **Conectar y Montar Pendrive de Despliegue:** Inserte el pendrive con la carpeta `despliegue` en el Nodo 1 y móntelo en `/mnt/usb_datos`.
2.  **Ejecutar Script Orquestador:**
    ```bash
    cd /mnt/usb_datos/scripts_despliegue/
    bash ./configure_cluster_mejorado.sh
    ```
    Este script automatiza la creación del clúster, la plantilla de VM, el despliegue con Terraform y la configuración de HA y replicación.

## 5. Configuración y Parametrización

La configuración principal del sistema se gestiona a través de los siguientes archivos:

*   **`despliegue/archivos_config/terraform/main.tf`:**
    *   Define la infraestructura de la VM.
    *   Parámetros clave: `name`, `node_name`, `vm_id`, `clone.vm_id`, `network`.
    *   Contiene la configuración de `cloud-init` que prepara la VM con Docker.

*   **`despliegue/archivos_config/.env`:**
    *   Archivo crítico que contiene las variables de entorno para la aplicación Docker.
    *   Parámetros clave: `SECRET_KEY`, `DEBUG`, `DATABASE_URL`, `ALLOWED_HOSTS`.
    *   **Importante:** Este archivo se copia a la VM y es leído por Docker Compose.

*   **`instalador/proxmox/archivos_config/answer-nodeX.toml`:**
    *   Archivos de respuesta para la instalación desatendida de Proxmox.
    *   Parámetros clave: `country`, `keyboard`, `timezone`, `password`, `network`.

## 6. Verificación de la Implementación (Smoke Tests)

Una vez que el script orquestador finaliza, realice estas pruebas para verificar que todo funciona.

1.  **Verificar Estado del Clúster:**
    *   **Acción:** Acceda a la interfaz web de Proxmox.
    *   **Resultado Esperado:** Ambos nodos (`pve-node1`, `pve-node2`) deben aparecer con un icono verde, indicando que están online y en quórum.

2.  **Verificar Estado de la VM y HA:**
    *   **Acción:** En la vista de árbol, localice la VM `vm-dashboard` (ID 100).
    *   **Resultado Esperado:** La VM debe estar en estado "running". En la pestaña "HA" del Datacenter, el recurso debe estar listado y activado.

3.  **Verificar Acceso al Dashboard:**
    *   **Acción:** Abra un navegador y vaya a la IP de la VM (ej. `http://192.168.1.100`).
    *   **Resultado Esperado:** Debe cargar la página de inicio de sesión del dashboard.

4.  **Verificar Servicios Internos:**
    *   **Acción:** Conéctese por SSH a la VM y ejecute `docker ps`.
    *   **Resultado Esperado:** Deben listarse tres contenedores (`web`, `db`, `data_collector`) con el estado "Up".

5.  **Verificar Flujo de Datos:**
    *   **Acción:** Inicie sesión en el dashboard.
    *   **Resultado Esperado:** Después de unos minutos, los gráficos del dashboard deben empezar a mostrar datos de CPU, RAM y disco. Esto confirma que el colector de datos está funcionando y comunicándose con la aplicación.

## 7. Resolución de Problemas Comunes (Troubleshooting)

*   **Problema:** El script `configure_cluster_mejorado.sh` falla.
    *   **Solución:** El script es idempotente. Revise el último mensaje de error. A menudo, los problemas se deben a conectividad de red entre los nodos o a que el pendrive no está montado correctamente. Corrija el problema y vuelva a ejecutar el script.

*   **Problema:** Terraform no puede aplicar el plan (`terraform apply` falla).
    *   **Solución:** Verifique que el usuario y el token de API de Terraform se crearon correctamente en Proxmox (Paso 4 del script). Asegúrese de que la plantilla de VM (ID 9000) existe y está disponible.

*   **Problema:** La VM se crea pero no obtiene una dirección IP.
    *   **Solución:** Verifique la configuración de red en `main.tf` (gateway, bridge). Asegúrese de que el servicio DHCP de su red funciona o que la IP estática asignada es correcta y está libre.

*   **Problema:** Se accede al dashboard, pero no se muestran datos.
    *   **Solución:**
        1.  Conéctese por SSH a la VM.
        2.  Revise los logs del colector: `docker logs municipal_project-data_collector-1`.
        3.  Busque errores de conexión a la API de Proxmox. Verifique que las credenciales y la URL de la API en el archivo `.env` dentro de la VM son correctas.
