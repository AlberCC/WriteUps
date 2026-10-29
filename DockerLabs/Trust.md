# Writeup: Máquina Trust (DockerLabs)

2026-10-29 · Autor: albercc

## Resumen ejecutivo

En el laboratorio Trust (contenedor Docker en 172.17.0.2) se obtuvo acceso como root en tres pasos. La web expuso el usuario `mario`, SSH aceptó una contraseña débil de una lista común y `sudo` permitía ejecutar `vim` como root sin restricciones. Riesgo global: **crítico**.

## Alcance y entorno

- **Objetivo**: 172.17.0.2 (contenedor Docker, red bridge `docker0`).
- **Puertos** (nmap):

| Puerto | Servicio | Versión |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH 9.2p1 Debian 2+deb12u10 (confirmado por banner) |
| 80/tcp | HTTP | Anunciado como "PHP cli server 5.5 or later" |

- **Observación**: la raíz `/` devuelve la página por defecto de Apache2 (10701 bytes), que también responde 200 a rutas inexistentes. Eso no encaja con el banner de nmap.

## Reconocimiento y enumeración web

1. **gobuster dir** con `common.txt` y extensiones `php,txt,bak,zip`. Primera ejecución inválida: el servidor devolvía 200 a cualquier ruta. Se repitió con `--exclude-length 10701`.
2. **gobuster dir** con `raft-small-words.txt` y extensiones `php,txt,bak,zip,html`, mismo filtro. Sin resultados nuevos.
3. **Resultado**: única ruta válida `/secret.php` (200, 927 bytes). Su salida muestra "Hola Mario" y "Esta web no se puede hackear", lo que filtra un nombre de usuario.
4. **ffuf** sobre parámetros GET (`burp-parameter-names.txt`, 6453 palabras, filtro 927 bytes): ningún parámetro cambió la respuesta.
5. **ffuf** sobre ficheros con backups y editores (`.bak`, `.old`, `~`, `.swp`, `.orig`): sin resultados.
6. **ffuf** de virtual hosts con `Host: FUZZ.trust.local` (dominio supuesto, sin validar): sin resultados.

## Acceso SSH

- **Herramienta**: hydra, módulo `ssh`, 4 hilos (`-t 4`), con `-f` para parar al primer acierto.
- **Usuario**: `mario`, tomado de la pista de la web.
- **Lista**: `10k-most-common.txt` de SecLists (lista de contraseñas comunes).
- **Resultado**: `mario` / `chocolate` válido en el puerto 22.
- **Acceso**: sesión SSH como `mario` en el host `116f3c50a59c`.

## Escalada de privilegios

- **Comprobación**: `sudo -l` como `mario` devuelve `(ALL) /usr/bin/vim`.
- **Explotación**: `sudo vim -c ':!/bin/sh'` abre una shell como root.
- **Verificación**: `id` devuelve `uid=0(root) gid=0(root) groups=0(root)`, y `whoami` devuelve `root`.

## Hallazgos y mitigaciones

| # | Hallazgo | Severidad | Mitigación |
| --- | --- | --- | --- |
| 1 | Usuario revelado en la web (`secret.php`) | Media | No mostrar nombres de usuario en la aplicación. |
| 2 | Contraseña débil y común para SSH (`mario`) | Alta | Cambiar la contraseña y usar solo autenticación por clave; limitar intentos con fail2ban o similar. |
| 3 | `sudo` permite `vim` sin restricciones | Crítica | Quitar `vim` de sudoers. Si hace falta editar como root, usar `sudoedit`. |
| 4 | Respuesta 200 a cualquier ruta (catch-all) | Baja | Devolver 404 en rutas inexistentes; revisar el banner de nmap frente a la página real. |

## Herramientas y comandos

- **nmap**: detección de servicios y versiones.
- **gobuster** `dir`: fuerza bruta de rutas con `--exclude-length 10701` para filtrar el catch-all.
- **ffuf**: fuzzing de parámetros GET, ficheros de backup y virtual hosts.
- **hydra**: validación de credenciales SSH con `-t 4 -f`.
- **ssh**: sesión interactiva como `mario`.
- **sudo** y **vim**: escalada mediante `:!/bin/sh` (véase GTFOBins).

### Fuentes

- Listas de SecLists y `dirb`, instaladas en Parrot OS.
- GTFOBins, para la técnica de escape de vim.
