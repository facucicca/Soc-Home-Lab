# Lab 02: SIEM con Wazuh y detección propia sobre eventos de Sysmon

## Introducción

En este laboratorio armé un servidor Wazuh en una VM de Ubuntu Server, conecté a él la VM de Windows 11 con Sysmon del [Lab 01](../Lab.sysmon01) mediante el agente de Wazuh y escribí una regla de detección propia que genera una alerta cuando `notepad.exe` es lanzado desde `cmd.exe`, la misma actividad que había analizado a mano en el Lab 01.

El objetivo fue completar el ciclo: **evento en el endpoint → recolección → análisis → alerta en el SIEM**.

## Entorno

- **Equipo físico:** PC con 16 GB de RAM, VirtualBox
- **Red:** red NAT de VirtualBox (`NatNetwork`) compartida por las dos VMs
- **Servidor:** Ubuntu Server 24.04.5 LTS, 4 CPUs, 8 GB de RAM, 50 GB de disco, IP `10.0.2.5`
- **SIEM:** Wazuh 4.14.9 (instalación todo en uno: manager, indexer y dashboard)
- **Endpoint:** Windows 11 Enterprise (evaluación), 4 GB de RAM, 2 CPUs, Sysmon con la configuración de SwiftOnSecurity y agente de Wazuh, IP `10.0.2.4`

## Paso a paso

### 1. Preparar la red
Creé una red NAT en VirtualBox y conecté el Adaptador 1 de ambas VMs a ella, para que se vean entre sí y tengan internet.

### 2. Instalar Ubuntu Server
Instalé Ubuntu Server 24.04 en una VM nueva (`Wazuh-Server`), actualicé el sistema con `apt update` y `apt upgrade`, y comprobé la red, la memoria y el espacio con `ip a`, `free -h` y `df -h /`.

### 3. Instalar Wazuh
Usé el asistente oficial de instalación de la versión 4.14:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

Al terminar verifiqué que los cuatro servicios estuvieran activos y que los tres paquetes principales tuvieran la misma versión:

```bash
systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard filebeat
dpkg -l | grep wazuh
```

Resultado: los cuatro servicios en `active` y `wazuh-manager`, `wazuh-indexer` y `wazuh-dashboard` en la versión `4.14.9-1`.

### 4. Conectar el agente de Windows
Desde el panel de Wazuh (`https://10.0.2.5`) usé **Deploy new agent**, elegí Windows e indiqué `10.0.2.5` como dirección del servidor. Ejecuté el comando que generó el asistente en PowerShell como administrador e inicié el servicio. El agente `LAB-Win11En` pasó de *Never connected* a **Active**.

Los eventos de Sysmon llegaron al servidor, donde las reglas de Wazuh los analizan (por ejemplo, las reglas `92205`, `92066` y `92021`, que aparecieron en el panel).

### 5. Escribir una regla propia
En `local_rules.xml` (Server management → Rules → Manage rules files) agregué:

```xml
<group name="sysmon,local,">
  <rule id="100100" level="5">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.image" type="pcre2">(?i)notepad\.exe$</field>
    <field name="win.eventdata.parentImage" type="pcre2">(?i)cmd\.exe$</field>
    <description>Notepad lanzado desde cmd.exe (prueba de laboratorio)</description>
  </rule>
</group>
```

La regla parte de los eventos de creación de proceso de Sysmon (ID 1) y exige dos condiciones: que el programa sea `notepad.exe` y que su proceso padre sea `cmd.exe`. Después reinicié el manager para que se cargara.

![Regla 100100 en local_rules.xml](regla-100100.png)

### 6. Prueba
En la VM de Windows abrí `cmd` y escribí `notepad`. En **Threat Hunting → Events** apareció la alerta de la regla `100100` (nivel 5).

![Alerta de la regla 100100](alerta-100100.png)

## Análisis

- La alerta corresponde al mismo evento que analicé a mano en el Lab 01 (`Notepad.exe` con `cmd.exe` como proceso padre), ahora detectado automáticamente por el SIEM.
- Con una primera versión de la regla, que solo miraba el nombre del programa, aparecieron **dos alertas** para una sola apertura de Notepad. Al agregar la condición del proceso padre, apareció una. No investigué en detalle por qué eran dos.

![Primera versión de la regla: dos alertas](alerta-100100-v1.png)

- En el panel también aparecen alertas repetidas de las reglas `92205`, `92066` y `92021` (PowerShell y `SecEdit.exe`) sin que yo haya ejecutado nada. Al abrir una de ellas vi que es un evento de **creación de archivo** de Sysmon (ID 11): PowerShell de 32 bits (`SysWOW64`), ejecutado como `NT AUTHORITY\SYSTEM`, crea el archivo `__PSScriptPolicyTest_*.ps1` en `C:\Windows\SystemTemp`. Mi hipótesis es que lo generan los chequeos del propio agente de Wazuh, pero **no la comprobé**: queda pendiente revisar el `parentImage` de esos eventos.

![Detalle de una alerta de PowerShell](alerta-powershell-92205.png)

- Wazuh solo muestra como alerta los eventos que coinciden con alguna regla. Por eso la actividad normal, como abrir Notepad, no aparecía hasta que escribí la regla.

## Problemas y cómo los resolví

- **VM de Windows en pantalla negra en la PC.** La barra de estado de VirtualBox mostraba una tortuga con `Execution Engine: native API`, es decir, VirtualBox corría sobre la capa de Hyper-V de Windows. Con PowerShell comprobé que `hypervisorlaunchtype` estaba en `Auto` y que la seguridad basada en virtualización (VBS) estaba activa (`VirtualizationBasedSecurityStatus` = 2). Cambié el valor con `bcdedit /set hypervisorlaunchtype off`, reinicié la PC y después pude instalar Windows en la VM.
- **Disco lleno durante la instalación de Ubuntu** (error `DrvVD_DISKFULL`). VirtualBox pausó la VM para no corromper datos. Después de resolver el espacio en el disco de la PC, la instalación de Ubuntu se completó.
- **Archivo `.vbox` de la VM de Windows en 0 KB.** VirtualBox se cerró mientras movía la VM con el disco casi lleno, y la VM quedó como *Inaccesible* con el error `Document is empty`. El disco `.vdi` estaba intacto y existía una copia de respaldo `.vbox-prev` de la configuración.
- **No se podía pegar texto en la consola de Ubuntu Server.** El portapapeles compartido de VirtualBox necesita una sesión gráfica, y Ubuntu Server no la tiene. Probé enviar texto como pulsaciones de teclado con `VBoxManage controlvm ... keyboardputstring`.
- **La regla no se activaba.** Faltaba reiniciar el manager de Wazuh: las reglas nuevas solo se cargan al reiniciarlo.

## Limitaciones y próximos pasos

- No hice la prueba negativa (abrir Notepad sin pasar por `cmd` y comprobar que no alerta), así que no puedo afirmar que la regla distinga el contexto en todos los casos.
- La regla es un ejercicio y no una detección útil en producción: abrir Notepad no es malicioso.
- **Próximo laboratorio:** simular técnicas de ataque con Atomic Red Team, ver qué detecta Wazuh, mapearlas a MITRE ATT&CK y escribir reglas más realistas.
