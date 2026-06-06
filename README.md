# Importador seguro de PST a Maildir++ en cPanel/Dovecot

## Nota importante antes de comenzar

Este repositorio documenta un caso de estudio técnico. La implementación productiva completa no se publica, pero dispongo del flujo completo, probado y preparado para su adaptación a entornos reales en producción.

Si una persona, organización o empresa necesita una solución similar para convertir ficheros PST de Outlook a una estructura Maildir++ completa e importarla en buzones cPanel/Dovecot en producción, puede contactarme directamente.

Los medios de contacto están disponibles en el README.md de mi repositorio personal "v80000".

Toda consulta o posible implementación se tratará con privacidad y confidencialidad, el cual como se sabe de mí, siempre es mi prioridad.

Este flujo está pensado para migraciones donde no basta con "sacar correos de un PST", sino que hace falta reconstruir una estructura de buzón usable por Dovecot, preservando carpetas, subcarpetas, carpetas vacías, subscriptions, permisos, visibilidad IMAP y trazabilidad, ya que está pensado para PST de todo tipo, en producción lo usé con uno de 20GB.

## Descripción

Este proyecto documenta un flujo de automatización que diseñé y desarrollé para convertir ficheros PST (sean del tamaño que sean, en producción lo hice con uno de 20GB, con 3500 directorios, y 18500 mensajes) a una estructura Maildir++ completa y posteriormente importarla en un buzón real de cPanel/Dovecot.

El flujo completo está pensado para trabajar con una fase intermedia segura: primero se extrae el PST usando `readpst -r -o`, generando una estructura de carpetas con archivos `mbox`; después se convierte esa estructura a Maildir++, respetando carpetas vacías, subcarpetas y nombres de buzón compatibles con Dovecot; finalmente se importa el Maildir++ resultante al buzón final, se aplican permisos correctos y se fuerza un resync de Dovecot.

La solución no es simplemente "convertir un PST". En una migración real hay que controlar rutas, nombres de carpetas, separadores, carpetas vacías, mensajes inválidos, mbox vacíos, longitud de nombres, `subscriptions`, permisos, owner del buzón, compatibilidad con cPanel, visibilidad de carpetas en IMAP y logs de auditoría.

Este repositorio es una documentación técnica de mi proyecto y de su arquitectura. Los scripts productivos reales no se publican por motivos de confidencialidad, seguridad operativa y protección de propiedad intelectual, para implementación, contactarme diréctamente.

---

## Nota rápida sobre los ejemplos

Aunque este repositorio no incluye la implementación productiva real, recomiendo revisar la carpeta `complete_log_examples/`, donde incluyo salidas anonimizadas del flujo: comando de extracción con `readpst`, conversión de estructura mbox a Maildir++, importación al buzón final, logs de una importación correcta y ejemplos de errores o avisos controlados.

Estos ejemplos ayudan a entender cómo se comporta el sistema en la práctica, qué información queda registrada y cómo se puede auditar cada operación realizada sobre un buzón.

---

## Problema que resuelve

En migraciones desde Outlook o entornos antiguos puede ser necesario importar uno o varios ficheros `.pst` en buzones alojados en cPanel/Dovecot.

El problema es que el PST no es directamente utilizable como Maildir++.

Una conversión manual o incompleta puede provocar:

* Carpetas perdidas.
* Subcarpetas mal convertidas.
* Carpetas vacías omitidas.
* Mensajes importados en rutas incorrectas.
* Nombres incompatibles con el separador de Dovecot.
* Falta de `subscriptions`, haciendo que el usuario no vea carpetas en IMAP.
* Permisos incorrectos en el buzón final.
* Necesidad de forzar resync para que Dovecot vea correctamente el contenido.
* Falta de logs para saber qué se ha importado y qué se ha omitido.
* Riesgo al copiar directamente contenido al buzón final sin una fase previa de preparación.

En entornos donde se gestionan buzones de empresas, migraciones de correo o recuperaciones desde backups PST, el impacto puede ser considerable.

---

## Objetivo del proyecto

El objetivo fue diseñar un sistema automatizado para:

* Extraer el contenido de un PST usando `readpst -r -o`.
* Trabajar sobre una estructura intermedia basada en carpetas y archivos `mbox`.
* Convertir esa estructura a Maildir++.
* Preservar carpetas vacías.
* Preservar subcarpetas mediante normalización compatible con Dovecot.
* Crear `cur`, `new`, `tmp` y `maildirfolder` donde sea necesario.
* Convertir mensajes desde archivos `mbox` válidos.
* Omitir mbox vacíos o inválidos de forma controlada.
* Generar `subscriptions` automáticamente.
* Importar el Maildir++ final al buzón real.
* Excluir rutas no deseadas como `lost+found`.
* Ajustar owner y permisos.
* Forzar `doveadm force-resync`.
* Mostrar conteos antes y después de la importación.
* Generar logs auditables.

---

## Arquitectura general

El flujo se dividió en tres partes principales:

1. Extracción del PST a estructura intermedia con `readpst`.
2. Conversión de estructura mbox a Maildir++.
3. Importación del Maildir++ al buzón final cPanel/Dovecot.

La idea era no copiar nunca el resultado directamente al buzón final sin antes construir y revisar una estructura Maildir++ completa.

```text
Archivo PST
        ↓
Extracción con readpst -r -o
        ↓
Estructura intermedia con carpetas y mbox
        ↓
Validación de origen
        ↓
Creación de estructura Maildir++ completa
        ↓
Normalización de carpetas para separator = .
        ↓
Conversión de mbox válidos
        ↓
Creación de carpetas vacías y maildirfolder
        ↓
Generación de subscriptions
        ↓
Conteos y logs
        ↓
Importación al buzón destino
        ↓
Ajuste de owner y permisos
        ↓
doveadm force-resync
        ↓
Listado de carpetas visibles por Dovecot
        ↓
Resumen final
```

---

## Scripts diseñados

El sistema se diseñó en varias piezas, separando la fase de extracción, la fase de conversión y la fase de importación.

### 1. Extracción del PST con readpst

La primera fase utiliza `readpst` para extraer el PST a una estructura de carpetas.

El comando base es:

```text
readpst -r -o /migration/readpst_out archivo.pst
```

El uso de `-r` permite generar una estructura recursiva de carpetas, y `-o` define la ruta de salida donde quedará la estructura intermedia.

Esta fase no escribe todavía en el buzón final.

Su objetivo es dejar el PST convertido a un formato intermedio que pueda ser inspeccionado y procesado posteriormente.

---

### 2. Conversor de estructura mbox a Maildir++

Este componente recorre el resultado generado por `readpst`.

Comprueba, entre otras cosas:

* Que existe el directorio de origen.
* Cuántos directorios se han generado.
* Cuántos archivos `mbox` existen.
* Qué carpetas deben crearse aunque no tengan mensajes.
* Qué archivos `mbox` están vacíos.
* Qué archivos no parecen un `mbox` válido.
* Qué nombres de carpeta deben normalizarse.
* Qué rutas deben omitirse por longitud excesiva.

La conversión tiene en cuenta que Dovecot puede usar `separator = .`, por lo que las rutas internas del PST se transforman en nombres Maildir++ usando puntos.

Ejemplo:

```text
Clientes/Empresa A/Facturas
```

Se convierte en:

```text
.Clientes.Empresa A.Facturas
```

También se reemplazan caracteres problemáticos como `:` por `_`, manteniendo espacios y acentos cuando sea posible.

La estructura Maildir++ generada incluye:

```text
cur/
new/
tmp/
maildirfolder
```

Y al final se genera el fichero:

```text
subscriptions
```

---

### 3. Importador de Maildir++ al buzón cPanel/Dovecot

Este componente toma el Maildir++ final y lo importa en el buzón destino.

Comprueba, entre otras cosas:

* Que existe el origen Maildir++.
* Que existe el destino real del buzón.
* Número de carpetas Maildir++ en origen.
* Número de mensajes en origen.
* Número de subscriptions en origen.
* Copia completa de la estructura al destino.
* Exclusión de `lost+found`.
* Reasegurado de `cur`, `new`, `tmp` y `maildirfolder`.
* Ajuste de owner.
* Ajuste de permisos.
* `doveadm force-resync` sobre el usuario.
* Conteo final en destino.
* Listado de primeras carpetas vistas por Dovecot.

