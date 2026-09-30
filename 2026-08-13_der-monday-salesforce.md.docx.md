# DER · Modelo de datos Monday → Salesforce (Adduntia)

**Fecha:** 14 de agosto de 2026 · **Para:** Vurpix (referencia de diseño) **Fuente:** export completo de esquemas del workspace Monday (`esquemas_completo.csv`) \+ validación de alcance de Max \+ los procesos documentados end-to-end. **Qué es:** el grafo de entidades y relaciones **reales** (columnas `board_relation` de Monday) de los tableros que migran a Salesforce, con los sistemas externos que los alimentan. Los `mirror` de Monday no se dibujan: son lookups derivados de estas mismas relaciones.

## Grado de avance de este documento

Documento **vivo**: se completa a medida que avanza la documentación de procesos. Estado actual:

| Contenido | Estado |
| :---- | :---- |
| Entidades y relaciones del recorte Monday (26 entidades, 7 dominios) | ✅ completo |
| Atributos con impacto de diseño (los que condicionan el modelo) | ✅ los relevantes · el detalle campo por campo vive en cada `PROC_xx` |
| Semántica de monedas, identidad y titularidades (advertencias §4) | ✅ verificada contra el sistema |
| Relación **beneficiario** | 🆕 incorporada 14/8 — hoy **sin datos cargados**, se pide que exista en el diseño |
| Cardinalidades | 🟡 salen del uso conocido; Monday no las declara → a confirmar en el diseño de detalle |
| Ticketera (P-07) y Addepar (P-09) | 🔜 sus entidades están dibujadas, pero se refinarán al cerrar esos procesos |

