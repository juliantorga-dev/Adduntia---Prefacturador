# 28 · Acta de validación — Cruce tableros Monday × Procesos Adduntia

**Fecha:** 12 de agosto de 2026 · actualizada el 14/8
**Validó:** Maximiliano Scocozza
**Destino:** insumo de diseño para Salesforce × Vurpix

Las secciones 1 a 4 cierran la validación de alcance; la 5 son los requisitos de
diseño que hay que llevarse a las épicas.

> **Fuente de verdad.** Esta acta y el resto de la documentación de
> [`procesos/`](procesos/README.md) son la referencia vigente del alcance y de
> los procesos. Reemplazan a las planillas de trabajo previas del relevamiento,
> que quedaron superadas y no deben usarse como fuente.

---

## 1. Alcance de este documento

Este cruce responde una sola pregunta: **qué procesos corren o atraviesan las
tablas que teníamos en Monday y que vamos a migrar a Salesforce.**

Eso **no** define qué otros procesos van a Salesforce. Hay procesos nuevos, o que
nunca estuvieron sistematizados, que no pasaban por Monday. Esa es una decisión
posterior y no se toma acá.

El universo complementario ya está relevado: los 25 procesos de las 7 verticales
estratégicas que no figuran en el mapa y no pasan por Monday están documentados
en `procesos_faltantes.md` (Armonía Familiar y Gobierno, Tax Advisory, Real
Estate Strategy, Corporate Finance, Private Banking &
Portfolio Management, Asesoramiento Estratégico).

Este archivo debe leerse como *el recorte de Monday*, no como el inventario
completo de lo que va a Salesforce.

## 2. Glosario — dos acepciones de "ticket"

La palabra se usa con dos sentidos distintos y la distinción es material para el
diseño de la tipificación de casos:

- **Ticket** — tarea interna o relacionada con un cliente. Es lo que hoy vive en
  el tablero *Customer Success* y lo que va a ser el objeto central de la
  ticketera.
- **Ticket de operación** — caso especial: un pedido de un cliente que se deriva
  a la mesa con un conjunto específico de parámetros. Es lo que modela el tablero
  *Tickets de Operaciones*.

Son dos ramas distintas y deben separarse desde el primer nivel de tipificación.

## 3. Resolución tablero por tablero

### 3.1 Comisiones — el sistema de registro es el histórico

**Entran los dos tableros.** `Comisiones Adduntiers Históricos` es el **sistema de
registro** (el ledger de acuerdos) y `Comisiones Adduntiers` es la **vista del
acuerdo vigente**.

El histórico guarda la serie completa de acuerdos y es lo que consume hoy el
proceso de cálculo de comisiones. El comportamiento actual —y el que hay que
preservar— es: un cambio en el acuerdo vigente genera una fila nueva en el
histórico, cuya fecha de registro es la fecha de inicio del nuevo acuerdo.

Ver **Requisito R1**: esto no es un mapeo de tablero, es un requisito de modelo
de datos.

> **Nota.** Relevamientos previos marcaban solo el tablero vigente como parte del
> alcance. Queda corregido: **entran los dos**, con el histórico como sistema de
> registro.

### 3.2 Comisiones Adduntiers — alta de proceso

Se da de alta el proceso de **compensación a asesores (Adduntiers)** en el
inventario. Hoy no figura entre los 102 procesos relevados: el gap es del
inventario, no del tablero. El proceso existe, está automatizado y tiene
pipeline y repositorio propios.

### 3.3 Cálculo Comisiones — tabla de parámetros

**Entra al alcance.** No es un proceso: es una **tabla auxiliar de configuración**
con los parámetros de la comisión que Adduntia cobra a cada banco (banco, base
revenue, moneda, share).

Su función es permitir configurar **cómo se calculan las comisiones a nivel
global**, y por eso necesita ser **editable por el rol de negocio que administra
esa configuración** (hoy, Flor Daiban) — ver el documento de roles y privilegios.
El cálculo en sí sigue corriendo en el Datahub (3.1 y R1).

### 3.4 Divisa e Inflación — no se mantienen como tableros

No migran como tableros. El dato debe llegar a Salesforce **desde el Datahub**
(el ecosistema GCP de Adduntia), como referencia para todos los procesos de
cálculo de invoices que se transfieren de Monday a Salesforce.

Ver **Requisito R2**.

### 3.5 Tickets de Operaciones — referencia de campos

El tablero nunca se puso en producción. Entra como **referencia de campos**: su
esquema sirve para saber qué datos usan el equipo de FAs y la Mesa para dar
tratamiento a tickets de operaciones. No tiene datos productivos que migrar.

> **No tomar este tablero de forma literal.** Es material complementario de lo
> que ya se conversó con Franco sobre el alcance de operaciones: suma contexto de
> campos, no define el diseño.

### 3.6 Activities, Servicios ERP y Tarifario — tablas de soporte

Los tres son soporte: no aportan un proceso propio, atraviesan procesos de otros.

- **Activities** — agenda de actividades comerciales, atada a Leads.
- **Servicios ERP** — catálogo de servicios que sostiene la integración con
  Contabilium. Los códigos deben seguir casando 1 a 1 con el ERP.
- **Tarifario** — tabla de precios y costos que sustenta la definición de fee.
  La columna `Costo Adduntia` es margen, no precio: debe tener destino explícito.

### 3.7 Clientes — tabla maestra con proceso embebido

