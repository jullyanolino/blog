# Windows DHCP and DNS logs

Cuando intentamos incorporar logs de servidores DHCP y DNS basados en Windows, es importante saber la ubicación de estos. NOrmalmente se encuentran en sobre %SYSTEMROOT%\System32\dns\dns.log y %SYSTEMROOT%\System32\dhcp\DhcpSrvLog-xxx.log. No obstante esto puede cambiar de acuerdo a la configuración de cada cliente y en el caso del servidor DHCP el nombre del log  depende incluso del idioma del servidor.

Para saber donde tenemos que buscar los logs y configurar correctamente nuestros parses deberemos atender a las configuraciones especificas de cada servidor.

### DNSServer

La configuración del servidor DNS se guarda sobre la clave de registro  `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\DNS`
Y la información que a nosotros nos interesa está sobre la clave `Parameters`

![](/assets/img/winlogs/dns-server-log-config.png)

Los parámetros que nos interesan desde nuestra perspectiva son:
* LogFilePath: Controla la ubicación de nuestros logs y que deberemos usar en nuestra herramienta de ingesta: filebeat, nxlog...
* LogFileMaxSize: Tamaño al que empezarán a rotar los logs
* LogLevel: Información que se mostrará en los logs
* EventLogLevel: Configuracón de eventos de Windows.
  * 0 = Ninguno (000)
  * 1 = Errores (001)
  * 2 = Errores y Advertencias (010)
  * 7 = Todos (111)

Si quisieramos personalizar los parseadores de logs en función de los parámetros establecidos en el DNSServer, deberemos comprobar el parámetro *LogLevel*. A nivel binario cada posición habilitará un tipo de registro concreto:

Posibles parámetros a configurar:

* Registrar paquetes de respuesta entrantes sin coincidencia (bit 26): Esta opción es interesante, ya que aunque el dominio no exista quedará registrado en los logs.
* Detalles (bit 25): Registrara en varias líneas la información casi a nivel de paquete UDP/TCP.
* TCP (bit 16): Peticiones hechas con el protocolo TCP.
* UDP (bit 15): Peticiones hechas con el protocolo UDP.
* Entrante (bit 14): Registrar paquetes entrantes.
* Salientes (bit 13): Registrar paquetes salientes.
* Respuesta (bit 10): Registrar respuestas del servidor.
* Solicitud (bit 9): Registrar solicitudes al servidor.
* Actualizaciones (bit 6): Registrar actualizaciones de dominios. Un  equipo notifica que un registro DNS ha cambiado, por ejemplo pòr un cambio de hostname.
* Notificaciones (bit 5): Notificaciones de servidor a clientes.
* Consultas/Transferencias (bit 1): Consultas de registros DNS.

##### Aclaración sobre la opción detalles
```
22/12/2021 17:32:59 0E1C PACKET  0000017DED1D2C90 UDP Snd ::1             0003 R Q [8085 A DR  NOERROR] AAAA   (7)wrk10w1(9)cancamusa(3)com(0)
UDP response info at 0000017DED1D2C90
  Socket = 696
  Remote addr ::1, port 57613
  Time Query=4199, Queued=0, Expire=0
  Buf length = 0x0200 (512)
  Msg length = 0x005d (93)
  Message:
    XID       0x0003
    Flags     0x8580
      QR        1 (RESPONSE)
      OPCODE    0 (QUERY)
      AA        1
      TC        0
      RD        1
      RA        1
      Z         0
      CD        0
      AD        0
      RCODE     0 (NOERROR)
    QCOUNT    1
    ACOUNT    0
    NSCOUNT   1
    ARCOUNT   0
    QUESTION SECTION:
    Offset = 0x000c, RR count = 0
    Name      "(7)wrk10w1(9)cancamusa(3)com(0)"
      QTYPE   AAAA (28)
      QCLASS  1
    ANSWER SECTION:
      empty
    AUTHORITY SECTION:
    Offset = 0x0027, RR count = 0
    Name      "[C014](9)cancamusa(3)com(0)"
      TYPE   SOA  (6)
      CLASS  1
      TTL    3600
      DLEN   42
      DATA   
                PrimaryServer: (6)srvdc1[C014](9)cancamusa(3)com(0)
                Administrator: (10)hostmaster[C014](9)cancamusa(3)com(0)
                SerialNo     = 40
                Refresh      = 900
                Retry        = 600
                Expire       = 86400
                MinimumTTL   = 3600
    ADDITIONAL SECTION:
      empty
```

Frente a:
```
22/12/2021 17:34:34 0E1C PACKET  0000017DEF873CC0 UDP Snd ::1             0002 R Q [8385 A DR NXDOMAIN] A      (6)wrk7w1(9)cancamusa(3)com(0)
```
Aunque con esta opción si es posible ver la respuesta que le da el servidor al cliente dentro de *ANSWER SECTION*.

### Ejemplos de campo LogLevel


RegistrarPaquetes(26), Detalles(25), TCP(16), UDP(15), Entrante(14), Saliente(13), Respuesta(10), Solicitud(9),Actualizaciones(6), Notificaciones(5), Consultas/Transfer(1)
* 00020737 ->00000000000101000100000001-> Saliente, UDP, Consultas/Transfer, Solicitud
* 00024833 ->00000000000110000100000001-> Entrante, UDP, Consultas/Transfer, Solicitud
* 00037121 ->00000000001001000100000001-> Saliente, TCP, Consultas/Transfer, Solicitud
* 00061697 ->00000000001111000100000001-> SalienteyEntrante, TCPyUDP, Consultas/Transfer, Solicitud
* 00061984 ->00000000001111001000100000-> SalienteyEntrante, TCPyUDP, Actualizaciones, Respuesta
* 00061968 ->00000000001111001000010000-> SalienteyEntrante, TCPyUDP, Notificaciones, Respuesta
* 33616689 ->10000000001111001100110001-> RegistrarPaquetes, SalienteyEntrante, TCPyUDP, Consultas/TransferyActualizacionesyNotificaciones, SolicitudyRespuesta
* 50393905 ->11000000001111001100110001-> RegistrarPaquetes, Detalles, SalienteyEntrante, TCPyUDP, Consultas/TransferyActualizacionesyNotificaciones, SolicitudyRespuesta




### DHCPServer
La configuración del servidor DNS se guarda sobre la clave de registro H`KEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\DHCPServer`
Y la información que a nosotros nos interesa está sobre la clave `Parameters`

![](/assets/img/winlogs/dhcp-server-log-config.png)

Siendo los parámetros que nos interesan:
* DhcpLogFilePath: Ubicación de los logs de IPV4
* DhcpV6LogFilePath: UBicación de los logs de IPV6

El formato de los logs sigue el formato `DhcpSrvLog-xxx.log`. Ejemplos:
* DhcpSrvLog-Lun.log: Log del lunes para IPV4
* DhcpV6SrvLog-Mar.log: Log del martes para IPV6
* DhcpSrvLog-Mier.log: Log del miercoles para IPV4
* DhcpV6SrvLog-Mié.log: Log del miercoles para IPV6. Ojo, hay una tilde y es de 3 letras en contraste con IPV4.

Y por supuesto si estuviera en ingles serían:
* DhcpSrvLog-Mon.log: Log del lunes para IPV4
* DhcpV6SrvLog-Tue.log: Log del martes para IPV6