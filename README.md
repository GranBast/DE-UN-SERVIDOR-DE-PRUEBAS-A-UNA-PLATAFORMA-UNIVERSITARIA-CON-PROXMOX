# UNIFRANZ — Transformación de Infraestructura con Proxmox VE

> **De Servidor de Pruebas a Plataforma Universitaria Virtualizada**  



## 1. Resumen del Proyecto

Este repositorio contiene la documentación técnica, diagnósticos, guías de arquitectura y scripts de automatización para la transformación del servidor de virtualización del **Laboratorio 302** en la **Universidad Privada Franz Tamayo (UNIFRANZ - Sede Cochabamba)**.

Históricamente, el equipo operaba sin una estructura de gobernanza clara, lo que generó un desorden operativo severo y una condición de **almacenamiento crítico al 92.99% de capacidad real**. La propuesta consolida un diseño en **4 entornos aislados (Resource Pools)**, un plan de inversión de hardware de bajo costo y mecanismos de automatización vía la API REST de Proxmox.

---

##  2. Diagnóstico del Servidor (Laboratorio 302)

Auditoría realizada directamente vía CLI (`lscpu`, `sys_vendor`, `pvesm`, `qm list`) en el host `proxmox`:

| Componente / Parámetro | Especificación Real Auditada | Capacidad Máxima del Sistema | Estado Operativo |
| :--- | :--- | :--- | :--- |
| **Modelo de Equipo** | **HP Z820 Workstation** | Chasis Torre industrial 1125W | Operativo en excelente estado $24/7$. |
| **Procesador (CPU)** | **Intel(R) Xeon(R) CPU E5-2630 v2 @ 2.60GHz** (6C/12T) | Socket Dual LGA2011 (Hasta 2 CPUs / 24 hilos) | Carga media $< 5\%$. Instrucciones VT-x/EPT activas. |
| **Memoria RAM** | **32 GB DDR3 ECC Registered** | 16 Slots DIMM (Hasta 512 GB RAM ECC) | Memoria ECC activa (prevención de cuelgues por bit-flips). |
| **Almacenamiento Local** | **198.61 GiB (`local-lvm` Thin Pool)** | Múltiples bahías SAS/SATA + PCIe NVMe | ⚠️ **CRÍTICO: 92.99% ocupado (184.69 GiB consumidos)**. |
| **Hipervisor** | **Proxmox VE 9.2.2** (Kernel Linux 64-bit) | Clúster Corosync / Standalone | Modo Standalone actualizado a rama estable 9.x. |

>  **Alerta de Sobreaprovisionamiento (Overprovisioning):** Aunque la capacidad física real es de **198.61 GiB**, la suma de los discos virtuales asignados a las VMs (`VM 101` a `VM 115`) supera los **500 GiB**. Quedan menos de **14 GiB libres**, lo que genera un riesgo inminente de congelamiento del sistema de almacenamiento LVM-Thin.

---

##  3. Arquitectura de los 4 Entornos

Para garantizar gobernanza y evitar la interferencia entre clases y servicios de producción, se segmentó la plataforma mediante **Resource Pools** y etiquetas de red **VLAN 802.1Q**:

