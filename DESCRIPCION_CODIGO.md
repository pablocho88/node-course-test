# Descripción del Código del Repositorio

## Resumen General

Este repositorio contiene una **API REST desarrollada en Node.js** que gestiona un sistema de administración de cursos y evaluaciones técnicas para desarrolladores. Es un proyecto de examen diseñado por Fiqus/Cambá para evaluar conocimientos en desarrollo backend con Node.js.

## Arquitectura de la Aplicación

### Stack Tecnológico

- **Runtime**: Node.js
- **Framework Web**: Express.js
- **Base de Datos**: MongoDB (usando Mongoose como ODM)
- **Testing**: Mocha + Chai + Supertest
- **HTTP Client**: Request (para llamadas HTTP internas)
- **Logging**: Morgan
- **Linting**: ESLint

### Estructura del Proyecto

```
node-course-test/
├── app.js                  # Punto de entrada principal de la aplicación
├── routes.js              # Definición de todas las rutas de la API
├── config/                # Configuraciones por ambiente (dev, test, prod)
├── controllers/           # Lógica de negocio de cada entidad
├── models/               # Modelos de datos de MongoDB/Mongoose
├── middleware/           # Middlewares personalizados (cache, helpers)
├── services/             # Servicios externos (mock de AFIP)
├── test/                 # Tests automatizados
└── dump/                 # Datos de ejemplo para MongoDB
```

## Modelos de Datos

El sistema maneja las siguientes entidades principales:

### 1. **Technology** (Tecnología)
Representa las tecnologías sobre las que se dictan cursos (ej: NodeJS, VueJS, Java).
- `name`: Nombre legible de la tecnología
- `technologyId`: Identificador único tipo slug

### 2. **Course** (Curso)
Representa un curso con información completa sobre fechas, precio, estado, etc.
- `technologyId`: Referencia a la tecnología del curso
- `date`: Rango de fechas (from/to)
- `description`: Descripción del curso
- `status`: Estado del curso (new, registration open, ongoing, finished)
- `classes`: Array de clases del curso (embebido)
- `students`: Array de IDs de estudiantes inscritos
- `price`: Precio del curso

### 3. **CourseClass** (Clase de Curso)
Representa una clase individual dentro de un curso (subdocumento embebido).
- `name`: Nombre de la clase
- `description`: Descripción
- `content`: Contenido de la clase
- `tags`: Etiquetas opcionales

### 4. **Student** (Estudiante/Desarrollador)
Representa a los alumnos que toman cursos.
- `firstName`: Nombre
- `lastName`: Apellido
- `billingAddress`: Dirección de facturación (subdocumento)
- `creditCards`: Array de tarjetas de crédito (subdocumento)

### 5. **Address** (Dirección)
Subdocumento que representa una dirección de facturación.
- `street1`, `street2`, `city`, `state`, `zipCode`, `country`

### 6. **CreditCard** (Tarjeta de Crédito)
Subdocumento con información de tarjeta de crédito.
- `firstName`, `lastName`
- `last4Numbers`: Últimos 4 dígitos
- `creditCardAPIToken`: Token de API para procesamiento
- `isDefault`: Si es la tarjeta por defecto

### 7. **Evaluation** (Evaluación)
Representa una evaluación asociada a un curso.
- `courseId`: ID del curso evaluado
- `date`: Rango de fechas de la evaluación
- `abstract`: Resumen de la evaluación
- `notes`: Array de notas de estudiantes (subdocumento)

### 8. **EvaluationStudent** (Nota de Estudiante)
Subdocumento que representa la calificación de un estudiante.
- `studentId`: ID del estudiante
- `qualification`: Calificación numérica
- `status`: Estado (passed/failed)

## Controladores y Funcionalidades

### 1. **GenericController**
Controlador base que implementa operaciones CRUD estándar:
- `list()`: Listar todos los registros con filtros opcionales
- `create()`: Crear nuevo registro
- `read()`: Obtener un registro por ID
- `update()`: Actualizar un registro existente
- `remove()`: Eliminar un registro

### 2. **CoursesController**
Gestión de cursos con funcionalidades adicionales:
- Filtrado por `status` y `technologyId`
- Validación de fechas
- Gestión de clases embebidas

### 3. **StudentsController**
Gestión de estudiantes (usa el controlador genérico).

