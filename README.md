# OpenClawCPU
Installing OpenClaw in an old MacMini PC


# Guía Completa: Mac Mini 2011 → Servidor OpenClaw + Ollama

## Resumen del proyecto

**Hardware:** Mac Mini Mid 2011 · Core i5 2.3GHz · 16GB RAM · SSD 256GB  
**Objetivo:** Servidor local de IA con procesamiento secuencial (batch), controlado vía Telegram  
**OS elegido:** Debian 12 "Bookworm" Server (netinstall, sin escritorio)  
**Costo operativo:** $0 (sin API, sin VPS, todo local)

---

## ¿Por qué Debian 12 Minimal?

Se evaluaron varias opciones. Debian 12 netinstall (sin entorno gráfico) es la elección óptima por estas razones:

- **Consumo en reposo:** ~150–200MB RAM (vs ~400MB Ubuntu Server, vs ~800MB cualquier distro con escritorio)
- **Sin systemd bloat:** Instalación mínima, solo lo necesario
- **Compatibilidad probada:** Funciona en Mac Mini 2011 con EFI boot nativo
- **Node.js 22+ disponible:** Vía NodeSource, necesario para OpenClaw
- **Ollama soportado:** Binario x86_64 oficial para Linux
- **Estabilidad:** Ciclo de soporte largo, actualizaciones de seguridad sin sorpresas
- **Disco:** La instalación base ocupa ~2GB, dejando ~250GB libres para modelos y datos

**Descartados:** Ubuntu Server (más pesado, snaps innecesarios), Alpine (musl libc da problemas con Node.js y Ollama), Arch (rolling release = riesgo en servidor headless).

---

## FASE 1: Preparar el pendrive de instalación

### Qué necesitás

- Un pendrive USB de al menos 2GB (se borra todo su contenido)
- Acceso a otra computadora (o a la Mac Mini con Mountain Lion) para descargar la ISO y flashearla
- Cable Ethernet (el WiFi Broadcom necesita drivers no-libres, Ethernet funciona de entrada)
- Teclado USB y monitor/TV con HDMI o Thunderbolt/Mini DisplayPort

### Paso 1.1 — Descargar la ISO de Debian 12

Descargá la imagen **netinstall amd64** desde:

```
https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/
```

El archivo se llama algo como `debian-12.x.x-amd64-netinst.iso` (~600MB).

> **Importante:** Descargá la versión **netinst** (instalación por red), no la DVD completa. Es más liviana y descarga solo lo que necesitás durante la instalación.

### Paso 1.2 — Flashear el pendrive

**Desde macOS (Mountain Lion):**

```bash
# Identificar el disco del pendrive
diskutil list

# Desmontar (reemplazá diskN con el número correcto, ej: disk2)
diskutil unmountDisk /dev/diskN

# Flashear (usá rdiskN para velocidad, reemplazá la ruta de la ISO)
sudo dd if=~/Downloads/debian-12.x.x-amd64-netinst.iso of=/dev/rdiskN bs=1m

# Esperar a que termine (puede tardar unos minutos, no hay barra de progreso)
# Cuando termina, eyectar
diskutil eject /dev/diskN
```

**Desde Linux (otra PC):**

```bash
# Identificar el pendrive
lsblk

# Flashear (reemplazá /dev/sdX con el dispositivo correcto)
sudo dd if=debian-12.x.x-amd64-netinst.iso of=/dev/sdX bs=4M status=progress
sync
```

