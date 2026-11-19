# Patrón Saga × Database per Service (Módulo M9: Reservas de Viajes)
### Trabajo Práctico de Investigación — Desarrollo de Software (UTN FRRe 2026)
**Grupo 8** · *Tema 11: Patrón Saga* | *Tema 12: Database per Service*

---

## 📌 Resumen Ejecutivo

Este repositorio contiene la **implementación funcional completa, prototipo técnico y suite de verificación de resiliencia (hardening)** de los patrones de arquitectura distribuida **Patrón Saga (Tema 11)** y **Database per Service (Tema 12)**, aplicados al subsistema de movilidad urbana de la cátedra (**Módulo 9: Reservas de Viajes**).

La solución resuelve el problema fundamental de la pérdida del commit global ACID en arquitecturas de microservicios, coordinando la activación de viajes programados a través de transacciones compensatorias locales y preservando la **Consistencia Eventual** sin compartir bases de datos relacionales ni permitir consultas SQL cruzadas.

---

## 🏛️ Decisiones Arquitectónicas Clave

### 1. Database per Service (Tema 12)
* **Aislamiento Físico Estricto:** Cada subsistema posee su propio motor y archivo físico SQLite independiente en la carpeta `data/`:
  - `data/m9_reservas.db`: Ciclo de vida de la reserva, historial inmutable de auditoría y claves de idempotencia.
  - `data/m7_pagos.db`: Transacciones financieras, pre-autorizaciones y liberaciones de fondos.
  - `data/m5_despacho.db`: Solicitudes de despacho, asignación de vehículos y conductores.
* **Cero SQL Cruzado:** Ningún servicio accede a las tablas de otro; toda comunicación inter-servicio se realiza a través de APIs de dominio fuertemente tipadas en TypeScript.

### 2. Patrón Saga Orquestada (Tema 11)
Se seleccionó la variante de **Orquestación Centralizada en M9** (`ActivationSagaOrchestrator`) para centralizar la máquina de estados del negocio y garantizar trazabilidad determinística:
* **$T_1$ (Transacción Local M9):** Bloqueo semántico y transición a `EN_PROCESO_DESPACHO` con Control de Concurrencia Optimista (OCC).
* **$T_2$ (Transacción Compensable M7):** Pre-autorización y retención de fondos financieros en `m7_pagos.db`.
* **$T_3$ (Transacción Pivotante M5 - Punto de No Retorno):** Búsqueda y asignación de chofer cercano.
* **$T_r$ (Transacción Reintentable M8):** Notificación multicanal al pasajero y transición definitiva a `ACTIVADA`.
* **Rollback Lógico ($C_2 \to C_1$):** Si la transacción pivotante falla (ej. sin conductores disponibles), el orquestador ejecuta las compensaciones en orden inverso:
  - **$C_2$ (Compensación M7):** Liberación de fondos (saldo retenido vuelve a \$0).
  - **$C_1$ (Compensación M9):** Transición de la reserva a `VENCIDA` con auditoría inmutable.

### 3. Blindaje de Resiliencia Operativa (Hardening)
* **Candado Atómico Anti-TOCTOU:** Registro atómico `EN_PROCESO` de la clave de idempotencia (`Idempotency-Key`). Peticiones en vuelo concurrentes son rechazadas con `HTTP 409 Conflict`, y reintentos posteriores reciben la respuesta cacheada (`HTTP 200`).
* **Control de Concurrencia Optimista (OCC):** Sentencia SQL `UPDATE reservas SET estado = ? WHERE id = ? AND estado = 'CONFIRMADA'`. Si dos operadoras intentan activar la misma reserva, la segunda es rechazada con `HTTP 409 Conflict`.
* **Compensación Defensiva:** Si el subsistema de pagos (M7) se encuentra caído durante el rollback $C_2$, el orquestador no crashea con error 500: registra la alerta `REQUIERE_RECONCILIACION_MANUAL` en el historial de auditoría y garantiza la ejecución de $C_1$, evitando que la reserva quede congelada en el limbo.
* **Webhooks Asíncronos Tolerantes a Fallos:** Endpoint `/webhooks/despacho` para registrar transiciones de viaje (`VIAJE_INICIADO`, `VIAJE_FINALIZADO`) tolerante a repeticiones de red sin duplicación de eventos.

