[README.md](https://github.com/user-attachments/files/28444023/README.md)
# Huellitas — Sistema de Gestión de Pedidos Veterinaria

Sistema distribuido de procesamiento de órdenes de compra para una clínica
veterinaria, implementado con **Java 11**, **gRPC** como middleware y **SQLite**
como base de datos.

---

## Arquitectura de Capas

```
┌─────────────────────────────────────────────────────────────┐
│               CLIENTE gRPC (HuellitasClientTest)            │
└────────────────────────────┬────────────────────────────────┘
                             │  Protobuf / gRPC
┌────────────────────────────▼────────────────────────────────┐
│          MIDDLEWARE — OrdenGrpcServiceImpl                   │
│   Serialización · Routing · Manejo de errores distribuidos  │
└────────────────────────────┬────────────────────────────────┘
                             │  llama a →
┌────────────────────────────▼────────────────────────────────┐
│          CAPA DE NEGOCIO — OrdenService                      │
│   Regla 1: Validación de stock                              │
│   Regla 2: IVA 16%  +  Descuento 10% si subtotal > $1,500  │
│   Regla 3: CREADA → VALIDADA → APROBADA / RECHAZADA         │
└────────────────────────────┬────────────────────────────────┘
                             │  usa →
┌────────────────────────────▼────────────────────────────────┐
│          PERSISTENCIA — SQLite  (huellitas.db)               │
│   clientes (20) · productos (20) · ordenes (20)             │
└─────────────────────────────────────────────────────────────┘
```

---

## Requisitos

| Herramienta | Versión mínima |
|-------------|---------------|
| Java JDK    | 11            |
| Apache Maven| 3.8           |

---

## Compilar y ejecutar

### 1. Compilar (genera stubs gRPC desde el .proto)

```bash
mvn clean compile
```

### 2. Iniciar el servidor gRPC (terminal 1)

```bash
mvn exec:java -"Dexec.mainClass="com.huellitas.HuellitasServer"
```

El servidor escucha en el puerto **50051** y crea `huellitas.db` con los
20 registros de cada tabla en el primer arranque.

### 3. Ejecutar las pruebas de integración (terminal 2)

```bash
mvn exec:java -"Dexec.mainClass="com.huellitas.HuellitasClientTest"
```

### 4. (Opcional) Crear JAR ejecutable

```bash
mvn package
java -jar target/huellitas-1.0-SNAPSHOT.jar          # servidor
java -cp target/huellitas-1.0-SNAPSHOT.jar com.huellitas.HuellitasClientTest
```

---

## Reglas de negocio

| # | Regla | Detalle |
|---|-------|---------|
| 1 | Validación de stock | Se rechaza la orden si `cantidad > stock` disponible |
| 2 | IVA + Descuento | IVA = 16% sobre la base imponible. Si subtotal > $1,500 se aplica 10% de descuento antes del IVA |
| 3 | Estados | `CREADA → VALIDADA → APROBADA` ó `RECHAZADA` (sin saltar estados) |

---

## Casos de prueba incluidos

| Caso | Entrada | Resultado esperado |
|------|---------|-------------------|
| 1 – Orden válida | CLI-001, PRD-001, qty=2 | APROBADA, total $580 |
| 2 – Sobregiro | CLI-004, PRD-007, qty=999 | RECHAZADA (stock insuficiente) |
| 3 – ID inválido | CLI-999 (no existe) | gRPC INVALID\_ARGUMENT |
| BONUS – Descuento | CLI-003, PRD-015, qty=2 | APROBADA con 10% descuento |

---

## Productos disponibles

| ID | Nombre | Tipo | Precio | Stock |
|----|--------|------|--------|-------|
| PRD-001 | Vacuna Antirrábica Canina | VACUNA | $250 | 150 |
| PRD-002 | Vacuna DHPP | VACUNA | $320 | 120 |
| PRD-003 | Vacuna Bordetella | VACUNA | $280 | 80 |
| PRD-004 | Vacuna Leptospirosis | VACUNA | $310 | 100 |
| PRD-005 | Vacuna Antirrábica Felina | VACUNA | $230 | 130 |
| PRD-006 | Vacuna Triple Felina | VACUNA | $350 | 90 |
| PRD-007 | Vacuna Leucemia Felina | VACUNA | $400 | 70 |
| PRD-008 | Frontline Plus Perros | MEDICAMENTO | $450 | 200 |
| PRD-009 | NexGard Antipulgas | MEDICAMENTO | $520 | 180 |
| PRD-010 | Heartgard Antiparasitario | MEDICAMENTO | $380 | 160 |
| PRD-011 | Apoquel Antialérgico | MEDICAMENTO | $650 | 100 |
| PRD-012 | Bravecto 3 meses | MEDICAMENTO | $890 | 120 |
| PRD-013 | Drontal Plus | MEDICAMENTO | $180 | 250 |
| PRD-014 | Milbemax Felino | MEDICAMENTO | $290 | 140 |
| PRD-015 | Royal Canin Adulto Grande 15kg | ALIMENTO | $1,200 | 50 |
| PRD-016 | Purina Pro Plan Cachorro 7.5kg | ALIMENTO | $850 | 80 |
| PRD-017 | Hill's Science Diet Senior 6.8kg | ALIMENTO | $950 | 60 |
| PRD-018 | Eukanuba Adulto Mediano 12kg | ALIMENTO | $1,100 | 45 |
| PRD-019 | Pedigree Adulto 15kg | ALIMENTO | $650 | 100 |
| PRD-020 | Whiskas Gato Adulto 7kg | ALIMENTO | $480 | 90 |

---

## Estructura del repositorio

```
huellitas/
├── pom.xml
├── README.md
└── src/
    └── main/
        ├── proto/
        │   └── orden.proto              ← Contrato gRPC
        └── java/com/huellitas/
            ├── domain/                  ← Capa de negocio (sin dependencias de transporte)
            │   ├── model/               ← Entidades: Producto, Cliente, Orden, enums
            │   ├── repository/          ← Interfaces de repositorio
            │   └── service/
            │       └── OrdenService.java ← Las 3 reglas de negocio
            ├── infrastructure/          ← Capa de infraestructura
            │   ├── db/
            │   │   └── DatabaseManager.java  ← SQLite + seed de 20 registros/tabla
            │   ├── repository/          ← Implementaciones JDBC
            │   └── grpc/
            │       └── OrdenGrpcServiceImpl.java  ← Middleware gRPC
            ├── HuellitasServer.java     ← Punto de entrada del servidor
            └── HuellitasClientTest.java ← Script de pruebas de integración
```
