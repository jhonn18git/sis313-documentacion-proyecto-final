# Bitacora de Avance

**Proyecto 12: Disaster Recovery Active-Passive**
**SIS313 - Infraestructura de Plataformas**

---

## Entrada 1

- **Fecha:** 27/05/2026
- **Responsable:** Jhonn Llanos
- **Actividad:** Seleccion del proyecto y configuracion de ZeroTier
- **Detalle:** Se eligio el Proyecto 12 como proyecto final del semestre. Se creo la red ZeroTier (ID: 633e31d8a2a83120) para conectividad remota entre las laptops del grupo. Se instalo ZeroTier en los equipos Windows de todos los integrantes y se aprobaron los dispositivos desde el panel de administracion. Se definio la arquitectura: site-a-master y site-a-backup a cargo de Jhonn, site-a-db a cargo de Camila, site-b a cargo de Jose Luis.
- **Dificultad superada:** Inicialmente dos integrantes aparecian con la misma IP ZeroTier. Se resolvio pidiendo a cada uno que ejecutara `ipconfig` individualmente para confirmar su IP real.
- **Estado:** Completado

---

## Entrada 2

- **Fecha:** 27/05/2026
- **Responsable:** Jhonn Llanos
- **Actividad:** Creacion de VMs y configuracion de red en Site A
- **Detalle:** Se crearon site-a-master y site-a-backup en VirtualBox con Ubuntu Server 24.04 LTS (1 GB RAM, 20 GB disco). Se configuro reenvio de puertos SSH para acceso desde Warp terminal. Se asignaron IPs estaticas via netplan en la red interna intnet-siteA: 192.168.10.10 para master y 192.168.10.11 para backup.
- **Dificultad superada:** Ninguna dificultad relevante en esta etapa.
- **Estado:** Completado

---

## Entrada 3

- **Fecha:** 27/05/2026
- **Responsable:** Jhonn Llanos
- **Actividad:** Instalacion de Keepalived y prueba de failover local
- **Detalle:** Se instalo Keepalived y NGINX en ambos nodos. Se configuro site-a-master como MASTER (prioridad 101) y site-a-backup como BACKUP (prioridad 100) con VIP 192.168.10.100. Se agrego vrrp_script para monitorear el estado de NGINX. Se realizo prueba de failover detectando que la VIP migraba a ambas VMs simultaneamente (split-brain) debido al uso de multicast en VirtualBox.
- **Dificultad superada:** Se reconfiguro Keepalived utilizando `unicast_src_ip` y `unicast_peer` en lugar de multicast, resolviendo el problema de split-brain. Se ajusto tambien el `weight` del vrrp_script a -60 para forzar correctamente la cesion del rol MASTER.
- **Estado:** Completado

---

## Entrada 4

- **Fecha:** 27/05/2026
- **Responsable:** Camila Montecinos
- **Actividad:** Configuracion de RAID 5 y MariaDB en site-a-db
- **Detalle:** Se creo la VM site-a-db con 3 discos virtuales adicionales de 2 GB. Se instalo mdadm y se creo RAID 5 con /dev/sdb, /dev/sdc y /dev/sdd. El arreglo fue formateado con ext4 y montado en /mnt/raid5. Se instalo MariaDB, se creo la base de datos 'empresa' con tabla 'empleados' y se creo el usuario backup_user con acceso remoto.
- **Dificultad superada:** Al reiniciar la VM, el RAID no monto por una entrada incorrecta en /etc/fstab (el RAID se nombro automaticamente md127 en lugar de md0). Se resolvio editando el fstab desde el modo mantenimiento del sistema.
- **Estado:** Completado

---

## Entrada 5

- **Fecha:** 28/05/2026
- **Responsable:** Jose Luis Maldonado
- **Actividad:** Configuracion de site-b (Sitio Pasivo de Recuperacion)
- **Detalle:** Se configuro la VM site-b con RAID 5, MariaDB, NGINX y ZeroTier. Se creo la base de datos 'empresa' local para recibir restauraciones. Se creo el directorio /opt/backups para los dumps remotos. Se creo el script /opt/scripts/failover.sh que restaura la BD y levanta NGINX registrando el RTO.
- **Dificultad superada:** Mismo problema de fstab con md127 que en site-a-db. Resuelto desde modo mantenimiento.
- **Estado:** Completado

---

## Entrada 6

- **Fecha:** 17/06/2026
- **Responsable:** Jhonn Llanos, Sebastian Aillon
- **Actividad:** Implementacion de backup.sh y prueba completa de disaster recovery
- **Detalle:** Se configuro autenticacion SSH sin contrasena entre site-a-master (usuario jhonn y root) y site-b para permitir la ejecucion automatica de backup.sh via cron. Se desarrollo el script backup.sh que ejecuta mysqldump remoto contra site-a-db y transfiere el archivo via rsync hacia site-b. Se configuro un cron job para ejecutar el backup cada 30 minutos. Se ejecuto manualmente failover.sh en site-b, restaurando exitosamente los datos y verificando su integridad.
- **Dificultad superada:** El script fallaba al ejecutarse con sudo porque la llave SSH se habia generado para el usuario normal y no para root, que es el usuario que ejecuta el cron. Se genero una segunda llave SSH para el usuario root y se copio hacia site-b.
- **Estado:** Completado

---

## Pendientes

- Script health_check.sh para monitoreo automatizado de servicios (RAID, MariaDB, NGINX)
- Reduccion del tiempo de cron a modo de prueba para demostraciones en clase
- Ensayo final cronometrado de la defensa grupal