### 4. **EvaluationsController**
Gestión de evaluaciones (usa el controlador genérico).

### 5. **TechnologiesController**
Gestión de tecnologías (usa el controlador genérico).

### 6. **BillingController**
Controlador especializado para facturación con dos endpoints principales:

#### `getChargeableStudents()`
Obtiene los estudiantes que deben ser facturados:
1. Busca cursos con estado "finished"
2. Obtiene las evaluaciones de esos cursos
3. Filtra estudiantes que aprobaron (status: "passed")
4. Agrupa por estudiante y suma los precios de todos los cursos aprobados
5. Devuelve información de facturación (nombre, apellido, dirección, precio total)

#### `getInvoices()`
Genera facturas usando el servicio de AFIP:
1. Obtiene estudiantes a facturar (usando `getChargeableStudents`)
2. Para cada estudiante, llama al servicio AFIP para generar factura
3. Implementa reintentos automáticos en caso de fallo
4. Devuelve array de facturas generadas con número, datos del estudiante y monto

### 7. **StatsController**
Controlador de estadísticas.

#### `failuresByStates()`
Genera estadísticas de reprobados por estado:
1. Busca evaluaciones con estudiantes reprobados
2. Obtiene los IDs de estudiantes con status "failed"
3. Consulta información de estudiantes para obtener sus direcciones
4. Agrupa y cuenta reprobados por estado (state)
5. Devuelve objeto con formato `{Estado: Cantidad}`

## Servicios

### **AFIP Mock API** (`services/afip-mock-api.js`)
Simula un servicio de facturación de AFIP (Administración Federal de Ingresos Públicos de Argentina):
- Valida estructura de datos de factura (nombre, dirección, importe)
- Genera números de factura secuenciales
- Simula fallos aleatorios (25% de probabilidad) para testear reintentos
- Devuelve ID de factura generada o error

## Middleware

### 1. **responseHelpers**
Agrega métodos helper al objeto `res` para estandarizar respuestas:
- `response200(data, message)`: Respuesta exitosa
- `response404(message)`: Not Found
- `response500(error, message)`: Error del servidor

### 2. **cacheMiddleware**
Implementa un sistema de caché en memoria para requests GET:

**Funcionamiento:**
- Almacena respuestas de GET en un objeto global (`globalCache`)
- En requests GET subsecuentes, retorna datos del cache si existen
- En POST/PUT, invalida entradas relacionadas en el cache:
  - POST: Invalida la lista y queries relacionadas
  - PUT: Invalida el recurso específico y la lista
- Permite configurar rutas a ignorar vía `config.ignoreCacheRoutes`

**Ventajas:**
- Mejora performance evitando consultas redundantes a la DB
- Reduce latencia en endpoints frecuentemente consultados

**Limitaciones:**
- Cache solo en memoria (se pierde al reiniciar)
- No tiene TTL (Time To Live)
- No es adecuado para ambientes con múltiples instancias

## Rutas de la API

La API está montada en el path base `/api` y expone:

### Rutas CRUD Genéricas (para cada entidad):
```
GET    /api/courses          # Listar cursos (soporta ?status=X&technologyId=Y)
POST   /api/courses          # Crear curso
GET    /api/courses/:id      # Obtener curso por ID
PUT    /api/courses/:id      # Actualizar curso
DELETE /api/courses/:id      # Eliminar curso

GET    /api/students         # Listar estudiantes
POST   /api/students         # Crear estudiante
GET    /api/students/:id     # Obtener estudiante
PUT    /api/students/:id     # Actualizar estudiante
DELETE /api/students/:id     # Eliminar estudiante

GET    /api/evaluations      # Listar evaluaciones
POST   /api/evaluations      # Crear evaluación
GET    /api/evaluations/:id  # Obtener evaluación
PUT    /api/evaluations/:id  # Actualizar evaluación
DELETE /api/evaluations/:id  # Eliminar evaluación

GET    /api/technologies     # Listar tecnologías
POST   /api/technologies     # Crear tecnología
GET    /api/technologies/:id # Obtener tecnología
PUT    /api/technologies/:id # Actualizar tecnología
DELETE /api/technologies/:id # Eliminar tecnología
```