**Desde Windows:** Usá [Rufus](https://rufus.ie/) o [balenaEtcher](https://etcher.balena.io/). Seleccioná la ISO, seleccioná el pendrive, flasheá.

---

## FASE 2: Instalar Debian 12 en la Mac Mini

### Paso 2.1 — Bootear desde el pendrive

1. Conectá el cable Ethernet al router/switch
2. Insertá el pendrive USB
3. Conectá teclado y monitor
4. Encendé la Mac Mini manteniendo presionada la tecla **Option (⌥)** inmediatamente
5. Aparece el menú de boot de EFI. Seleccioná el disco USB (aparece como "EFI Boot" o similar)
6. En el menú de GRUB del instalador, elegí **"Install"** (modo texto, no graphical install)

> **Si la pantalla queda en negro:** En el menú de GRUB, presioná `e` para editar la entrada, buscá la línea que empieza con `linux` y agregá `nomodeset` antes de `---`. Luego presioná F10 para continuar.

### Paso 2.2 — Instalación paso a paso

Seguí el instalador de Debian con estas opciones:

1. **Idioma:** Español (o el que prefieras)
2. **País:** Argentina
3. **Teclado:** Latinoamericano (o el que uses)
4. **Hostname:** `macmini` (o el nombre que quieras darle en la red)
5. **Dominio:** Dejá en blanco
6. **Contraseña de root:** Ponele una contraseña fuerte. Anotala.
7. **Usuario:** Creá un usuario (ej: `peter`). Anotá la contraseña.
8. **Zona horaria:** Buenos Aires
9. **Particionado:**
   - Elegí **"Guiado - utilizar el disco entero"**
   - Seleccioná el SSD de 256GB
   - Elegí **"Todos los ficheros en una partición"** (más simple, y no necesitamos separar /home en un servidor)
   - Confirmá y escribí los cambios
10. **Mirror de Debian:** Argentina (o el más cercano)
11. **Proxy:** Dejá en blanco (a menos que tu red tenga proxy)
12. **Popularity-contest:** No
13. **Selección de software — ESTO ES CRÍTICO:**
    - **DESMARCÁ TODO** excepto:
      - ☑ SSH server
      - ☑ Standard system utilities
    - **NO** marques "Debian desktop environment", ni GNOME, ni nada con GUI
    - Esto es lo que hace que el sistema sea ultra-liviano
14. **GRUB bootloader:** Sí, instalarlo en el disco principal
15. Reiniciar y retirar el pendrive

### Paso 2.3 — Primer login

Después del reinicio, deberías ver el login de texto. Logueate con tu usuario.

> **Si la Mac Mini bootea macOS viejo en lugar de Debian:** Reiniciá con Option presionado y seleccioná el disco con GRUB/Debian. Si querés que sea permanente, dentro de macOS andá a Preferencias del Sistema > Disco de Arranque y seleccioná el disco de Debian.

---

## FASE 3: Configuración base del sistema

### Paso 3.1 — Acceso root y actualización

```bash
# Convertirse en root
su -

# Actualizar todo
apt update && apt upgrade -y

# Instalar herramientas esenciales
apt install -y sudo curl wget git htop nano net-tools ufw
```

### Paso 3.2 — Dar sudo al usuario

```bash
# Agregar tu usuario al grupo sudo
usermod -aG sudo peter

# Salir de root
exit

# Reloguearte como tu usuario para que tome efecto
exit
# (volver a loguearte)
```

### Paso 3.3 — Configurar IP estática (opcional pero recomendado)

Para poder conectarte siempre por SSH a la misma IP:

```bash
sudo nano /etc/network/interfaces
```

Reemplazá la configuración de la interfaz Ethernet (generalmente `enp1s0f0` o similar) con:

```
auto enp1s0f0
iface enp1s0f0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

> Ajustá la dirección IP, gateway y DNS según tu red local. Para ver el nombre de tu interfaz: `ip link show`

```bash
sudo systemctl restart networking
```

### Paso 3.4 — Configurar firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

### Paso 3.5 — Configurar SSH para acceso remoto

Desde este punto, podés desconectar monitor y teclado y manejar todo por SSH desde tu computadora principal:

```bash
# Desde tu otra computadora:
ssh peter@192.168.1.100
```

### Paso 3.6 — Desactivar servicios innecesarios

```bash
# Ver qué servicios están corriendo
systemctl list-units --type=service --state=running

# Desactivar los que no necesitamos
sudo systemctl disable --now avahi-daemon 2>/dev/null
sudo systemctl disable --now ModemManager 2>/dev/null
sudo systemctl disable --now bluetooth 2>/dev/null
sudo systemctl disable --now cups 2>/dev/null
```

### Paso 3.7 — Optimizar el uso de memoria (swap y swappiness)

Con 16GB de RAM y modelos de IA, queremos minimizar el uso de swap:

```bash
# Reducir la tendencia a usar swap (default es 60, lo ponemos en 10)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# Asegurar que hay un archivo de swap de respaldo (2GB es suficiente)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Hacer permanente
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Aplicar los cambios de sysctl
sudo sysctl -p
```

### Paso 3.8 — Verificar el estado del sistema

```bash
# RAM libre (deberías ver ~15.5GB libres)
free -h

# Disco (deberías ver ~250GB libres)
df -h /

# CPU
lscpu | grep "Model name"

# Temperatura (importante en Mac Mini)
# Instalar lm-sensors
sudo apt install -y lm-sensors
sudo sensors-detect --auto
sensors
```

---

## FASE 4: Instalar Ollama

### Paso 4.1 — Instalar Ollama

```bash
curl -fsSL https://ollama.ai/install.sh | sh
```

### Paso 4.2 — Habilitar el servicio

```bash
sudo systemctl enable ollama
sudo systemctl start ollama

# Verificar que está corriendo
sudo systemctl status ollama
```

### Paso 4.3 — Descargar modelos adecuados para tu hardware

Con 16GB de RAM y **sin GPU dedicada** (solo CPU), los modelos que funcionan bien son:

| Modelo | Tamaño en disco | RAM requerida | Velocidad aprox. | Uso recomendado |
|--------|----------------|---------------|-------------------|-----------------|
| `qwen2.5:3b` | ~2GB | ~4GB | ~5-8 tok/s | **Mejor equilibrio calidad/velocidad** |
| `llama3.2:3b` | ~2GB | ~4GB | ~5-8 tok/s | Buena alternativa general |
| `phi3:mini` | ~2.3GB | ~4GB | ~4-6 tok/s | Bueno para razonamiento |
| `gemma2:2b` | ~1.6GB | ~3GB | ~8-10 tok/s | El más rápido, menos capaz |
| `qwen2.5:7b` | ~4.7GB | ~8GB | ~2-3 tok/s | Más capaz, más lento |

**Recomendación:** Empezá con `qwen2.5:3b` como modelo principal (buen español, rápido en CPU) y opcionalmente `qwen2.5:7b` para tareas complejas:

```bash
# Modelo principal (rápido)
ollama pull qwen2.5:3b

# Modelo pesado para tareas que requieren más capacidad (opcional)
ollama pull qwen2.5:7b

# Verificar que funcionan
ollama run qwen2.5:3b "Decime hola en español"
```

> **Nota sobre rendimiento en CPU:** En tu i5 2.3GHz con 16GB, un modelo 3B genera ~5-8 tokens/segundo. Esto significa que una respuesta de 200 palabras tarda ~30-60 segundos. No es instantáneo, pero funciona. Para tareas batch (como querés) es perfectamente viable.

### Paso 4.4 — Configurar Ollama para optimizar memoria

```bash
# Crear override para el servicio de Ollama
sudo mkdir -p /etc/systemd/system/ollama.service.d

sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

Contenido:

```ini
[Service]
# Liberar el modelo de RAM después de 2 minutos de inactividad (default es 5min)
# Esto libera RAM para OpenClaw entre tareas
Environment="OLLAMA_KEEP_ALIVE=2m"

# Limitar a 1 request simultáneo (batch mode)
Environment="OLLAMA_NUM_PARALLEL=1"

# Usar 4 threads (tu i5 tiene 4 threads)
Environment="OLLAMA_NUM_THREAD=4"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

---

## FASE 5: Instalar Node.js 22+ (requerido para OpenClaw)

```bash
# Instalar Node.js 22 LTS vía NodeSource
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verificar versión
node --version   # Debería mostrar v22.x.x
npm --version

# Configurar npm para instalar paquetes globales sin sudo
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

---

## FASE 6: Instalar y configurar OpenClaw

### Paso 6.1 — Instalar OpenClaw

```bash
npm install -g openclaw@latest

# Verificar
openclaw --version
```

### Paso 6.2 — Onboarding inicial

```bash
openclaw onboard --install-daemon
```

Durante el onboarding:

1. **Mode:** Elegí `local` (no cloud)
2. **AI Provider:** Seleccioná `Ollama`
3. **Model:** Seleccioná `qwen2.5:3b` (o el que descargaste)
4. **Channel:** Seleccioná `Telegram` (lo configuramos en la próxima fase)
5. Si te pide bot token de Telegram, podés saltearlo por ahora y configurarlo después

### Paso 6.3 — Configurar OpenClaw para procesamiento secuencial (batch)

Este es el punto clave para tu caso de uso. Editá la configuración:

```bash
nano ~/.openclaw/openclaw.json
```

Configuración completa optimizada para tu hardware:

```json
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "GENERA_UN_TOKEN_SEGURO_AQUI"
    }
  },
  "agents": {
    "defaults": {
      "maxConcurrent": 1,
      "subagents": {
        "maxConcurrent": 1
      },
      "queue": {
        "mode": "collect"
      },
      "compaction": {
        "mode": "safeguard"
      },
      "model": {
        "primary": "ollama/qwen2.5:3b",
        "fallbacks": ["ollama/gemma2:2b"]
      }
    }
  },
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434",
        "apiKey": "ollama-local",
        "api": "ollama",
        "models": [
          {
            "id": "qwen2.5:3b",
            "name": "Qwen 2.5 3B",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 32768,
            "maxTokens": 4096
          },
          {
            "id": "qwen2.5:7b",
            "name": "Qwen 2.5 7B",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 32768,
            "maxTokens": 4096
          }
        ]
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "TU_BOT_TOKEN_AQUI",
      "dmPolicy": "pairing",
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    }
  }
}
```

### Explicación de la configuración clave:

**`maxConcurrent: 1`** — Este es el corazón de tu sistema batch. Significa que OpenClaw procesa **una sola tarea a la vez**. Cuando mandás un mensaje mientras hay una tarea en curso, se encola automáticamente.

**`queue.mode: "collect"`** — Los mensajes que lleguen mientras hay una tarea en ejecución se recolectan y se procesan todos juntos en el siguiente turno. Así funciona tu ejemplo:

1. Le pedís un deep research → empieza a procesarlo
2. A los 2 min le mandás un recordatorio → se encola
3. A los 5 min le mandás transcribir un video → se encola
4. Cuando termina el deep research, toma los 2 mensajes encolados y los procesa secuencialmente

**`OLLAMA_KEEP_ALIVE=2m`** — Después de 2 min sin uso, Ollama libera el modelo de RAM. Esto le da a OpenClaw más RAM para procesar la cola y gestión interna.

### Paso 6.4 — Generar un token de seguridad

```bash
# Generá un token aleatorio
openssl rand -hex 24
```

Copiá el resultado y ponelo en el campo `"token"` dentro de `gateway.auth` en el JSON.

---

## FASE 7: Configurar Telegram con BotFather

### Paso 7.1 — Crear el bot en Telegram

1. Abrí Telegram en tu celular o computadora
2. Buscá **@BotFather** (verificá que tenga la tilde azul de verificación)
3. Mandále `/newbot`
4. Seguí las instrucciones:
   - **Nombre del bot:** El que quieras (ej: "Mi Asistente IA")
   - **Username:** Debe terminar en `bot` o `_bot` (ej: `peter_macmini_bot`)
5. BotFather te va a dar un **token** con formato: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`
6. **Guardá este token de forma segura.** Es como una contraseña.

### Paso 7.2 — Obtener tu ID numérico de Telegram

1. En Telegram, buscá **@userinfobot**
2. Mandále `/start`
3. Te va a responder con tu ID numérico (ej: `123456789`). Anotalo.

### Paso 7.3 — Configurar BotFather opciones extras

Seguí en el chat con @BotFather:

```
/setdescription
→ Seleccioná tu bot
→ "Asistente IA personal corriendo en Mac Mini"

/setprivacy
→ Seleccioná tu bot
→ "Disable" (así puede ver mensajes en grupos si lo necesitás)
```

### Paso 7.4 — Poner el token en OpenClaw

```bash
nano ~/.openclaw/openclaw.json
```

Reemplazá `"TU_BOT_TOKEN_AQUI"` con el token real que te dio BotFather.

Opcionalmente, agregá la restricción de acceso por usuario:

```json
"channels": {
  "telegram": {
    "enabled": true,
    "botToken": "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz",
    "dmPolicy": "pairing",
    "allowFrom": [123456789],
    "groups": {
      "*": {
        "requireMention": true
      }
    }
  }
}
```

> Reemplazá `123456789` con tu ID numérico de Telegram. Esto asegura que solo vos podés hablar con el bot.

---

## FASE 8: Arrancar todo y verificar

### Paso 8.1 — Verificar que Ollama está corriendo

```bash
sudo systemctl status ollama
# Debería decir "active (running)"

# Test rápido
curl http://localhost:11434/api/tags
# Debería listar tus modelos
```

### Paso 8.2 — Verificar OpenClaw

```bash
# Diagnóstico
openclaw doctor --fix

# Ver modelos disponibles
openclaw models list
```

### Paso 8.3 — Arrancar el gateway

```bash
# Iniciar el gateway (primer plano para verificar que anda)
openclaw gateway

# Deberías ver algo como:
# ✓ Gateway started on port 18789
# ✓ Telegram channel started
```

Si todo funciona, cerralo con Ctrl+C y arrancalo como servicio:

```bash
# El daemon ya debería estar instalado desde el onboarding
openclaw gateway start
```

### Paso 8.4 — Pairing con Telegram

1. Abrí Telegram y buscá tu bot por su username
2. Mandále `/start` o "Hola"
3. El bot te va a responder con un **código de pairing**
4. En la terminal de la Mac Mini:

```bash
openclaw pairing approve telegram <CODIGO>
```

5. A partir de ahora, podés mandarle mensajes y te va a responder.

### Paso 8.5 — Test del sistema batch

Probá el flujo secuencial:

1. Mandá: "Escribime un resumen de 500 palabras sobre la historia de Argentina"
2. Mientras procesa (vas a ver el indicador de "escribiendo..."), mandá: "Recordame mañana comprar café"
3. Mandá también: "¿Cuál es la capital de Japón?"

Deberías ver que:
- Primero completa el resumen
- Luego procesa los otros dos mensajes que se encolaron

---

## FASE 9: Configurar arranque automático

### Paso 9.1 — Servicio systemd para el gateway

El onboarding de OpenClaw debería haber creado un servicio automáticamente. Verificá:

```bash
systemctl --user status openclaw-gateway
```

Si no existe, crealo manualmente:

```bash
mkdir -p ~/.config/systemd/user/

nano ~/.config/systemd/user/openclaw-gateway.service
```

Contenido:

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target ollama.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=%h/.npm-global/bin/openclaw gateway
Restart=always
RestartSec=10
Environment="NODE_ENV=production"
Environment="OLLAMA_API_KEY=ollama-local"

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable openclaw-gateway
systemctl --user start openclaw-gateway

# Habilitar linger para que el servicio corra sin login
sudo loginctl enable-linger peter
```

### Paso 9.2 — Verificar que todo arranca al reiniciar

```bash
sudo reboot
```

Esperá unos 2 minutos y después:

```bash
ssh peter@192.168.1.100

# Verificar servicios
sudo systemctl status ollama
systemctl --user status openclaw-gateway

# Mandar un mensaje desde Telegram para confirmar
```

---

## FASE 10: Optimizaciones finales

### 10.1 — Monitoreo de recursos

Creá un script simple para ver el estado:

```bash
nano ~/status.sh
```

```bash
#!/bin/bash
echo "=== SISTEMA ==="
echo "Uptime: $(uptime -p)"
echo ""
echo "=== MEMORIA ==="
free -h
echo ""
echo "=== CPU ==="
top -bn1 | head -5
echo ""
echo "=== TEMPERATURA ==="
sensors 2>/dev/null | grep -i temp || echo "lm-sensors no configurado"
echo ""
echo "=== DISCO ==="
df -h / | tail -1
echo ""
echo "=== OLLAMA ==="
systemctl is-active ollama
curl -s http://localhost:11434/api/tags | python3 -c "
import sys,json
try:
    d=json.load(sys.stdin)
    for m in d.get('models',[]): print(f'  {m[\"name\"]} ({m[\"size\"]/(1024**3):.1f}GB)')
except: print('  Error leyendo modelos')
"
echo ""
echo "=== OPENCLAW ==="
systemctl --user is-active openclaw-gateway 2>/dev/null || echo "no activo"
```

```bash
chmod +x ~/status.sh
```

### 10.2 — Limpieza automática de logs

```bash
# Limitar journald a 100MB
sudo nano /etc/systemd/journald.conf
# Descomentar y poner: SystemMaxUse=100M

sudo systemctl restart systemd-journald
```

### 10.3 — Desactivar lo que no usás

```bash
# Si no usás WiFi (estás con Ethernet)
sudo systemctl disable --now wpa_supplicant 2>/dev/null

# Desactivar IPv6 si no lo usás
echo 'net.ipv6.conf.all.disable_ipv6=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## Resumen de puertos y servicios

| Servicio | Puerto | Acceso |
|----------|--------|--------|
| SSH | 22 | Red local |
| Ollama API | 11434 | Solo localhost |
| OpenClaw Gateway | 18789 | Solo localhost |
| Telegram | Outbound | Long-polling (no requiere puerto entrante) |

---

## Presupuesto de RAM aproximado

| Componente | RAM estimada |
|------------|-------------|
| Debian 12 (base) | ~150MB |
| OpenClaw Gateway (Node.js) | ~200-400MB |
| Ollama (idle, modelo descargado) | ~100MB |
| Ollama (qwen2.5:3b cargado) | ~3-4GB |
| Ollama (qwen2.5:7b cargado) | ~6-8GB |
| **Buffer libre (caché del sistema)** | **~8-12GB** |

Con `OLLAMA_KEEP_ALIVE=2m`, el modelo se descarga de RAM 2 minutos después de la última consulta, liberando los 3-8GB para caché de disco y otras operaciones.

---

## Troubleshooting

**"El bot de Telegram no responde":**
```bash
# Ver logs del gateway
openclaw logs --follow
# o
journalctl --user -u openclaw-gateway -f
```

**"Ollama es muy lento":**
- Verificá que `OLLAMA_NUM_THREAD=4` está configurado
- Probá con un modelo más chico (`gemma2:2b`)
- Verificá temperatura: `sensors` — si supera 90°C, limpiar ventiladores

**"Se queda sin memoria":**
```bash
# Ver qué consume más
ps aux --sort=-%mem | head -10
# Reducir OLLAMA_KEEP_ALIVE a 1m
# Usar exclusivamente modelos 3B o menores
```

**"El gateway no arranca después del reboot":**
```bash
# Verificar que linger está habilitado
loginctl show-user peter | grep Linger
# Debería decir Linger=yes
# Si no:
sudo loginctl enable-linger peter
```

---

## Comandos útiles del día a día

```bash
# Estado general
~/status.sh

# Logs de OpenClaw en vivo
openclaw logs --follow

# Reiniciar el gateway
openclaw gateway restart

# Cambiar de modelo
openclaw models set ollama/qwen2.5:7b

# Actualizar OpenClaw
npm update -g openclaw@latest
openclaw gateway restart

# Ver cola de tareas (si hay verbose habilitado)
openclaw logs --follow | grep -i "queue\|enqueue\|lane"
```

---

## Siguiente nivel (ideas futuras)

- **Acceso remoto fuera de tu red:** Configurá Tailscale (VPN mesh gratuita) para acceder desde cualquier lado
- **Modelo más potente por API híbrido:** Podrías agregar OpenRouter como fallback para tareas pesadas (deep research) usando un modelo cloud barato, y dejar el 3B local para tareas rápidas
- **Más skills de OpenClaw:** Explorar la librería de skills para automatizaciones (cron jobs, web scraping, etc.)
- **Backup automático:** Cron job que haga backup de `~/.openclaw/` a un pendrive o nube

---

*Guía creada por Peter — Mac Mini Mid 2011, Buenos Aires, Argentina*  
*Fecha: Abril 2026*
