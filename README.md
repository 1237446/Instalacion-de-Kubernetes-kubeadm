# Configuracion de seguridad para servidores

Esta guia sirve para **aumentar la seguridad**, esto implica aplicara la autenticación robusta y  reducción de la superficie de ataque 

## SSH
El servicio de ssh es el modo por el cual nos conectamos a la maquina remotamanete, por ello es recomandable reforzar la seguridad para evitar el acceso no deseado al servidor

> [\!IMPORTANT]
> Antes de aplicar estos cambios: Asegúrate de tener una sesión SSH abierta y activa. No cierres esa sesión hasta haber probado en una segunda terminal que puedes conectar con la nueva configuración. Si cometes un error, podrías bloquearte fuera de tu propio servidor.

### 1. Configuración del Daemon (`/etc/ssh/sshd_config`)

Edita el archivo de configuración en el servidor. Aquí están los ajustes críticos para máxima seguridad.

```ssh
# Include configuration from conf.d
Include /etc/ssh/sshd_config.d/*.conf

# 1. RED Y PROTOCOLO
Port 22
AddressFamily inet
#ListenAddress 0.0.0.0

# 2. LOGS Y CIFRADO
SyslogFacility AUTH
LogLevel VERBOSE

# Criptografía Robusta (Algoritmos modernos solamente)
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group14-sha256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com

# 3. AUTENTICACIÓN (CORE)
LoginGraceTime 30s
PermitRootLogin no
StrictModes yes
MaxAuthTries 3
MaxSessions 2

PubkeyAuthentication no

# Archivos de llaves autorizadas
AuthorizedKeysFile .ssh/authorized_keys

# Deshabilitar métodos inseguros
HostbasedAuthentication no
IgnoreRhosts yes
PasswordAuthentication yes
PermitEmptyPasswords no

# Habilitar el sistema de desafío-respuesta
ChallengeResponseAuthentication yes

# 4. CONFIGURACIÓN 2FA (DOBLE FACTOR)
# En versiones nuevas, 'ChallengeResponse' se llama 'KbdInteractiveAuthentication'
KbdInteractiveAuthentication yes
UsePAM yes

# LA REGLA MAESTRA DE SEGURIDAD
# Obliga a tener: 1. Llave SSH  Y ADEMÁS  2. Código del celular
# AuthenticationMethods publickey,keyboard-interactive

# 5. CONTROL DE ACCESO
AllowUsers oti

# 6. SEGURIDAD DE SESIÓN Y TUNELIZADO
AllowAgentForwarding no
AllowTcpForwarding no
GatewayPorts no
X11Forwarding no
PrintMotd no
TCPKeepAlive yes

# Detectar y matar conexiones muertas o zombies
ClientAliveInterval 300
ClientAliveCountMax 0

# Opciones varias
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
```

Pasos finales obligatorios:
Edita PAM: Asegúrate de que /etc/pam.d/sshd tenga auth required pam_google_authenticator.so.

Verifica la sintaxis:

Bash

sudo sshd -t
(Si no sale nada, está perfecto. Si sale error, corrige la línea que te indique).

Reinicia:

Bash

sudo systemctl restart sshd

























## Fail2Ban

## UFW



## Snort

Configurar un servidor SSH para **máxima seguridad** (hardening) implica aplicar el principio de "defensa en profundidad": autenticación robusta, reducción de la superficie de ataque y criptografía moderna.

Aquí tienes una guía paso a paso, desde lo esencial hasta lo avanzado.

---

### ⚠️ Advertencia Crítica

**Antes de aplicar estos cambios:** Asegúrate de tener una sesión SSH abierta y activa. **No cierres esa sesión** hasta haber probado en una **segunda terminal** que puedes conectar con la nueva configuración. Si cometes un error, podrías bloquearte fuera de tu propio servidor.

---

### 1. Generación de Llaves SSH Modernas (Cliente)

Olvídate de las contraseñas y de las llaves RSA antiguas. Usa el algoritmo **Ed25519**, que es más seguro y rápido.

En tu máquina local (tu PC, no el servidor):

```bash
ssh-keygen -t ed25519 -C "admin@tudominio.com" -a 100

```

* **Nota:** Cuando te pida una `passphrase`, **pon una contraseña fuerte**. Esto protege la llave privada en caso de que roben tu PC.
* Copia la llave al servidor: `ssh-copy-id -i ~/.ssh/id_ed25519.pub usuario@ip-servidor`

---

---

### 3. Autenticación de Doble Factor (2FA)

Para "máxima seguridad", una llave SSH no es suficiente (alguien podría robar tu laptop desbloqueada). Agrega Google Authenticator.

1. Instala el módulo: `sudo apt install libpam-google-authenticator` (Debian/Ubuntu) o `dnf install google-authenticator` (RHEL/CentOS).
2. Ejecuta `google-authenticator` con tu usuario y escanea el QR.
3. Edita `/etc/pam.d/sshd`:
Agrega al final: `auth required pam_google_authenticator.so`
Comenta: `@include common-auth` (para desactivar pedir contraseña de sistema, si solo quieres Llave + Código 2FA).
4. En `sshd_config`, cambia:
```ssh
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive

```


*Esto obliga a tener la llave Y el código del celular.*

---

### 4. Protección Activa (Fail2Ban o CrowdSec)

Incluso si no pueden entrar, los bots consumirán recursos intentándolo.

* **Instalar Fail2Ban:**
Crea un archivo `/etc/fail2ban/jail.local`:
```ini
[sshd]
enabled = true
port    = 2222
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
bantime = 1h

```


Esto bloqueará la IP del atacante en el firewall tras 3 intentos fallidos.

---

### 5. Nivel "Paranoico" (Opcional pero recomendado)

1. **Port Knocking:** El puerto SSH se mantiene cerrado en el firewall hasta que envíes una secuencia específica de paquetes a otros puertos "secretos".
2. **Llaves FIDO2/U2F:** Si tienes una YubiKey, puedes generar llaves que **requieran** que toques el hardware físico para conectar:
```bash
ssh-keygen -t ed25519-sk

```


3. **Restricción por IP:** Si siempre te conectas desde una IP fija (oficina/VPN), restringe el acceso en `sshd_config` usando `Match Address` o directamente en el firewall (UFW/IPTables).

---

### Resumen de Pasos de Verificación

1. Generar llaves Ed25519 con passphrase.
2. Copiar llave pública al servidor.
3. Editar `/etc/ssh/sshd_config` (Puerto, Protocolo, Cifrado, Sin Root, Sin Passwords).
4. Validar sintaxis: `sudo sshd -t` (Si esto da error, no reinicies).
5. Reiniciar servicio: `sudo systemctl reload sshd`.
6. **Probar conexión en una terminal nueva.**

¿Te gustaría que profundice en cómo configurar **Fail2Ban** para que los baneos sean permanentes o prefieres ver la configuración del **2FA**?