```mermaid
flowchart TD
    subgraph HOST["NODO PROXMOX VE (Laboratorio 302 - Host 'proxmox')"]
        HW["🖥️ HP Z820 Workstation | Intel Xeon E5-2630 v2<br/>6 vCPUs | 64 GB RAM ECC | 2.2 TB NVMe + 4 TB HDD"]
        VMBR0["🔌 CONMUTADOR VIRTUAL PRINCIPAL<br/>(Linux Bridge vmbr0)"]
        
        HW --- VMBR0

        subgraph VLANS["Aislamiento de Red (VLANs 802.1Q)"]
            V10["VLAN 10<br/>Producción"]
            V20["VLAN 20<br/>Laboratorios"]
            V30["VLAN 30<br/>Desarrollo"]
            V40["VLAN 40<br/>Pruebas"]
        end

        VMBR0 --> V10
        VMBR0 --> V20
        VMBR0 --> V30
        VMBR0 --> V40

        subgraph POOLS["Piscinas de Recursos (Resource Pools)"]
            subgraph P_PROD["PISCINA PRODUCCIÓN<br/>(CPU Weight: 100 | RAM: 8 GB | NVMe: 100 GB)"]
                VM201["• Web Portal (VM 201)"]
                VM202["• Bot Acad. (VM 202)"]
                VM203["• DB Prod. (VM 203)"]
            end

            subgraph P_LABS["PISCINA LABORATORIOS<br/>(CPU Weight: 80 | RAM: 16 GB | NVMe: 200 GB)"]
                DOC1["• Docente 1 (Sistemas)"]
                DOC2["• Docente 2 (Redes)"]
                DOC3["• Docente 3 (Bases de Datos)"]
                DOC4["• Docente 4 (IA)"]
            end

            subgraph P_DEV["PISCINA DESARROLLO<br/>(CPU Weight: 50 | RAM: 12 GB | NVMe: 150 GB)"]
                DEV1["• Staging Apps"]
                VM401["• DB Dev (VM 401)"]
                DEV3["• Git Runner"]
            end

            subgraph P_TEST["PISCINA PRUEBAS<br/>(CPU Weight: 20 | RAM: 8 GB | HDD: 100 GB)"]
                CT1["• LXC Temp 1"]
                CT2["• LXC Temp 2"]
                VMTEST["• Sandbox VM"]
            end
        end

        V10 --> P_PROD
        V20 --> P_LABS
        V30 --> P_DEV
        V40 --> P_TEST

        subgraph STORAGE["Sistema de Almacenamiento Híbrido (Storage)"]
            NVME[("💾 Storage NVMe (2 TB Enterprise)<br/>VMs de Producción, Labs y Desarrollo")]
            HDD[("💿 Storage HDD (4 TB Enterprise)<br/>Repositorio VZDump Backups, ISOs y Pruebas")]
        end

        P_PROD --> NVME
        P_LABS --> NVME
        P_DEV --> NVME
        P_TEST --> HDD
    end

    style HOST fill:#f8f9fa,stroke:#333,stroke-width:2px
    style VMBR0 fill:#d0e1fd,stroke:#1a56db,stroke-width:2px
    style P_PROD fill:#d1e7dd,stroke:#0f5132,stroke-width:1px
    style P_LABS fill:#cff4fc,stroke:#055160,stroke-width:1px
  ```
## 💰 4. Plan de Expansión de Hardware y Presupuesto

Para solventar la saturación de almacenamiento y permitir el despliegue multitarea, se justifica el siguiente requerimiento técnico:

| Componente | Especificación Técnica | Cantidad | Costo ($USD) | Costo (Bs.) | Justificación Técnica |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **Módulo RAM** | 32 GB DDR3 ECC Registered 1600MHz | 1 | $138.00 | Bs. 950.00 | Eleva la RAM a **64 GB**, evitando el swapping en laboratorios simultáneos. |
| **SSD NVMe + PCIe** | 2 TB Enterprise High-TBW + Adaptador PCIe x4 | 1 | $210.00 | Bs. 1,450.00 | Elimina el cuello de botella de `local-lvm` reemplazando los 200 GB saturados. |
| **Disco HDD 3.5"** | 4 TB SATA3 7200RPM Enterprise (WD Red / Seagate) | 1 | $130.00 | Bs. 900.00 | Se instala en bahía caddy de la Z820 para backups VZDump e ISOs. |
| **TOTAL ESTIMADO** | — | — | **$478.00** | **Bs. 3,300.00** | Inversión eficiente para extender la vida útil del servidor por 3 años. |

---

## ⚙️ 5. Conceptos y Operaciones Técnicas Clave

El proyecto fundamenta el uso de las herramientas de Proxmox VE bajo 4 ejes (*Necesidad, Aplicación, Comprobación y Limitaciones*):

