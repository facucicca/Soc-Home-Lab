# Lab 01: Monitoreo de procesos en Windows 11 con Sysmon

## Introducción

En este laboratorio armé una máquina virtual con Windows 11 Enterprise, instalé Sysmon usando PowerShell y comprobé que funcionaba generando yo mismo una actividad y buscándola como evento en el Visor de eventos de Windows.

Es la primera etapa de mi home lab de SOC. Más adelante, los logs de Sysmon van a ser la fuente de datos de una herramienta de monitoreo (SIEM).

## Entorno

- **Virtualización:** VirtualBox
- **Sistema operativo:** Windows 11 Enterprise (ISO de evaluación de Microsoft)
- **Recursos de la VM:** 4 GB de RAM, 2 CPUs,  60 GB de disco, con EFI y TPM 2.0 activados
- **Herramientas:** Sysmon (Sysinternals) y la configuración `sysmonconfig-export.xml` de SwiftOnSecurity

## Paso a paso

### 1. Instalar VirtualBox
Instalé VirtualBox con las opciones por defecto.

### 2. Crear la VM e instalar Windows
Configuré el hardware de la VM y apunté la ISO de Windows 11 Enterprise. Dejé que el sistema operativo se instalara por completo hasta llegar al escritorio. Después:

1. Apagué Windows desde el menú Inicio.
2. Quité la ISO del almacenamiento de la VM.
3. Arranqué la VM de nuevo para comprobar que iniciaba bien desde el disco, sin volver a instalar.
4. Recién ahí tomé un snapshot como copia de seguridad.

### 3. Descargar Sysmon y su configuración
Sysmon es una herramienta gratuita de Microsoft (Sysinternals) que registra con mucho detalle lo que pasa en Windows: procesos que se ejecutan, conexiones de red, archivos creados. Va a ser la fuente de datos de mi herramienta de monitoreo.

Lo descargué desde dentro de la VM y lo extraje en `C:\Tools\Sysmon`.

También descargué `sysmonconfig-export.xml` del repositorio [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config). Este archivo no es obligatorio para usar Sysmon, pero filtra el ruido y deja en el Visor de eventos lo que sirve para monitorear.

### 4. Instalar Sysmon
Desde PowerShell, ejecutado como administrador:

```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

Para confirmar que el servicio estaba activo:

```powershell
Get-Service Sysmon64
```

El servicio quedó en estado Running.

### 5. Prueba de funcionamiento
Para forzar que se registrara algo, abrí una consola `cmd` y escribí `notepad` para abrir el Bloc de notas.

Después abrí el Visor de eventos y fui a:

`Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`

Filtré por **Event ID 1** (Process Create) y encontré el evento de la ejecución de Notepad, con su ruta y su línea de comandos:

`C:\Program Files\WindowsApps\Microsoft.WindowsNotepad_...\Notepad\Notepad.exe`

<img width="1535" height="908" alt="image" src="https://github.com/user-attachments/assets/a53f1d95-b0ad-4eb1-9d7b-f7a813fe953b" />

## Análisis

Lo más importante de cada evento no es solo el nombre del programa, sino **quién lo lanzó**. Estos son los procesos que analicé:

| Proceso (Image) | Proceso padre (ParentImage) | Qué significa |
|---|---|---|
| cmd.exe | explorer.exe | Abrí la consola desde el escritorio |
| WindowsTerminal.exe | svchost.exe | La ventana de la terminal; probablemente la abre el sistema por su cuenta, no `cmd.exe` |
| Notepad.exe | cmd.exe | Lo lancé yo desde la consola |

Algunas cosas que aprendí:

- En Windows 11, Notepad corre desde `C:\Program Files\WindowsApps\...` y no desde `System32`, como en versiones anteriores. Es lo esperado.
- Por eso importa mirar la **ruta completa** y no solo el nombre. Un `notepad.exe` ejecutándose desde una carpeta temporal o de usuario sería sospechoso.
- Al principio vi un evento de `WindowsTerminal.exe` con `svchost.exe` como padre y pensé que se debía a algo que yo había hecho con la consola. Al seguir la cadena completa vi que la que yo lancé fue `explorer.exe` → `cmd.exe` → `Notepad.exe`, y que la terminal es un proceso aparte que abre el sistema. Aprendí a no sacar conclusiones antes de reconstruir la cadena completa.
- El campo `IntegrityLevel: Medium` indica que Notepad corrió con permisos de usuario estándar, no elevados. Un proceso sospechoso con nivel `High` o `System` tendría más capacidad de daño.
- Sysmon guarda la hora en UTC (19:40) y el Visor la muestra en hora local (16:40). En un SOC los logs se correlacionan en UTC.

## Problemas y cómo los resolví

**Síntoma:** al volver a abrir la VM, empezaba a instalar Windows 11 otra vez, aunque ya lo había instalado y había llegado al escritorio. Pasó incluso después de restaurar un snapshot.

**Diagnóstico:** revisé el almacenamiento de la VM y el orden de arranque. El disco virtual ocupaba unos 17 GB, así que ese dato por sí solo no me decía si la instalación estaba completa o no. Los snapshots guardan también la configuración de la VM, incluida la ISO conectada.

**Causa probable:** la VM arrancaba desde la ISO en vez de desde el disco, y los snapshots viejos conservaban esa configuración.

**Solución:** terminé la instalación completa, apagué desde el menú Inicio, quité la ISO, eliminé los snapshots anteriores, comprobé que la VM arrancaba sola desde el disco y recién entonces tomé un snapshot nuevo.

**Lección:** probar siempre un arranque completo antes de guardar un snapshot.
