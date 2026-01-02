# Configuracion de seguridad para servidores

Esta guia sirve para **aumentar la seguridad**, esto implica aplicara la autenticación robusta y  reducción de la superficie de ataque 

## Acceso remoto
El servicio de ssh es el modo por el cual nos conectamos a la maquina remotamanete, por ello es recomandable reforzar la seguridad para evitar el acceso no deseado al servidor

### Implementación de Autenticación de Doble Factor (2FA)

Asumiremos que usas **Ubuntu/Debian** (si usas CentOS/Rocky, avísame, los comandos cambian ligeramente).

#### Paso 1: Instalar el módulo en el servidor

Conéctate a tu servidor y ejecuta:

```bash
sudo apt update
sudo apt install libpam-google-authenticator

```

#### Paso 2: Generar el secreto (En el servidor)

Ejecuta la herramienta con tu usuario (no como root, a menos que entres como root):

```bash
google-authenticator
```

Te hará varias preguntas y mostrará un **Código QR** gigante en la terminal.

1. **Escanea el QR** con la app de autenticación de tu celular.
2. **Preguntas:**
* *Make tokens "time-based"?* -> **y**
* *Update the .google_authenticator file?* -> **y**
* *Disallow multiple uses of the same token?* -> **y** (Evita ataques de repetición).
* *Increase the time skew?* -> **n** (Más seguro) o **y** (Si el reloj de tu servidor no es muy preciso).
* *Enable rate-limiting?* -> **y** (Protege contra fuerza bruta).

> [\!IMPORTANT]
> Te dará unos "Emergency scratch codes". Cópialos y guárdalos en papel. Son la única forma de entrar si pierdes tu celular.

#### Paso 3: Configurar PAM (El portero del sistema)

Edita el archivo de reglas de autenticación:

```bash
sudo nano /etc/pam.d/sshd
```

Añade esta línea al final del archivo:

```text
auth required pam_google_authenticator.so
```

*(Opcional: Si quieres desactivar la contraseña del usuario y solo usar Llave + Código 2FA, busca la línea `@include common-auth` y pon un `#` al inicio para comentarla).*

### Configuración del Daemon SSH (El servicio)

Edita el archivo de configuración en el servidor. Aquí están los ajustes críticos para máxima seguridad. 

```bash
sudo nano /etc/ssh/sshd_config
```