Es la tabla maestra de cliente **y** contiene el proceso de alta embebido. Todo
el registro del alta tiene que vivir en algún lado.

Ver **Requisito R5**.

### 3.8 Customer Success — referencia para armar la ticketera

Entra como referencia. El propósito del tablero era el **seguimiento interno de
los temas y tareas que se tienen con cada cliente**, y eso es lo que la ticketera
tiene que resolver.


### 3.10 Real Estate, Otras Inversiones y Activos Financieros — no migran

Eran la forma de ver el patrimonio de las familias. Hoy esa información **vive en
Addepar**, que es donde se ingesta, se procesa y se publica el informe
patrimonial. Estas tablas quedan fuera del alcance de Salesforce.

### 3.11 Cobros — es Invoices

No es un tablero aparte. `Cobros` e `Invoices` son lo mismo, y es uno de los
tableros más relevantes del alcance: ahí vive todo el proceso de facturación y
seguimiento de pagos de clientes.

## 4. Tableros excluidos del alcance

| Tablero | Motivo |
|---|---|
| Seguimiento Banca Diamond/Black | Fuera de alcance. |
| Seguimiento Banca Platinum | Fuera de alcance. |
| Interno Adduntia sim | Fuera de alcance. |
| Activos | Reemplazado por Addepar (3.10). |
| Real Estate | Reemplazado por Addepar (3.10). |
| Otras Inversiones | Reemplazado por Addepar (3.10). |
| Divisa | No como tablero; el dato llega desde el Datahub (3.4). |
| Inflación | No como tablero; el dato llega desde el Datahub (3.4). |

---

## 5. Requisitos que surgen de la validación

Estos cinco puntos **no son mapeos de tablero**. Son requisitos de diseño de
Salesforce que salen de las definiciones de arriba y que deben leerse al escribir
las épicas.

### R1 · Ledger de acuerdos de comisiones con vigencia temporal

**Qué se necesita.** El acuerdo vigente es editable, pero editarlo **no
sobrescribe**: cierra el registro anterior y crea uno nuevo, cuya fecha de
registro es la fecha de inicio del nuevo acuerdo. La serie completa queda
consultable.

**Qué implica.** No puede modelarse como tabla plana ni como un campo editable
sobre una única fila. Requiere un objeto con vigencia (fecha desde / fecha
hasta), la automatización que cierra y da de alta al editar, y una vista o campo
calculado que resuelva "acuerdo vigente".

**Por qué es crítico.** El proceso de cálculo de comisiones consume el histórico
completo, no sólo el acuerdo vigente. Migrar únicamente el estado actual rompe el
cálculo y la pérdida no es recuperable: es el ledger.

**Criterio de aceptación.** (a) Editar un acuerdo vigente produce dos registros
consultables, con fechas contiguas y sin solapamiento. (b) Para un período
histórico dado, el cálculo de comisiones sobre Salesforce reproduce los mismos
resultados que el proceso actual.

### R2 · Datos de referencia desde el Datahub hacia Salesforce

**Qué se necesita.** Divisa e inflación llegan a Salesforce desde el Datahub
(GCP), como datos de referencia para los procesos de cálculo de invoices que se
transfieren de Monday.

**Alerta de alcance.** La dirección es **GCP → Salesforce (entrante)**. El
alcance del proyecto contempla un *"conector de exportación de datos hacia el
Data Hub de GCP (BigQuery)"*, que es la dirección **saliente**. Es otro conector
y otro esfuerzo.

**Acción.** Verificar contra el SOW antes de darlo por incluido. Si el conector
entrante no está contemplado, tratarlo como change request y plantearlo ahora, no
cuando Vurpix esté construyendo la facturación.

**A definir con Vurpix.** Frecuencia y granularidad de la sincronización, y si el
consumo es por lectura directa o por réplica en Salesforce.

### R3 · Modelo de acceso — se trata aparte

Los requisitos de **visibilidad y privilegios** no se enumeran acá: se tratan de
manera integral en el **documento de roles y privilegios**, que cubre qué ve y
qué edita cada equipo y cada rol en todos los objetos.

### R4 · La tipificación de casos debe separar las dos acepciones de "ticket"

Ver glosario (sección 2). El alcance de Fase 1 incluye gestión de casos con hasta
tres niveles de tipificación: **ticket interno/de cliente** y **ticket de
operación** son ramas distintas y deben separarse desde el primer nivel. Si
colisionan en la misma categoría, la tipificación queda mal construida y
corregirla implica rehacer el árbol.

### R5 · El registro del alta de cliente debe sobrevivir

`Clientes` es tabla maestra y a la vez soporta el proceso de alta, con estado de
alta, estado final, fecha de cierre y duración.

**Esos cuatro campos son hoy la única medición de performance del onboarding.**

El destino queda a definición de Vurpix —objeto propio de Alta, o estado sobre la
cuenta alimentado por los Screen Flows de onboarding de Fase 1—, pero los cuatro
datos deben existir y ser reportables.

---

*Acta redactada el 12/8/2026 a partir de las definiciones de Max, sobre el export
de esquemas del workspace de Monday, el alcance del proyecto FSC y las notas del
Workshop Técnico del 31/7. Actualizada el 14/8. El detalle de cada proceso está
en [`procesos/`](procesos/README.md) y el modelo de datos en el
[DER](2026-08-13_der-monday-salesforce.md).*
