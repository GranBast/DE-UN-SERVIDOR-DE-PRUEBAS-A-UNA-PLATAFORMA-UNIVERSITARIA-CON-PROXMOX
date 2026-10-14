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
    style P_DEV fill:#fff3cd,stroke:#664d03,stroke-width:1px
    style P_TEST fill:#f8d7da,stroke:#842029,stroke-width:1px
    style NVME fill:#e2e3e5,stroke:#41464b,stroke-width:2px
    style HDD fill:#e2e3e5,stroke:#41464b,stroke-width:2px
