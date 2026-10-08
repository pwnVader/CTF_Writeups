
# Whoami-Labs Laboratorio SUID (Easy)

---

## Resumen

El servidor web (Apache, puerto 8080) sirve una página de aprendizaje que esconde
credenciales en el HTML (`<span class="hidden">`). Con `student:skylar/99` entro por
SSH. Dentro no hay `sudo`, pero `/usr/bin/find` es **SUID root**: la clásica
combinación me deja un shell root.

---

## 1. Reconocimiento

Despliegue con `sudo bash startlab.sh suid.tar` → puertos mapeados al host:
**2224→22 (SSH)** y **8083→8080 (HTTP)**.

![[Pasted image 20261007191813.png]]

```bash
nmap -p- --min-rate 2000 -T4 172.17.0.2
# 22/tcp  open  ssh     (OpenSSH_8.9p1 Ubuntu)
# 8080/tcp open  http    (Apache/2.4.52 Ubuntu)
```

`/robots.txt` → 404. `/server-status` → 403. El sitio es una sola página
estática (`/var/www/html/index.html`) con material didáctico sobre SUID.

## 2. Acceso inicial credenciales ocultas en el HTML

El HTML contiene texto con la clase `hidden`, cuyo color (`#0a0a0f`) es idéntico
al fondo: invisible en el navegador, trivial con `curl`:

![[Pasted image 20261007191928.png]]

![[Pasted image 20261007191957.png]]

```bash
ssh student@172.17.0.2
# uid=1000(student) gid=1000(student)
```
## 3. Enumeración

```bash
find / -perm -4000 -type f 2>/dev/null
```

Binarios SUID inusuales para una distro estándar:

```
/usr/bin/find     ← GTFOBins
/usr/bin/cp       ← GTFOBins
/usr/bin/mv       ← GTFOBins
/usr/bin/su, /usr/bin/passwd, /usr/bin/mount ...  (normales)
```

`sudo` **no está instalado** (`sudo: command not found`) → descartado.
No hay cron, no hay binarios propios, el único flag en todo el FS es
`/root/flag.txt` (inaccesible como student). La web no tiene más rutas
interesantes. El vector es 100% SUID.

![[Pasted image 20261007192153.png]]

## 4. Escalada de privilegios

![[Pasted image 20261007192325.png]]

![[Pasted image 20261007192242.png]]

![[Pasted image 20261007192403.png]]

## 5. Flag

![[Pasted image 20261007192648.png]]

```
R****7
```

![[Pasted image 20261007192710.png]]
## 6. Mitigaciones

- No dejar `SUID` en binarios que no lo necesitan: `chmod u-s /usr/bin/find /usr/bin/cp /usr/bin/mv`.
- Auditar con `find / -perm -4000 -type f 2>/dev/null`.
- No esconder credenciales con CSS el HTML viaja entero al cliente.
- Passwords débiles + reutilización (`skylar/99`).

## Marco MITRE ATT&CK

El laboratorio anuncia **T1574.002 (DLL/Shared Library Hijacking)**, pero el
vector real del exploit es **T1548.001 (Abuse Elevation Control Mechanism:
Setuid and Setgid)** combinado con **T1036.001 (Masquerading: Inline Code
Generation)** para el binario SUID generado. T1574.002 no aplica: no hay
hijacking de librerías en esta escena.