---

## 📂 Estructura del Repositorio

```text
.
├── Documentacion/                               # Documentación formal de la investigación
│   ├── TPI Investigación.pdf                    # Informe técnico formal (27 páginas)
│   └── Saga × Database per Service — Grupo 8.pptx # Presentación para coloquio (25 slides)
├── data/                                        # Bases de datos SQLite físicamente aisladas
│   ├── m5_despacho.db                           # Base de datos exclusiva de Despacho
│   ├── m7_pagos.db                              # Base de datos exclusiva de Pagos
│   └── m9_reservas.db                           # Base de datos exclusiva de Reservas M9
├── postman/                                     # Colección para pruebas en Postman
│   └── Demo_Saga_Grupo8.postman_collection.json
├── scripts/                                     # Suite de pruebas automatizadas y demos
│   ├── run-demo.js                              # Demostración interactiva con tablas
│   ├── test-success.js                          # Validación de Flujo Exitoso
│   ├── test-failure.js                          # Validación de Fallo y Rollback C2 -> C1
│   ├── test-hardening.js                        # Validación de Concurrencia (TOCTOU, OCC, Defensiva)
│   ├── test-coverage.js                         # Verificación integral (6/6 tests)
│   └── test-reproducir-fallas.js                # Demostración de vulnerabilidades apagando protecciones
├── src/                                         # Código fuente en TypeScript
│   ├── app.ts                                   # Configuración de Express y rutas
│   ├── index.ts                                 # Punto de entrada del servidor
│   ├── clients/                                 # Clientes de integración inter-servicio (M7, M5, M8, M2)
│   ├── config/                                  # Configuración de bases SQLite aisladas
│   ├── controllers/                             # Controladores HTTP (Reservas, Saga, Webhooks, Inspector)
│   ├── domain/                                  # Tipos de dominio, estados y errores personalizados
│   ├── repositories/                            # Capa de persistencia (Reservas, Historial, Idempotencia)
│   ├── saga/                                    # Orquestador central de la Saga de Activación
│   └── utils/                                   # Logger estructurado con requestId
├── Dockerfile                                   # Empaquetado para despliegue en contenedor
├── docker-compose.yml                           # Orquestación de contenedores
├── package.json                                 # Dependencias y scripts de npm
└── tsconfig.json                                # Configuración de compilador TypeScript
```

---

## 🚀 Requisitos e Instalación

* **Node.js:** Versión `>= 22.0.0` (utiliza el módulo nativo `node:sqlite`).
* **npm:** Versión `>= 10.0.0`.

### Instalación de dependencias:
```bash
npm install
```

### Compilación de TypeScript:
```bash
npm run build
```

---

## 🧪 Ejecución de Pruebas y Demostraciones

### 1. Suite Completa de Tests Automatizados (Recomendado)
Compila el proyecto y ejecuta en cadena los 4 scripts de prueba, validando los 6 escenarios críticos:
```bash
npm run test:all
```

### 2. Suite de Hardening y Concurrencia Real
Prueba el comportamiento bajo latencia inyectada y ráfagas paralelas:
```bash
node scripts/test-hardening.js
```
*Valida:*
- Candado Anti-TOCTOU $\to$ `[HTTP 200, HTTP 409]`
- Concurrencia Optimista (OCC) $\to$ `[HTTP 200, HTTP 409]`
- Compensación Defensiva ante caída de M7 $\to$ `Transición garantizada C1 con REQUIERE_RECONCILIACION_MANUAL`.

### 3. Demostración Interactiva en Consola
Ejecuta la demo con tablas formateadas comparando los datos en las 3 bases:
```bash
npm run demo
```

---

## 🌐 Ejecución del Servidor HTTP

Para levantar el servidor API en el puerto `8090`:
```bash
npm run start:dist
```

### Endpoints Principales:

| Método | Endpoint | Descripción |
| :---: | :--- | :--- |
| `POST` | `/reservas` | Crea una nueva reserva programada (valida antelación $\ge 2$ hs según RF-9.6). |
| `GET` | `/reservas` | Lista todas las reservas registradas en M9. |
| `GET` | `/reservas/:id` | Obtiene el detalle de una reserva específica. |
| `GET` | `/reservas/:id/historial` | Consulta el historial inmutable de auditoría (eventos append-only). |
| `POST` | `/reservas/:id/activar-saga` | **Dispara la Saga de Activación.** Soporta header `Idempotency-Key` y payload `{"forzarFalloM5": true}`. |
| `POST` | `/webhooks/despacho` | Recibe callbacks de M5 (`VIAJE_INICIADO`, `VIAJE_FINALIZADO`). |
| `GET` | `/inspeccionar-bases` | **Auditor:** Inspecciona el estado simultáneo de las 3 bases de datos físicas. |
| `GET` | `/health` | Chequeo de salud del servicio y conectividad SQLite. |

---

## 🐳 Ejecución con Docker

Para construir y levantar el prototipo aislado en Docker:
```bash
docker compose up --build
```
El servicio quedará disponible en `http://localhost:8090`.

---

## 👥 Integrantes y Roles de Exposición — Grupo 8 (UTN FRRe 2026)

| # | Integrante | Legajo | Rol en la Investigación y Exposición Oral |
| :-: | :--- | :-: | :--- |
| **1** | **Fernández D'Elia, Sol** | `29286` | Introducción, Evolución Arquitectónica (Monolito a Microservicios) y planteo del problema de consistencia. |
| **2** | **Rosin, Gianna Florencia** | `28951` | Tema 12: Database per Service vs. Antipatrón Monolito Distribuido (Shared Database). |
| **3** | **Salas, Milagros Nadir** | `28816` | Aplicación de Database per Service en M9, Regla de Negocio RF-9.6 (Antelación $\ge 2$ hs) y Prototipo en Capas. |
| **4** | **Alcaraz, Fabricio Nahuel** | `29192` | Ecosistema de M9 (M5 Despacho, M7 Pagos, M8 Notificaciones) y Estrategias de Consulta Inter-Servicio sin JOINs. |
| **5** | **Maciel Sanchez, Gabriel Angel** | `28872` | Tema 11: Adiós al Commit Global (ACID $\to$ CAP/PACELC $\to$ 2PC vs Saga) y Fundamentos del Patrón Saga. |
| **6** | **Benítez, Agustín Nicolas** | `28372` | El Problema de la Falta de Aislamiento (ACID sin 'I') y Variantes de Coordinación (Coreografía vs. Orquestación en M9). |
| **7** | **Lopez, Mateo Jeremias** | `29868` | Circuito de Activación en Código Real: Flujo Exitoso (Happy Path $T_1 \to T_2 \to T_3 \to T_r$). |
| **8** | **Sanabria Cayre, Ernesto Pablo** | `28909` | Manejo de Fallas en Pivotante, Rollback Lógico ($C_2 \to C_1$), Compensación Defensiva y Operador de Terminal. |
| **9** | **Machuca, Avril Micaela** | `29061` | Resiliencia Operativa y Hardening: Candado Atómico Anti-TOCTOU, OCC y Webhooks Idempotentes. |
| **10** | **López, Gustavo Javier** | `28818` | Matriz de Decisión / Trade-offs (Cuándo SÍ / Cuándo NO), Suite de Testing Automatizado y Cierre. |

---

## 📚 Documentación Técnica
Para una explicación exhaustiva de los fundamentos teóricos, comparativas tecnológicas (2PC vs Saga, Coreografía vs Orquestación) y límites arquitectónicos, consultar los documentos incluidos en la carpeta `Documentacion/`:
* [TPI Investigación.pdf](Documentacion/TPI%20Investigación.pdf) — Informe escrito oficial (27 páginas).
* [Saga × Database per Service — Grupo 8.pptx](Documentacion/Saga%20×%20Database%20per%20Service%20—%20Grupo%208.pptx) — Diapositivas de la defensa oral.
