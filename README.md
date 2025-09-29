# RKE2 Ansible Cluster

Playbook de Ansible para automatizar la creación y configuración de un cluster de Kubernetes usando RKE2.

## Tabla de Contenidos

- [Descripción](#descripción)
- [Requisitos del Sistema](#requisitos-del-sistema)
- [Estructura del Cluster](#estructura-del-cluster)
- [Configuración](#configuración)
- [Instalación](#instalación)
- [Uso](#uso)
- [Arquitectura](#arquitectura)
- [Verificación](#verificación)
- [Solución de Problemas](#solución-de-problemas)
- [Referencias](#referencias)

## Descripción

Este proyecto automatiza el despliegue de un cluster de Kubernetes seguro y conforme utilizando RKE2 (Rancher Kubernetes Engine 2). El playbook configura automáticamente:

- Nodo maestro principal con RKE2 server
- Nodos adicionales del plano de control para alta disponibilidad
- Nodos worker para ejecutar las cargas de trabajo
- Configuración de seguridad endurecida (CIS Benchmarks)
- Container Runtime integrado (containerd)
- Políticas de red predeterminadas

## Requisitos del Sistema

### Hardware Mínimo

- **Nodo Maestro/Control**: 2 CPU, 4GB RAM, 20GB almacenamiento
- **Nodos Worker**: 2 CPU, 4GB RAM, 20GB almacenamiento
- Red privada entre todos los nodos

### Software

- Sistema operativo compatible (Ubuntu 18.04+, CentOS/RHEL 7+, SLES 15+)
- Python 3.6+ instalado en el nodo de control
- Acceso SSH a todos los nodos del cluster
- Privilegios sudo en todos los nodos

### Dependencias

- Ansible 2.9+

## Estructura del Cluster

El cluster se compone de tres tipos de nodos:

- **Master**: Nodo maestro principal que inicializa el cluster
- **Control-planes**: Nodos adicionales del plano de control para alta disponibilidad
- **Workers**: Nodos que ejecutan las cargas de trabajo de aplicaciones

## Configuración

### 1. Configurar el Inventario

Edita el archivo `inventory.yaml` con las direcciones IP de tus nodos:

```yaml
all:
  children:
    control-planes:
      hosts:
        control-plane1:
          ansible_host: 192.168.1.101
        control-plane2:
          ansible_host: 192.168.1.102
    master:
      hosts:
        master:
          ansible_host: 192.168.1.100
    workers:
      hosts:
        worker1:
          ansible_host: 192.168.1.111
        worker2:
          ansible_host: 192.168.1.112
        worker3:
          ansible_host: 192.168.1.113
  vars:
    rke2_load_balancer: 192.168.1.200   # IP del balanceador de carga
    rke2_version: v1.29.1+rke2r1        # Versión específica de RKE2
    token: "TuTokenSeguroAqui"          # Token para unir nodos al cluster
```

### 2. Configurar Autenticación SSH

Asegúrate de tener acceso SSH sin contraseña a todos los nodos:

```bash
eval "$(ssh-agent -s)"
ssh-add /ruta/a/tu/clave_privada
```

## Instalación

### 1. Clonar el Repositorio

```bash
git clone <url-del-repositorio>
cd rke2-ansible
```

### 2. Verificar Conectividad

```bash
ansible all -i inventory.yaml -m ping --user root
```

## Uso

### Despliegue Completo del Cluster

```bash
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --ask-become-pass
```

### Despliegue por Etapas

Puedes ejecutar etapas específicas usando tags:

```bash
# Solo preconfiguración
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --tags preconfig

# Solo configurar el nodo maestro
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --tags master

# Solo configurar planos de control
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --tags control-planes

# Solo configurar workers
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --tags workers
```

## Arquitectura

El playbook realiza las siguientes operaciones:

### Fase de Preconfiguración (todos los nodos)
- Configuración del hostname
- Actualización del archivo `/etc/hosts`
- Configuración de seguridad y endurecimiento del sistema
- Configuración de firewall y políticas de red
- Instalación de RKE2

### Fase del Nodo Maestro
- Inicialización del cluster RKE2 server
- Configuración de etcd embebido
- Generación de tokens para control planes y workers
- Configuración de certificados y PKI

### Fase de Control Planes
- Unión al cluster como nodos server adicionales
- Configuración de alta disponibilidad del etcd
- Replicación de configuración y certificados

### Fase de Workers
- Unión al cluster como nodos agent
- Configuración para ejecutar cargas de trabajo
- Aplicación de políticas de seguridad

## Verificación

### Verificar el Estado del Cluster

```bash
# Desde el nodo maestro
/var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
/var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get pods -A
/var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml cluster-info
```

### Obtener el Archivo kubeconfig

```bash
# Desde el nodo maestro
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
# Modificar la IP del servidor en el archivo config si es necesario
```

## Solución de Problemas

### Problemas Comunes

1. **Error de conectividad SSH**
   - Verificar que las claves SSH estén configuradas correctamente
   - Comprobar que el usuario tenga privilegios sudo

2. **Fallo en la instalación de RKE2**
   - Verificar conectividad a internet
   - Comprobar que el sistema cumple los requisitos mínimos
   - Verificar que no haya conflictos de firewall

3. **Nodos no se unen al cluster**
   - Verificar que los tokens no hayan expirado
   - Comprobar conectividad de red entre nodos
   - Revisar logs: `journalctl -u rke2-server` o `journalctl -u rke2-agent`

### Logs Útiles

```bash
# Ver logs del servidor RKE2
journalctl -u rke2-server -f

# Ver logs del agente RKE2
journalctl -u rke2-agent -f

# Estado del servicio
systemctl status rke2-server
systemctl status rke2-agent

# Logs de contenedores
crictl logs <container-id>
```

## Referencias

1. [Documentación oficial de RKE2](https://docs.rke2.io/)
2. [Instalación de RKE2](https://docs.rke2.io/install/quickstart/)
3. [Requisitos del Sistema](https://docs.rke2.io/install/requirements/)
4. [Configuración de Seguridad CIS](https://docs.rke2.io/security/cis_self_assessment/)
5. [Alta Disponibilidad](https://docs.rke2.io/install/ha/)
6. [Configuración Avanzada](https://docs.rke2.io/install/configuration/)
7. [Documentación de Ansible](https://docs.ansible.com/)