El objetivo es que el resultado no solo exista en disco, sino que sea visible y utilizable por el buzón IMAP final.

---

### 4. Ejecución controlada y logs

Cada fase genera logs independientes.

Esto permite saber:

* Qué PST se ha tratado.
* Qué ruta intermedia se ha generado.
* Cuántas carpetas venían del PST.
* Cuántos mbox se han detectado.
* Cuántos mensajes se han convertido.
* Qué carpetas se han omitido.
* Qué avisos han aparecido.
* Qué Maildir++ final se ha generado.
* Qué buzón final ha recibido la importación.
* Qué owner y permisos se aplicaron.
* Qué carpetas ve Dovecot tras el resync.

---

## Estrategia de seguridad

El flujo se diseñó pensando en entornos reales, ya que lo desarrollé por la necesidad de una empresa, y utilicé en producción.

Algunas medidas importantes:

* No se importa directamente desde el PST al buzón final.
* Se usa una ruta intermedia generada por `readpst`.
* Se construye un Maildir++ final antes de tocar el buzón destino.
* Se comprueba la existencia de origen y destino.
* Se registran conteos antes y después.
* Se preservan carpetas vacías.
* Se genera `subscriptions`.
* Se normalizan nombres de carpeta.
* Se omiten rutas demasiado largas.
* Se omiten mbox vacíos o inválidos.
* Se reasegura la estructura Maildir en destino.
* Se aplican permisos y owner después de la copia.
* Se fuerza resync de Dovecot.
* Se genera log completo de cada ejecución.

---

## Consideraciones sobre rollback

Este flujo no modifica el PST original.

La fase de conversión trabaja sobre una salida intermedia y genera un Maildir++ separado, por lo que se puede repetir el proceso sin perder el origen.

En producción, antes de importar al buzón final, lo correcto es disponer de:

* Backup del buzón destino.
* Ruta de staging separada.
* Validación del Maildir++ generado.
* Ventana de mantenimiento si el buzón está en uso.
* Posibilidad de retirar la importación y restaurar el estado previo.

El objetivo no era solo automatizar la conversión, sino automatizarla de forma revisable y recuperable.

---

## Logs y trazabilidad

Cada ejecución genera logs.

Los logs permiten revisar:

* Fecha y hora de inicio.
* Ruta origen generada por `readpst`.
* Ruta de salida Maildir++.
* Directorios encontrados.
* Archivos `mbox` encontrados.
* Carpetas creadas.
* Mensajes importados.
* Carpetas omitidas.
* Warnings.
* Ruta destino final.
* Usuario Dovecot.
* Owner aplicado.
* Permisos aplicados.
* Resultado de `doveadm force-resync`.
* Conteos finales.
* Carpetas visibles por Dovecot.

Esto permite auditar qué se ha hecho y facilita la revisión en caso de incidencia.

---

## Ejemplo de extracción PST

```text
readpst -r -o /migration/readpst_out archivo.pst
```

Salida esperada:

```text
Opening PST file and indexes...
Processing Folder "Root"
Processing Folder "Inbox"
Processing Folder "Clientes"
Processing Folder "Clientes/Empresa A"
Processing Folder "Enviados"
Processing Folder "Archivo 2021"
Finished successfully
```

---

## Ejemplo de salida de conversión

```text
[+] Inicio: 2026-06-06 10:15:11
[+] Origen readpst: /migration/readpst_out/mailboxOK
[+] Salida Maildir++: /migration/maildir_final_cliente
[+] Log: /migration/logs/convertir_maildirpp_2026-06-06_101511.log

[+] Comprobando origen...
    Directorios encontrados: 84
    Archivos mbox encontrados: 61

[+] Fase 1: creando estructura completa de carpetas, incluidas carpetas vacías...

[+] Fase 2: importando mensajes desde archivos mbox...

----------------------------------------
[+] [1/61] Importando
    Carpeta PST: Inbox
    Maildir++:   .Inbox
    MBOX:        /migration/readpst_out/mailboxOK/Inbox/mbox
    Destino:     /migration/maildir_final_cliente/.Inbox

[+] Fase 3: generando subscriptions...

[+] Resumen:
    Carpetas Maildir++ generadas:
84
    Mensajes en cur:
18342
    Mensajes en new:
0
    Subscriptions:
84 /migration/maildir_final_cliente/subscriptions

[+] Terminado: 2026-06-06 10:43:02
```

