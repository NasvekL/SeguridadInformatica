# Seguridad Informática
- [Burp Suite](#burp-suite)
- [John the Ripper](#john-the-ripper)
- [Metasploit](#metasploit)
- [Nmap](#nmap)
- [Wireshark](#wireshark)
- [Sqlmap](#sqlmap)

Este documento recopila información y notas personales sobre diversas herramientas y técnicas en el campo de la seguridad informática.

## Burp Suite

Burp Suite es una herramienta esencial para el análisis de seguridad de aplicaciones web. Permite realizar pruebas manuales y automatizadas para identificar vulnerabilidades de seguridad.

### Path Traversal
Permite al atacante aceder a archivos que no deberia. Por ejemplo `https://insecure-website.com/loadImage?filename=../../../etc/passwd` permitiria acceder a `/var/www/images/../../../etc/passwd` que daria lugar a que se accediese en realidad a `/etc/passwd`.
### SSRF

- Se puede eludir la autenticación de dos factores (2FA) si está implementada en una página distinta.

### Comandos útiles
| Propósito del comando        | Linux    | Windows         |
|------------------------------|----------|-----------------|
| Nombre del usuario actual    | `whoami` | `whoami`        |
| Sistema operativo            | `uname -a` | `ver`          |
| Configuración de red         | `ifconfig` | `ipconfig /all` |
| Conexiones de red            | `netstat -an` | `netstat -an` |
| Procesos en ejecución        | `ps -ef` | `tasklist`      |

#### Bypassing 2FA
- Es posible eludir el 2FA cuando la verificación se realiza en una página distinta.

#### Uso de la línea de comandos para testing
- Ejemplo de comando: `stockreport.pl 381 29` en línea de comandos para acceder al stock.
  - Payload de ejemplo: `productId=1&storeId=1|whoami`. Si se ejecuta desde terminal, este payload puede ser procesado sin mayores restricciones.

### Detección de vulnerabilidades de inyección SQL

- Para detectar vulnerabilidades de inyección SQL manualmente, realiza un conjunto sistemático de pruebas contra cada punto de entrada en la aplicación.
- Sintaxis específica de SQL que evalúa al valor base (original) del punto de entrada, y a un valor diferente, y busca diferencias sistemáticas en las respuestas de la aplicación.
Condiciones booleanas tales como OR 1=1 y OR 1=2, y busca diferencias en las respuestas de la aplicación.
Cargas útiles diseñadas para provocar retrasos de tiempo cuando se ejecutan dentro de una consulta SQL, y busca diferencias en el tiempo que tarda en responder.
Cargas útiles OAST diseñadas para desencadenar una interacción de red fuera de banda cuando se ejecutan dentro de una consulta SQL, y monitorea cualquier interacción resultante.

#### Ejemplo de prueba de inyección SQL

GET /filter?category=Accessories' OR 1=1-- HTTP/2





## John the Ripper

(TBD - Información por agregar)

## Metasploit

(TBD - Información por agregar)

## Nmap

(TBD - Información por agregar)

## Wireshark

(TBD - Información por agregar)

## Sqlmap

(TBD - Información por agregar)