```ini
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

* **Verifica la sintaxis:**
  Si no sale nada, está perfecto. Si sale error, corrige la línea que te indique.
  ```Bash
  sudo sshd -t
  ```

* **Reinicio del servicio:**
  ```Bash
  sudo systemctl restart sshd
  ```

> [\!IMPORTANT]
> Antes de aplicar estos cambios: Asegúrate de tener una sesión SSH abierta y activa. No cierres esa sesión hasta haber probado en una segunda terminal que puedes conectar con la nueva configuración. Si cometes un error, podrías bloquearte fuera de tu propio servidor.

### Protección Activa Fail2Ban

Al usar la autenticación por contraseña, estará expuesto a ataques de fuerza bruta (robots probando miles de claves por minuto). Fail2Ban detecta esto y bloquea la IP del atacante.

*   **Instalacion:**
    ```bash
    sudo apt update
    sudo apt install fail2ban -y
    ```
*   **Configuración (La regla de oro):**

    Fail2Ban trae un archivo de configuración por defecto llamado jail.conf. Nunca edites este archivo directamente, porque una actualización del sistema podría sobrescribirlo y borrar tus reglas.
    
    Crea un archivo `jail.local`:

    ```bash
    sudo nano /etc/fail2ban/jail.local
    ```

    Copia y pega el siguiente contenido en el archivo. He ajustado los valores para proteger tu acceso por contraseña.
    
    ```ini
    [DEFAULT]
    # Tiempo que la IP estará baneada (ej: 1h = 1 hora, 1d = 1 día)
    bantime = 1h
    
    # Ventana de tiempo para contar fallos (si falla X veces en 10 min -> BAN)
    findtime = 10m
    
    # Número máximo de intentos fallidos antes de banear
    maxretry = 5
    
    # No te banees a ti mismo (agrega la IP de tu casa/oficina si es fija)
    # Separa las IPs con espacio. 127.0.0.1 es el propio servidor.
    ignoreip = 172.16.8.208
    
    [sshd]
    enabled = true
    # IMPORTANTE: Si usas el puerto 2222, escribe: port = 2222
    # Si usas el puerto normal, escribe: port = ssh
    port    = 22
    logpath = %(sshd_log)s
    backend = systemd
    ```

> [\!NOTE]
> * **bantime:** 1 hora es buen inicio. Si ves ataques muy persistentes, cámbialo a `24h`.
> * **maxretry:** 5 intentos es razonable para escribir tu contraseña. Si pones 3, ten cuidado de no equivocarte tú mismo.

*   **Iniciar el servicio:**
    Activa Fail2Ban para que arranque siempre con el servidor y ejecútalo ahora:

    ```bash
    sudo systemctl enable fail2ban
    sudo systemctl start fail2ban
    ```

*   **Comandos útiles para el día a día:**
    Ahora que está corriendo, necesitas saber cómo gestionarlo.

    **Ver el estado general (cuántas cárceles/jails hay):**
    
    ```bash
    sudo fail2ban-client status
    ```
    ```bash
    Status
    |- Number of jail:      1
    `- Jail list:   sshd
    ```    
    
    **Ver el estado de SSH (cuántos baneados hay):**
    
    ```bash
    sudo fail2ban-client status sshd
    ```
    ```bash
    Status for the jail: sshd
    |- Filter
    |  |- Currently failed: 0
    |  |- Total failed:     5
    |  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
    `- Actions
       |- Currently banned: 1
       |- Total banned:     1
       `- Banned IP list:   10.212.135.243
    ```

    *Te mostrará "Currently banned" con la lista de IPs bloqueadas.*
    
    **DESBANEAR una IP (¡Muy importante!):**
    Si tú o un compañero se equivocan de contraseña y quedan bloqueados, entra desde otra IP (o desde la consola física) y ejecuta:
    
    ```bash
    sudo fail2ban-client set sshd unbanip <DIRECCION_IP>
    ```

> [\!TIP]
> **Un truco extra:** Si quieres ver "en vivo" cuando alguien intenta entrar y Fail2Ban lo bloquea, puedes usar este comando para leer los logs en tiempo real: tail -f /var/log/fail2ban.log











## Firewall

Al estar usando los servidores como nodos de kubernetes, para el uso del firewall se debe aplicar el principio de "Capas de Seguridad" (Defense in Depth):

1. **Capa 1 (UFW en el Host):** Protege al **Servidor Linux** (El sistema operativo). Aquí decides quién puede administrar la máquina (SSH).
2. **Capa 2 (NetworkPolicy en K8s):** Protege a **las Aplicaciones** (Tus servicios). Aquí decides quién puede consumir tus servicios (Bacula, Web, DB).

Aquí tienes el diagrama mental de cómo fluye el tráfico y cómo configurarlo paso a paso.

---

### Paso 1: Configurar UFW (Protegiendo el Host)

En UFW, tu objetivo es cerrar todo excepto el SSH y dejar que Kubernetes "respire" (los nodos necesitan hablar entre ellos).

**Análisis de tus Redes**
*  **Red de PODS (Datos reales):** 10.0.0.0/16 (Subred del CNI Cilium)

*  **Red de SERVICIOS (Dato previo):** 10.43.0.0/16 (Endpoints de RK2).

*  **Red FÍSICA:** 172.16.0.0/12 (Tu IP física es 172.16.9.180).

**Comandos a ejecutar (Ejecutar en TODOS los nodos):**

