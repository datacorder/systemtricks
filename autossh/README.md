##Tuneles SSH pesistentes

####Descripción

Hacer fowarding de puertos entre servidores a través de túneles SSH permite solucionar de forma fácil y segura situaciones que de otra forma o no tienen solución o simplemente son imposibles. Por ejemplo problemas de acceso a servidores protegidos por ferreas políticas de filtrado perimetral, replicación en tiempo real de bases de datos, ejecución remota de entornos gráficos, y un largo etcétera de situaciones.

La configuración mostrada está probada en Debian Jessie. Debería funcionar igual en Ubuntu y sistemas similares.

Tenemos un escenario con dos máquinas: server_A y server_B.

server_A es una máquina inaccesible desde Internet, a la que queremos acceder a alguno de sus servicios (ej. SSH o MySQL)
.
server_B es una máquina accesible desde Internet a la que tenemos acceso, que utilizaremos como máquina de salto hacia server_A.

####En server_A
- Crear un usuario (ej. tunels)
- Generar clave y privada y pública (ssh-keygen)
- Copiar clave pública a server_B (scp ~/.ssh/id_rsa.pub usuario@servidor_B:)

####En server_B
- Crear directorio ~/.ssh y darle permisos correctos (chmod 700 ~/.ssh)
- Copiar clave pública del usuario de server_A (cat ~/id_rsa.pub >~/.ssh/authorized_keys)

A partir de ahora el usuario puede autenticar desde server_A a server_B sin requerir contraseña.
Antes de seguir, probar que efectivamente no se pide contraseña. Si no es así los siguientes pasos no funcionarán.

####En server_A
- Instalar autossh (aptitude install autossh)
- El fichero /etc/systemd/system/autossh.service debe contener lo siguiente:

```
[Unit]
Description=AutoSSH para tuneles persistentes
After=network.target

[Service]
Type=simple
User=tunels
EnvironmentFile=/etc/default/autossh
ExecStart=
ExecStart=/usr/bin/autossh $SSH_OPTIONS
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```
Las directivas After y WantedBy pueden diferir en otros sistemas como Ubuntu. Para que funcione simplemente mantener la que traiga la distribución. La sección que debe ser igual es [Service]

- El fichero /etc/default/autossh debe contener lo siguiente:
```
AUTOSSH_USER=<usuario>
AUTOSSH_POLL=60
AUTOSSH_FIRST_POLL=30
AUTOSSH_GATETIME=0
AUTOSSH_PORT=40040
SSH_OPTIONS="-M 0 -N -p <puerto_ssh_destino> <usuario>@servidor_B -R localhost:<puerto_destino1>:localhost:<puerto_origen1> \
-R localhost:<puerto_destino2>:localhost:<puerto_origen2> \
-i /home/<usuario>/.ssh/id_rsa"
```

Explicación de los parámetros:
- _**usuario**_: usuario creado para la generación de las claves
- _**puerto_origen1**_: puerto del server_A al que queremos acceder desde el server_B
- _**puerto_destino1**_: puerto en server_B a traves del cual accederemos al puerto_origen del server_A

_**puerto_origen2**_ y _**puerto_destino2**_ permite hacer forward de más puertos en una solo autossh. Se pueden incluir tantas líneas como se quieran. Yeah.

####Ejecucion
- Refrescar la config del demonio: service autossh reload
- Iniciar tulen: service autossh start

En /var/log/daemon.log se puede hacer debug de lo que ocurre

Si todo ha ido bien, en server_B deben aparecere los <puerto_destinoX> mediante un netstat (ej. netstat -antp|grep LISTEN)

