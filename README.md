# SeguridadInformatica


burp suite

ssrf

muchas veces se puede bypaseaer el f2a si esta en una pagina distinta

stockreport.pl 381 29 en linea de comandos para acceder al stock. productId=1&storeId=1|whoami si se ejecuta desde terminal eso se pasa facil

Purpose of command	Linux	Windows

Name of current user	whoami	whoami

Operating system	uname -a	ver

Network configuration	ifconfig	ipconfig /all

Network connections	netstat -an	netstat -an

Running processes	ps -ef	tasklist

How to detect SQL injection vulnerabilities
You can detect SQL injection manually using a systematic set of tests against every entry point in the application. To do this, you would typically submit:

The single quote character ' and look for errors or other anomalies.
Some SQL-specific syntax that evaluates to the base (original) value of the entry point, and to a different value, and look for systematic differences in the application responses.
Boolean conditions such as OR 1=1 and OR 1=2, and look for differences in the application's responses.
Payloads designed to trigger time delays when executed within a SQL query, and look for differences in the time taken to respond.
OAST payloads designed to trigger an out-of-band network interaction when executed within a SQL query, and monitor any resulting interactions.






jhon the riper

metasplot

nmap