* **KVM vs LXC:** Uso de KVM para sistemas aislados/Windows y LXC para servicios Linux ligeros (ej. CT Alpine ~30 MB RAM).
* **Clones Vinculados (Linked Clones):** Despliegue de entornos estudiantiles en $<5$ segundos consumiendo $<10$ MB de espacio inicial desde la plantilla base `VM 107`.
* **RBAC & Realms:** Separación de administradores (`pam` realm) y docentes/alumnos (`pve` realm) restrictivos a su respectivo Resource Pool.
* **Continuidad de Servicio:**
  * **Producción:** RTO $< 30$ min | RPO $= 24$ horas.
  * **Laboratorios:** RTO $< 2$ horas | RPO $= 7$ días.

---

## 📜 6. Guía de Estudio Rápida (Hito 3)

<details>
<summary><b>💬 Haz clic aquí para desplegar las 10 preguntas y respuestas clave de la evaluación</b></summary>

1. **¿Por qué Proxmox VE es un hipervisor Tipo 1 (Bare-Metal)?**  
   *Respuesta:* Se instala directamente sobre el hardware sin SO anfitrión comercial, utilizando un kernel Debian modificado con KVM/LXC para máximo rendimiento.
2. **Diferencia entre KVM y LXC:**  
   *Respuesta:* KVM virtualiza hardware completo con kernel propio; LXC comparte el kernel del host Proxmox, siendo más ligero pero limitado a Linux.
3. **Peligro del sobreaprovisionamiento en LVM-Thin:**  
   *Respuesta:* Al asignar más espacio virtual del disponible, si los datos reales llenan el 100% físico, el pool se congela y corrompe los sistemas de archivos.
4. **Snapshot vs Backup:**  
   *Respuesta:* Snapshot guarda el estado/delta en el mismo disco (reversión rápida); Backup es una copia comprimida independiente en otro medio físico.
5. **Ventaja de Linked Clones:**  
   *Respuesta:* Aprovisiona VMs en segundos reutilizando el disco base de una plantilla, ahorrando hasta un 90% de almacenamiento.
6. **Rol del Quorum en Corosync:**  
   *Respuesta:* Mantiene la mayoría de votos ($>50\%$) en un clúster para tomar decisiones de Alta Disponibilidad (HA) sin incurrir en *Split-Brain*. Requiere mínimo 3 nodos.
7. **Función de las VLANs (802.1Q):**  
   *Respuesta:* Etiqueta el tráfico en los conmutadores virtuales para aislar lógicamente la red a nivel de Capa 2 entre los entornos.
8. **Utilidad de la API REST:**  
   *Respuesta:* Permite automatizar por código (puerto 8006) la clonación masiva, el encendido/apagado y la purga de entornos temporales.
9. **¿Por qué no es posible la Migración en Vivo en 1 nodo?**  
   *Respuesta:* Requiere transferir el estado de CPU/RAM entre 2 o más hosts activos conectados a un almacenamiento compartido (SAN/NFS/Ceph).
10. **Realms PAM vs PVE:**  
    *Respuesta:* `pam` autentica contra el sistema operativo Linux (administradores); `pve` autentica internamente en Proxmox sin dar acceso al host (docentes/alumnos).
</details>

---

## 📂 7. Estructura del Repositorio

```text
.
├── README.md                      # Documentación principal del repositorio
├── docs/
│   ├── informe_tecnico_v7.pdf     # Informe técnico final formateado para impresión
│   ├── diagrama_arquitectura.puml # Código fuente PlantUML del diagrama
│   └── diagrama_arquitectura.mmd  # Código fuente Mermaid.js del diagrama
├── scripts/
│   ├── api_purga_lxc.py           # Tarea 1: Script API REST para purga de LXCs vencidos
│   └── api_deploy_estudiantes.sh  # Tarea 2: Script Bash/Cloud-Init para clonación masiva
└── references/
    └── auditoria_lab302.log       # Salida de comandos CLI de la auditoría inicial
