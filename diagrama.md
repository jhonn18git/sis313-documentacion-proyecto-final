# Diagrama de Arquitectura

**Proyecto 12: Disaster Recovery Active-Passive**

## Leyenda de estado

- 🟢 **Operativo:** componente probado y funcionando correctamente
- 🟡 **En configuracion:** componente parcialmente implementado
- ⚪ **Pendiente:** componente no implementado aun

## Diagrama general

```
┌──────────────────────────────────────────────────────────────────────┐
│                  RED ZEROTIER  10.163.244.0/24                       │
│                  Network ID: 633e31d8a2a83120                        │
│                                                                        │
│  ┌─────────────────────────────────┐      ┌─────────────────────┐  │
│  │         SITE A (Jhonn)           │      │   SITE A-DB (Camila) │  │
│  │                                   │      │                       │  │
│  │   VIP: 192.168.10.100  🟢        │      │   site-a-db  🟢      │  │
│  │                                   │      │   ZT: .251            │  │
│  │  ┌──────────────┐                │      │                       │  │
│  │  │ site-a-master│  🟢            │◄─────┤   MariaDB 10.11       │  │
│  │  │ 192.168.10.10│                │ mysql│   BD: empresa         │  │
│  │  │ Keepalived   │                │      │                       │  │
│  │  │ MASTER       │                │      │   RAID 5 (md127) 🟢   │  │
│  │  │ NGINX        │                │      │   /mnt/raid5          │  │
│  │  └──────┬───────┘                │      │   3 discos x 2GB      │  │
│  │         │ VRRP unicast           │      └───────────┬───────────┘  │
│  │         │ 192.168.10.0/24  🟢   │                  │              │
│  │  ┌──────┴───────┐                │                  │ rsync/dump   │
│  │  │site-a-backup │  🟢            │                  │ (backup.sh)  │
│  │  │ 192.168.10.11│                │                  │  🟢          │
│  │  │ Keepalived   │                │                  ▼              │
│  │  │ BACKUP       │                │      ┌───────────────────────┐ │
│  │  │ NGINX        │                │      │   SITE B (Jose Luis)  │ │
│  │  └──────────────┘                │      │                       │ │
│  │                                   │      │   site-b  🟢          │ │
│  └───────────────────────────────────┘      │   ZT: .239            │ │
│                                               │                       │ │
│                                               │   NGINX (standby) 🟢  │ │
│                                               │   MariaDB local  🟢   │ │
│                                               │   RAID 5 (md127) 🟢   │ │
│                                               │   /opt/backups   🟢   │ │
│                                               │   failover.sh    🟢   │ │
│                                               └───────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘

         Scripts de automatizacion (Sebastian)
         ──────────────────────────────────────
         backup.sh        🟢  cron */30 en site-a-master
         failover.sh      🟢  ejecucion manual en site-b
         health_check.sh  ⚪  pendiente
```

## Flujos de datos

| # | Flujo | Origen -> Destino | Descripcion | Estado |
|---|---|---|---|---|
| 1 | Keepalived VRRP | site-a-master <-> site-a-backup | Monitoreo de NGINX cada 2 seg via unicast. VIP migra en menos de 5 seg si el master cae. | 🟢 Operativo |
| 2 | Consulta MariaDB | site-a-master -> site-a-db | Conexion MySQL remota hacia 10.163.244.251:3306 con usuario backup_user. | 🟢 Operativo |
| 3 | Backup automatizado | site-a-master -> site-b | mysqldump + rsync cada 30 min hacia /opt/backups en site-b via ZeroTier y SSH sin contrasena. | 🟢 Operativo |
| 4 | Failover DR | site-b (activacion manual) | failover.sh restaura BD desde dump mas reciente, levanta NGINX. RTO medido en segundos. | 🟢 Operativo |

## Topologia fisica

```
Laptop 1 (Jhonn)              Laptop 2 (Camila)           Laptop 3 (Jose Luis)
┌─────────────────┐           ┌─────────────────┐         ┌─────────────────┐
│ VirtualBox       │           │ VirtualBox       │         │ VirtualBox       │
│ ┌──────────────┐ │           │ ┌──────────────┐ │         │ ┌──────────────┐ │
│ │site-a-master │ │           │ │ site-a-db    │ │         │ │  site-b      │ │
│ └──────────────┘ │           │ └──────────────┘ │         │ └──────────────┘ │
│ ┌──────────────┐ │           └────────┬─────────┘         └────────┬─────────┘
│ │site-a-backup │ │                    │                            │
│ └──────────────┘ │                    │                            │
└────────┬──────────┘                   │                            │
         │                              │                            │
         └──────────────────┬───────────┴────────────────────────────┘
                             │
                    Red ZeroTier (overlay VPN)
                    Funciona independientemente
                    de la red WiFi fisica usada
```