Debes permitir el tráfico desde la red de Pods y hacia ella, de lo contrario Cilium bloqueará los paquetes cuando crucen de un nodo a otro.
```bash
# 1. Permitir tráfico de la RED DE PODS (CRÍTICO: Aquí estaba la diferencia)
sudo ufw allow from 10.0.0.0/16 to any comment 'Cilium Pods CIDR Real'

# 2. Permitir tráfico de la RED DE SERVICIOS
sudo ufw allow from 10.43.0.0/16 to any comment 'K8s Services CIDR'

# 3. Permitir tráfico VXLAN de Cilium (Comunicación entre nodos)
sudo ufw allow 8472/udp comment 'Cilium VXLAN'

# 4. Permitir Salud de Cilium
sudo ufw allow 4240/tcp comment 'Cilium Health'

# 5. Permitir API de Kubernetes (Solo necesario en Masters, pero seguro en todos)
sudo ufw allow 6443/tcp comment 'K8s API'

# 6. Permitir registro de nodos RKE2
sudo ufw allow 9345/tcp comment 'RKE2 Node Registration'

# 7. Recargar para aplicar
sudo ufw reload
```

*  **Verificación Final:**
    Después de aplicar esto, verifica que Cilium no tenga errores de conectividad:
    ```bash
    kubectl -n kube-system exec -ti ds/cilium -- cilium status --verbose
    ```
    
**Resultado:** Tu servidor Linux está blindado. Solo entras tú por el 2222.

---

### Paso 2: Configurar NetworkPolicy (Protegiendo Bacula)

Ahora, aunque el tráfico llegue a Kubernetes, vamos a poner un "muro interno" para que **nadie** pueda hablar con el puerto 9101 (Bacula) excepto tú o tus clientes de backup autorizados.

**Requisito previo:**
Para que esto funcione, tu Kubernetes debe usar un CNI que soporte políticas (como **Calico**, **Cilium**, o **Canal**). Si usas K3s básico con Flannel puro, las políticas se ignoran (avísame si usas K3s).

**El Manifiesto (`bacula-firewall.yaml`):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: proteger-bacula
  namespace: default  # Asegúrate que Bacula esté en este namespace
spec:
  # 1. A quién aplico esta regla? (Busca los pods con label app: bacula-dir)
  podSelector:
    matchLabels:
      app: bacula-dir # <--- VERIFICA ESTE LABEL CON 'kubectl get pods --show-labels'
  
  # 2. Qué tipo de tráfico controlo?
  policyTypes:
  - Ingress
  
  # 3. Reglas de entrada (Whitelist)
  ingress:
  - from:
    # A) Permitir acceso desde tu IP de admin (ej. VPN o Casa)
    - ipBlock:
        cidr: 192.168.1.50/32
    
    # B) Permitir acceso desde los Clientes Bacula (los servidores que respaldas)
    - ipBlock:
        cidr: 172.16.9.100/32 # Ejemplo IP de un servidor cliente
    
    # C) Permitir tráfico interno del cluster (DNS, Monitorización)
    - namespaceSelector: {} 

    ports:
    - protocol: TCP
      port: 9101
    - protocol: TCP
      port: 9097

```

**Aplicar la regla:**

```bash
kubectl apply -f bacula-firewall.yaml

```

---

### ¿Cómo comprobar que funciona?

Esta es la belleza de separar las capas:

1. **Prueba SSH:** Intenta conectar por SSH al puerto 2222.
* *Debe funcionar.* (Gracias a UFW).


2. **Prueba Bacula (desde IP permitida):** Intenta conectar a la consola de Bacula.
* *Debe funcionar.* (La NetworkPolicy te deja pasar).


3. **Prueba Bacula (desde el celular/otra IP):** Intenta conectar al puerto 9101.
* *Debe fallar.* (La NetworkPolicy bloquea silenciosamente el paquete, aunque UFW lo haya dejado pasar hasta la puerta del cluster).



¿Sabes qué distribución de Kubernetes estás usando (K3s, RKE2, MicroK8s)? Esto es vital para confirmar si las *NetworkPolicies* funcionarán o si necesitas instalar un plugin extra.


## IDS (Snort)