### Rutas Especializadas:
```
GET  /api/admin/billing/getChargeableStudents  # Obtener estudiantes a facturar
GET  /api/admin/billing/getInvoices            # Generar y obtener facturas
POST /api/afip                                 # Mock de servicio AFIP
GET  /api/stats/failuresByStates              # Estadísticas de reprobados por estado
```

## Flujo de Datos Principal

### Flujo de Facturación:
1. **Finalización de Curso**: Curso cambia a status "finished"
2. **Evaluación**: Se registra evaluación con notas de estudiantes
3. **Cálculo de Facturables**: 
   - Sistema identifica estudiantes que aprobaron cursos finalizados
   - Agrupa y suma precios de todos los cursos aprobados por estudiante
4. **Generación de Facturas**:
   - Se obtiene lista de estudiantes a facturar
   - Para cada uno se genera factura vía servicio AFIP
   - Se implementan reintentos automáticos si AFIP falla
5. **Respuesta**: Se devuelve array de facturas con número, datos y monto

### Flujo de Estadísticas:
1. **Consulta de Evaluaciones**: Se buscan evaluaciones con reprobados
2. **Extracción de Estudiantes**: Se identifican IDs de estudiantes con status "failed"
3. **Obtención de Ubicaciones**: Se consultan direcciones de facturación
4. **Agregación**: Se cuentan reprobados agrupados por estado
5. **Respuesta**: Objeto con formato `{Estado: Cantidad}`

## Testing

El proyecto incluye tests automatizados en `test/dummy_tests.js`:

**Setup de Tests:**
- Limpia colecciones antes de cada ejecución
- Crea datos de prueba (tecnologías, estudiantes, cursos, evaluaciones)
- Inicializa contadores de facturación

**Tests Implementados:**
- Validación de endpoints de facturación
- Verificación de cálculos de estudiantes facturables
- Validación de generación de facturas
- Tests de estadísticas por estado
- Verificación de filtros en endpoints de cursos

## Características Técnicas Destacadas

### 1. **Promesas y Async/Await**
El código usa extensivamente Promises y async/await para manejo asincrónico:
```javascript
Course.find({status: "finished"})
  .then((courses) => { /* ... */ })
  .then((evaluations) => { /* ... */ })
  .catch((err) => { /* ... */ });
```

### 2. **Patrón de Controlador con Factory**
Los controladores son funciones factory que reciben dependencias:
```javascript
module.exports = (mongoose, request, config) => {
  // Lógica del controlador
  return { method1, method2 };
};
```

### 3. **Validación de Mongoose**
Uso de validadores de Mongoose con `runValidators: true` en updates.

### 4. **Manejo de Errores Estandarizado**
Todos los endpoints usan los response helpers para respuestas consistentes.

### 5. **Sistema de Reintentos**
Implementación recursiva de reintentos en llamadas a AFIP:
```javascript
if (data.status === 'success') {
  return resolve(data);
}
// Si falla, reintenta recursivamente
_generateInvoiceAFIP(student, url, json)
  .then(data => resolve(data))
  .catch(err => reject(err));
```

## Configuración por Ambiente

El sistema soporta tres ambientes:
- **dev**: Desarrollo local
- **test**: Ejecución de tests
- **prod**: Producción

Cada ambiente tiene su propia configuración de:
- Puerto de escucha
- URL de base de datos MongoDB
- Rutas a ignorar en cache

## Propósito del Repositorio

Este repositorio es un **examen práctico** para el curso de Node.js de Fiqus. Los ejercicios propuestos incluyen:

1. Agregar filtro por `technologyId` en el endpoint GET de cursos
2. Crear endpoint GET para obtener facturas
3. Implementar middleware de caché para requests GET
4. Crear endpoint de estadísticas de reprobados por estado

**Criterios de Evaluación:**
- Todos los ejercicios deben estar resueltos
- Todas las funcionalidades deben estar testeadas
- Tests deben pasar exitosamente
- No debe haber errores de linting
- Entrega mediante un único Pull Request

## Conclusión

El código implementa una API REST completa y bien estructurada para gestionar cursos, estudiantes, evaluaciones y facturación. Demuestra buenas prácticas de Node.js/Express incluyendo separación de responsabilidades, manejo de errores, testing, y patrones de diseño modernos. El sistema es funcional y sirve como base sólida para aprender desarrollo backend con el stack MEAN (MongoDB, Express, Angular/Vue, Node.js).