---

## Ejemplo de salida de importación

```text
[+] Inicio importación Maildir++ al buzón: 2026-06-06 11:05:31
[+] Origen: /migration/maildir_final_cliente
[+] Destino: /home/cpuser/mail/example.com/usuario
[+] Usuario Dovecot: usuario@example.com
[+] Owner: cpuser:mail
[+] Log: /migration/logs/importar_maildir_buzon_2026-06-06_110531.log

[+] Conteo origen:
    Carpetas Maildir++: 84
    Mensajes origen: 18342
    Subscriptions origen: 84

[+] Copiando Maildir++ al buzón destino...
[+] Incluye carpetas vacías, subcarpetas, mensajes, maildirfolder y subscriptions.
[+] Excluyendo lost+found.

[+] Reasegurando estructura Maildir en cada carpeta...

[+] Ajustando permisos...

[+] Forzando resync Dovecot...

[+] Conteo destino tras importar:
    Carpetas Maildir++ destino: 84
    Mensajes destino: 18342
    Subscriptions destino: 84

[+] Primeras carpetas vistas por Dovecot:
Archivo 2021
Clientes
Clientes.Empresa A
Clientes.Empresa A.Facturas
Inbox
Sent

[+] Importación completada: 2026-06-06 11:08:18
```

---

## Tecnologías utilizadas

* Linux
* Bash
* `readpst`
* `mb2md`
* Dovecot
* `doveadm`
* cPanel mail layout
* Maildir / Maildir++
* MBOX
* PST
* `rsync`
* Procesamiento de estructura de carpetas
* Normalización de rutas
* Logging
* Validación operacional

---

## Por qué es útil para empresas

Esta solución es especialmente útil para proveedores de hosting, empresas de soporte IT, administradores de servidores cPanel/Dovecot o equipos que tengan que migrar correos históricos desde PST a buzones IMAP reales.

En esos casos, no basta con extraer mensajes. Es necesario que el usuario final pueda entrar por IMAP y ver sus carpetas correctamente.

Un flujo como este permite:

* Migrar PST a buzones cPanel/Dovecot.
* Preservar estructura de carpetas.
* Preservar subcarpetas.
* Mantener carpetas vacías.
* Generar `subscriptions`.
* Evitar trabajo manual repetitivo.
* Reducir riesgo operativo.
* Mantener trazabilidad de la operación.
* Revisar conteos antes y después.
* Forzar visibilidad correcta en Dovecot.

---

## Alcance de este repositorio

Este repositorio no contiene los scripts productivos reales.

Incluye documentación técnica, arquitectura y ejemplos anonimizados de entrada, salida y logs para explicar el diseño técnico del flujo sin publicar la implementación productiva.

La implementación completa se mantiene privada por motivos de:

* Confidencialidad.
* Seguridad operativa.
* Protección de propiedad intelectual.
* Evitar uso incorrecto en entornos de producción.
* Evitar exposición de detalles internos de infraestructura.

---

## Estado del proyecto

Proyecto realizado en un contexto real de administración de sistemas, orientado a resolver un problema operativo en infraestructura de correo.

Este repositorio actúa como caso de estudio técnico y portfolio profesional.

---

## Aviso

Este repositorio no debe usarse como herramienta lista para producción, son solo explicaciones de como funciona todo, mi flujo personalizado, el cual SI está listo para adaptar y aplicar en producción, quien se encuentre con este problema, como digo, que me contacte.

Operaciones sobre PST, mbox, Maildir, Dovecot y buzones reales pueden causar pérdida de datos o importaciones incompletas si se ejecutan sin pruebas, backups, validación y conocimiento del entorno.

Antes de aplicar un flujo similar en producción, es imprescindible adaptarlo, probarlo en laboratorio y disponer de una estrategia de recuperación.