Los procesos ya documentados end-to-end están en [`procesos/README.md`](http://procesos/README.md); cada uno aporta al modelo y puede reabrir este documento.

---

## 1\. Diagramas

El modelo se presenta **por dominio** (un diagrama chico y legible por área, en lugar de un grafo único de 26 entidades). Una entidad que aparece en más de un dominio es **la misma tabla**: sus atributos se detallan solo en su dominio principal. El §1.0 muestra cómo se encadenan los dominios.

### 1.0 Vista de conjunto — cómo se conectan los dominios

`flowchart LR`  
    FUNNEL\["② `Funnel comercial<br/>Referrals · Leads"] -->|"Ganado"|` NUCLEO\["① `Núcleo maestro<br/>Clientes · Personas · Sociedades<br/>Cuentas Bancarias"]`  
    `NUCLEO -->` SERV\["③ `Servicios y facturación<br/>FO · CM · MSE · Trámites<br/>INVOICES · Pagos"]`  
    `NUCLEO -->` OPER\["④ `Operativa<br/>Ticketera · Transferencias"]`  
    `NUCLEO -->` COM\["⑤ `Comisiones<br/>Ledger · Vigente"]`  
    EXT\["⑥ `Externos<br/>Datahub GCP · Addepar · Contabilium"] -.-> SERV`  
    `EXT -.-> COM`  
    `EXT -.-> NUCLEO`

### 1.1 Núcleo maestro — el cliente y su gente

`erDiagram`  
    `CLIENTES ||--o{ TOMADORES_DECISION : "relacion persona-cliente"`  
    `CLIENTES ||--o{ BENEFICIARIOS : "relacion persona-cliente"`  
    `CLIENTES ||--o| CLIENTES : "conglomerado"`  
    `TOMADORES_DECISION }o--|| PERSONAS : "es una persona"`  
    `BENEFICIARIOS }o--|| PERSONAS : "es una persona"`  
    `PERSONAS ||--o{ FAMILIARES : "vinculo entre PERSONAS"`  
    `FAMILIARES }o--|| PERSONAS : "pariente (otra persona)"`  
    `CLIENTES ||--o{ STAFF : "staff asignado"`  
    `CLIENTES }o--o{ ADDUNTIERS : "asesor"`

    `CLIENTES {`  
        `string categoria "Black/Diamond/Platinum/Gold/..."`  
        `bool family_office`  
        `bool cash_management`  
        `bool tax_planning`  
        `string estado_alta_cliente "R5 - sobrevive"`  
        `string estado_final_alta "R5"`  
        `date fecha_cierre_alta "R5"`  
        `number duracion_alta "R5"`  
    `}`

### 1.2 Titularidades — propiedad ≠ relación comercial

`erDiagram`  
    `CLIENTES ||--o{ SOCIEDADES : "posee"`  
    `CLIENTES ||--o{ CUENTAS_BANCARIAS : "registra"`  
    `SOCIEDADES ||--o{ ACCIONISTAS : "subitems"`  
    `ACCIONISTAS }o--|| PERSONAS : "si personeria personal"`  
    `ACCIONISTAS }o--|| SOCIEDADES : "si personeria sociedad"`  
    `CUENTAS_BANCARIAS ||--o{ FIRMANTES : "subitems"`  
    `FIRMANTES }o--|| PERSONAS : "titular/firmante persona"`  
    `FIRMANTES }o--|| SOCIEDADES : "titular sociedad"`

### 1.3 Funnel comercial

`erDiagram`  
    `REFERRALS ||--o{ LEADS : "origina"`  
    `ACTIVITIES }o--|| LEADS : "agenda de"`  
    `LEADS ||--o| CLIENTES : "convierte en (Ganado)"`

    `LEADS {`  
        `string etapa_funnel "Leads/1a/2a/3a/Ganado/Perdido/Dormant"`  
        `string categoria_cliente`  
        `number patrimonio_total`  
    `}`

### 1.4 Servicios y facturación — INVOICES como hub

`erDiagram`  
    `FAMILY_OFFICE }o--|| CLIENTES : "abono por familia"`  
    `CASH_MANAGEMENT }o--|| SOCIEDADES : "abono por sociedad"`  
    `MANT_SOC_EXTERIOR }o--|| SOCIEDADES : "abono por sociedad"`  
    `TRAMITES_SOCIEDADES }o--|| SOCIEDADES : "tramite sobre"`  
    `TRAMITES_SOCIEDADES ||--o{ WORKFLOW_JURISDICCION : "detalle LLC/BVI/NEVIS"`  
    `TRAMITES_SOCIEDADES }o--o| SERVICIOS_ERP : "tipo de servicio"`

    `INVOICES }o--o| FAMILY_OFFICE : "generada por (auto)"`  
    `INVOICES }o--o| CASH_MANAGEMENT : "generada por (auto)"`  
    `INVOICES }o--o| MANT_SOC_EXTERIOR : "generada por (auto)"`  
    `INVOICES }o--o| TRAMITES_SOCIEDADES : "generada por (finalizar)"`  
    `INVOICES }o--|| CLIENTES : "factura a"`  
    `INVOICES }o--o| TARIFARIO : "precio / costo"`  
    `INVOICES ||--o{ PAGOS : "subitems (cobranzas)"`

    `INVOICES {`  
        `string concepto "catalogo de servicios facturables"`  
        `string origen "FO/MSE/CM/Tramites/..."`  
        `string moneda_contrato "MEP/USD/ARS/BNA"`  
        `string id_comprobante_erp "Contabilium"`  
        `string estado "Pendiente/Cobrado/NC/..."`  
    `}`  
    `PAGOS {`  
        `string id_pago "hoy SIN unicidad - dedup en GCP (bandaids 4-5)"`  
        `string moneda_recibido`  
        `string caja`  
        `number tipo_de_cambio "via Datahub"`  
        `string estado "cola de cobranza"`  
    `}`  
    `TRAMITES_SOCIEDADES {`  
        `string tipo_de_tramite "Crear NEVIS/LLC/BVI, Good Standing, ..."`  
        `string urgencia`  
        `string forma_de_pago`  
        `string finalizar "boton - dispara Invoice"`  
    `}`

### 1.5 Operativa diaria

`erDiagram`  
    `TICKETERA }o--|| CLIENTES : "cliente"`  
    `TICKETERA }o--o{ SOCIEDADES : "sociedades"`  
    `TICKETS_TRANSFERENCIAS }o--|| CLIENTES : "cliente"`  
    `TICKETS_TRANSFERENCIAS }o--|| CUENTAS_BANCARIAS : "cta origen / destino"`  
    `TICKETS_OPERACIONES }o--|| CLIENTES : "cliente"`  
    `TICKETS_OPERACIONES }o--|| CUENTAS_BANCARIAS : "account number"`

    `TICKETERA {`  
        `string area "Concierge/CF/RE/TaxPlanning/interno"`  
        `string tipologia`  
        `string prioridad`  
        `string visibilidad "R3 - restringida hasta liberar"`  
    `}`

### 1.6 Comisiones

`erDiagram`  
    `COMISIONES_VIGENTE ||--|| COMISIONES_LEDGER : "vista vigente del ledger (R1)"`  
    `COMISIONES_LEDGER }o--|| CLIENTES : "id cliente"`  
    `COMISIONES_LEDGER }o--o{ ADDUNTIERS : "adduntier"`

    `COMISIONES_LEDGER {`  
        `string adduntier "en SF: lookup, NUNCA texto libre (adv. 6)"`  
        `string banco "Pershing/Inviu/Morgan/..."`  
        `string tipologia "Referente/Asesor"`  
        `date inicio_acuerdo "fecha de registro = alta"`  
        `string moneda`  
    `}`

### 1.7 Sistemas externos

`erDiagram`  
    `DATAHUB_GCP ||--o{ INVOICES : "divisa e inflacion (R2)"`  
    `DATAHUB_GCP ||--o{ PAGOS : "tipo de cambio del dia"`  
    `DATAHUB_GCP ||--o{ COMISIONES_LEDGER : "resultados del calculo"`  
    `ADDEPAR ||--o{ CLIENTES : "patrimonio (widget AppExchange)"`  
    `CONTABILIUM ||--o{ INVOICES : "comprobantes / ERP ids"`  
    `CONTABILIUM ||--o{ SERVICIOS_ERP : "catalogo 1a1"`

## 2\. El modelo en prosa — las lógicas que el diagrama no cuenta

### El cliente y su gente

Un **cliente** es, en la práctica, un **conglomerado de tomadores de decisión**, que están registrados como **personas**. El tablero de Clientes guarda a los tomadores de decisión como filas hijas, cada una vinculada a su registro en Personas. Personas es además el repositorio KYC: documentos, pasaportes, condición de PEP, profiling y respaldo de origen de fondos.

#### Tres relaciones distintas que conviene no confundir

Esta es la aclaración más importante del modelo, porque las tres involucran personas pero **no unen las mismas entidades**:

| Relación | Une | Qué expresa |
| :---- | :---- | :---- |
| **Tomador de decisión** | Persona ↔ **Cliente** | quién decide por el cliente |
| **Beneficiario** | Persona ↔ **Cliente** | quién se beneficia de lo que el cliente tiene |
| **Familiar** | Persona ↔ **Persona** | el parentesco entre dos personas |

**Los familiares NO son una relación con el cliente.** Es un vínculo de persona a persona: la familia se mapea *sobre* las personas —típicamente sobre los tomadores de decisión— que tienen a su cónyuge, hijos o padres vinculados como otros registros de Personas, cada uno con su parentesco. Un familiar solo llega a estar relacionado con un cliente indirectamente, a través de la persona con la que está emparentado.

**El beneficiario, en cambio, sí es una relación persona ↔ cliente**, del mismo tipo que la de tomador de decisión, y hoy **no existe en Monday**: es un requisito nuevo para el diseño de Salesforce.

Se pide a este nivel —beneficiario *del cliente*, sin más precisión— a propósito. Un beneficiario real siempre lo es de **algo concreto** (de una cuenta bancaria, del alquiler de una propiedad, de una tarjeta), pero esa granularidad todavía no está relevada. Registrarlo a nivel cliente permite empezar a documentar quiénes son, y más adelante especializar la relación hacia el beneficio puntual sin rehacer el modelo.

> **Para el diseño:** la relación debe existir aunque **hoy no haya datos que migrar** — no se está cargando en ningún lado. Conviene modelarla desde el inicio (con un campo de tipo/rol que después admita "beneficiario de cuenta", "beneficiario de alquiler", etc.) para no tener que agregarla en caliente.

Los clientes también pueden agruparse entre sí (relación *conglomerado*), y cada cliente tiene su **categoría** (Black, Diamond, Platinum, Gold…), su **asesor** (Adduntier) y su **staff asignado por vertical**.

### Titularidades: no confundir propiedad con relación

Las **cuentas bancarias** tienen titulares y firmantes que **pueden ser personas o sociedades** (cada firmante declara su "personería"). Las **sociedades** tienen accionistas y autoridades que también **pueden ser personas u otras sociedades**, con su rol (presidente, director, member, managing member…) y su participación.

Las cuentas bancarias están **relacionadas con los clientes**, pero la titularidad **no necesariamente coincide** con esa relación: un cliente puede tener un tomador de decisión que no sea titular de ninguna de las cuentas que el cliente tiene registradas. Propiedad (quién es titular) y relación comercial (de qué cliente es la cuenta) son dos ejes distintos y el modelo debe mantenerlos separados.

### Servicios, abonos y facturación

Cada cliente tiene **abonos asignados a distintos servicios**. Los servicios producen abonos que se gestionan en **Invoices**:

- **Family Office** se contrata **por familia**: aplica a todas las sociedades del cliente. Define monto, moneda, frecuencia de pago y regla de actualización (inflación mensual/trimestral/semestral, MEP, ajuste personalizado).  
- **Cash Management** y **Mantenimiento de Sociedades Exterior** se contratan **por sociedad**, cada una con su propio abono, frecuencia y ajuste.  
- Los tres servicios recurrentes **generan automáticamente** el ítem en Invoices según la frecuencia de pago; la automatización actualiza la fecha del próximo pago.  
- **Trámites de sociedades** es el caso **manual**: el trámite corre su ciclo (tipo de trámite, urgencia, jurisdicción) y recién al presionar el botón **"finalizar"** dispara la factura.

**Invoices es el hub**: cada factura referencia al cliente, a la sociedad facturada, al servicio que la originó, al tarifario (precio y **costo Adduntia**, que es margen y necesita destino propio) y a sus **cobranzas como subitems** — con moneda de contrato vs. moneda de recibido, caja de destino y el tipo de cambio del día que provee el Datahub. El comprobante se emite en **Contabilium** y la factura guarda los IDs del ERP; el catálogo de **Servicios ERP** casa 1 a 1 con el de Contabilium.

### El trámite de sociedades y su detalle operativo

Los trámites cubren la creación y el mantenimiento de sociedades por jurisdicción (**LLC, BVI, NEVIS**: creación, good standing, apostillados, certificados, EIN…). El detalle operativo de cada jurisdicción —hoy workflows propios con sus estados— **sobrevive como insumo del proceso**: es la especificación de pasos y estados que Vurpix necesita para modelar el ciclo del trámite en Salesforce. El trámite consume el catálogo de servicios y factura al finalizar.

### El funnel comercial

Un **referente** (abogado, contador, cliente, evento, networking) **origina leads**. El lead avanza por el funnel —primera, segunda, tercera reunión— con su agenda registrada en **Activities**, hasta **Ganado** (se convierte en cliente, con su categoría y regla de inicio de cobro) o Perdido/Dormant. El lead carga patrimonio estimado, tipo de firma y qué servicios contrataría (Family Office, Banca Privada, Cash Management, Tax Planning).

### La operativa diaria

- **Tickets de transferencias**: mueven dinero entre cuentas (del cliente, de financieras, cables, arbitrajes), con cuenta de origen y destino contra Cuentas Bancarias, montos brutos y netos, y el costo/margen de Adduntia por operación.  
- **La ticketera** (hoy Customer Success): seguimiento interno de los temas y tareas que se tienen con cada cliente, por área y tipología, con prioridad y vencimiento. Requisito R3: debe soportar tickets de visibilidad restringida hasta que su owner los libere. Requisito R4: no mezclar su tipificación con los tickets de operación.  
- **Tickets de operaciones**: caso especial — un pedido del cliente que se deriva a la mesa con parámetros específicos (CUSIP, buy/sell, cantidad, precio, comisión, cuenta). El tablero actual sirve como referencia de campos; el alcance de implementación lo define Franco.

### Comisiones

Cada cliente tiene **acuerdos de comisión** con Adduntiers (referentes o asesores) por banco. El **ledger histórico** guarda la serie completa de acuerdos: un cambio en el acuerdo vigente crea una fila nueva cuya fecha de registro es el inicio del nuevo acuerdo (requisito R1 — vigencia temporal, nunca sobrescribir). El **cálculo** de comisiones corre en el **Datahub (GCP)** con sus parámetros por banco (base revenue, moneda, share) y no se reimplementa en Salesforce: SF guarda el ledger y recibe los resultados.

### Lo que llega de afuera

- **Datahub (GCP)**: divisa e inflación como datos de referencia para los cálculos de invoices (conector **entrante** — requisito R2, verificar SOW), el tipo de cambio del día para las cobranzas, y los resultados del cálculo de comisiones.  
- **Addepar**: el patrimonio de las familias (posiciones, cuentas financieras) vía widget de AppExchange.  
- **Contabilium**: emisión de comprobantes y catálogo de servicios.

## 3\. Leyenda de entidades → candidato Salesforce

| Entidad (tablero Monday) | ID | Candidato SF | Nota de diseño |
| :---- | :---- | :---- | :---- |
| **Clientes** | 6645107029 | `Account` (Household/ARC) | Maestra \+ proceso de alta embebido (R5). Subitems \= tomadores de decisión. |
| **Personas** | 6499162579 | `Contact` | KYC: DNI/pasaporte, PEP, profiling. Subitems \= **familiares: vínculo persona↔persona**, no con el cliente (ver §2). |
| **Beneficiarios** | *(no existe en Monday)* | Junction `Contact`↔`Account` con rol | **Requisito nuevo.** Relación persona↔cliente, hermana de "tomador de decisión". Sin datos hoy; se pide que exista en el diseño (ver §2). |
| **Sociedades** | 6603601229 | `Account` (record type Sociedad) u objeto propio | Radicación LLC/NEVIS/BVI/ARG/UY. Subitems \= accionistas/autoridades (persona o sociedad, % y rol). |
| **Cuentas Bancarias** | 6645481045 | Objeto propio / `FinancialAccount` (FSC) | Subitems \= titulares/firmantes (persona o sociedad). Enum Banco \= lista maestra. |
| **Leads** | 8496138428 | `Lead` \+ `Opportunity` | Funnel completo: intake → 3 reuniones → Ganado/Perdido. |
| **Referrals** | 8699280312 | `Lead` source / objeto Referral | Referentes que traen leads; parte del proceso de hunting. |
| **Activities** | 8496138511 | `Task`/`Event` estándar | Soporte — no modelar objeto propio. |
| **Invoices (= "Cobros")** | 4351812987 | Objeto `Invoice` \+ integración Contabilium | Hub de facturación. Las relaciones de otros tableros lo nombran "Cobros". |
| **Pagos (subitems de Invoices)** | 4370850329 | Objeto `Payment`/`Collection` con ID único | Hoy sin unicidad — la dedup la hace GCP (ver doc "Bandaids"). |
| **Family Office** | 6645439632 | Suscripción/Servicio | Abono por familia; genera invoices automáticas por frecuencia. |
| **Cash Management** | 6645453104 | Suscripción/Servicio | Abono por sociedad; generación automática. |
| **Mant. Sociedades Exterior** | 7235288183 | Suscripción/Servicio | Abono por sociedad; generación automática. |
| **Trámites sociedades ext.** | 4351811684 | `Case`/Order de trámite | Generación **manual** a Invoices (botón finalizar). Su detalle operativo por jurisdicción (workflows LLC/BVI/NEVIS, hoy tableros propios) es **insumo de diseño del proceso** y debe exportarse antes del descomisionado. |
| **Customer Success → Ticketera** | 8373364383 | `Case` | Referencia de diseño de la ticketera. R3 y R4 aplican. |
| **Tickets de Transferencias** | 8477353225 | `Case` (record type Transferencia) | Origen/destino contra Cuentas Bancarias; márgenes por operación. |
| **Tickets de Operaciones** | 8477471748 | Referencia de campos (Franco define) | Nunca usado en producción. Sin datos que migrar. |
| **Comisiones Adduntiers Históricos** | 8463662088 | Objeto Acuerdo de Comisión (ledger, R1) | Sistema de registro. Vigencia temporal obligatoria. |
| **Comisiones Adduntiers** | 8332390643 | Vista "acuerdo vigente" sobre el ledger | La cara editable de R1, no una tabla independiente. |
| **Cálculo Comisiones** | 8957411980 | **No migra** — queda en el Datahub | Parámetros del cálculo que sigue corriendo en GCP. SF guarda el ledger y recibe resultados. |
| **Staff** | 6521013827 | Asignación (junction Account↔User) | Staff por vertical y cliente. |
| **Adduntiers** | 6831751106 | `User` / lista de asesores | Referencia de registro. |
| **Servicios ERP** | 9179959643 | `Product2` / catálogo | Códigos casan 1 a 1 con Contabilium. |
| **Tarifario** | 4430928306 | `PriceBook` \+ campo costo | `Costo Adduntia` es margen: no existe en PriceBook estándar. |

**Externos:** `DATAHUB_GCP` (divisa/inflación — conector **entrante**, R2, verificar SOW; tipo de cambio de cobranzas; resultados del cálculo de comisiones) · `ADDEPAR` (patrimonio de las familias vía widget AppExchange) · `CONTABILIUM` (ERP: comprobantes y catálogo).

## 4\. Advertencias de lectura

1. Cardinalidades: las marcadas salen del uso conocido; Monday no declara cardinalidad — validar en el diseño de detalle.  
2. Los campos `mirror` del workspace son lookups derivados; en SF se resuelven con fórmulas/relaciones, no como campos propios.  
3. El enum `Banco` está repetido en 3 tableros (Cuentas Bancarias, Comisiones, Cálculo) → en SF debe ser **una** lista maestra.  
4. El enum `Concepto` de Invoices es, en la práctica, el catálogo de servicios facturables — cruzarlo contra Servicios ERP al modelar `Product2`.  
5. **Semántica de monedas — el enum no alcanza para saber en qué moneda está un importe.** Verificado contra los datos (15/8): en un contrato con moneda `MEP`, el campo de **precio de contrato está en dólares** mientras que el de **precio ajustado ya está pesificado**. Dos importes de la misma factura, con el mismo valor de enum, en monedas distintas — la diferencia solo se conoce por convención. Lo mismo del lado del cobro: `MEP` como moneda recibida significa "pesos liquidados al tipo de cambio MEP". Consecuencia para el diseño: **cada importe debe declarar su propia moneda de forma explícita**, y la vía de liquidación (MEP/CCL/BNA/caja) debe ser un dato separado del monto. Es la causa raíz de errores de conversión ya ocurridos.  
6. **Identidad del adduntier/referente por clave, no por nombre.** El ledger de comisiones guarda hoy el referente como texto (nombre completo en unos registros, nombre de pila en los históricos), lo que ya genera falsos positivos en los controles de "referentes faltantes" del pipeline. En SF el acuerdo de comisión debe referenciar al asesor por lookup a `User`/registro maestro — la migración del ledger requiere una tabla de mapeo nombre→identidad.  
7. **Cuentas de terceros en transferencias.** Los tickets de transferencias referencian cuentas de origen/destino que no siempre son cuentas de clientes (financieras, cables, arbitrajes). El catálogo de cuentas bancarias en SF debe admitir cuentas propias de Adduntia y de terceros, o las transferencias quedan sin referencia válida.

---

*Generado a partir de `esquemas_completo.csv` (export API Monday, 3/7/2026) y el sheet "Detalle esquemas tableros monday" (actualizado 12/8), con las definiciones de alcance de Max del 12–13/8. Complementa el documento "Bandaids actuales por limitaciones de Monday" (requisitos de integración) y el acta de validación del cruce.*  
