# Requerimientos del Sistema IoT de Seguridad

## Épicas

### EP-01: Infraestructura AWS IoT
**Como** administrador del sistema  
**Quiero** tener una infraestructura cloud escalable  
**Para** gestionar dispositivos IoT de seguridad de manera confiable

### EP-02: Backend de Gestión
**Como** operador de seguridad  
**Quiero** un backend que administre dispositivos y datos  
**Para** centralizar la información de seguridad

### EP-03: Visualización y Control
**Como** usuario del sistema  
**Quiero** ver los dispositivos en un mapa con geocercas  
**Para** monitorear la seguridad espacialmente

## Historias de Usuario

### AWS IoT (EP-01)

#### HU-01: Configuración de AWS IoT Core
**Como** desarrollador  
**Quiero** configurar AWS IoT Core mediante SDK  
**Para** gestionar las conexiones MQTT de dispositivos

**Criterios de aceptación:**
- El sistema debe usar AWS SDK para crear y configurar todos los recursos AWS
- Debe implementarse un servicio que centralice la creación de recursos AWS
- Los recursos deben crearse solo si no existen previamente

#### HU-02: Gestión de Certificados
**Como** administrador  
**Quiero** crear y gestionar certificados para dispositivos  
**Para** garantizar conexiones seguras

**Criterios de aceptación:**
- Cada dispositivo debe tener un certificado único o compartir uno por tipo
- Los certificados deben almacenarse de forma segura
- El sistema debe asociar automáticamente certificados a políticas y things

#### HU-03: Reglas IoT y CloudWatch
**Como** administrador  
**Quiero** configurar reglas IoT y CloudWatch con retención de 3 días  
**Para** procesar mensajes y mantener logs optimizados

**Criterios de aceptación:**
- Configurar retención de logs de CloudWatch a 3 días exactamente
- Crear reglas IoT que procesen mensajes según patrones específicos
- Implementar manejo de errores en las reglas IoT

### Backend (EP-02)

#### HU-04: Modelo de Datos PostgreSQL
**Como** desarrollador  
**Quiero** implementar un esquema PostgreSQL con Prisma  
**Para** almacenar información de dispositivos y lugares

**Criterios de aceptación:**
- Implementar el esquema definido con Prisma
- Crear índices para optimizar consultas frecuentes
- Incluir relaciones entre lugares, dispositivos y topics

#### HU-05: API REST de Lugares
**Como** operador  
**Quiero** APIs para gestionar lugares con geocercas  
**Para** definir zonas de monitoreo

**Criterios de aceptación:**
- Implementar CRUD completo para lugares
- Permitir definir geocercas en formato GeoJSON
- Optimizar consultas de dispositivos por lugar

#### HU-06: API REST de Dispositivos
**Como** operador  
**Quiero** APIs para registrar y administrar dispositivos  
**Para** mantener control del inventario de sensores

**Criterios de aceptación:**
- Implementar CRUD completo para dispositivos
- Registrar dispositivos en AWS IoT automáticamente
- Asociar dispositivos a lugares y topics

#### HU-07: Comunicación con Erlang
**Como** desarrollador  
**Quiero** integrar el backend con el servidor Erlang existente  
**Para** enviar eventos en tiempo real

**Criterios de aceptación:**
- Implementar cliente WebSocket para comunicación con Erlang
- Manejar reconexiones automáticas
- Implementar cola de mensajes para garantizar entrega

### Frontend (EP-03)

#### HU-08: Visualización de Mapa
**Como** usuario  
**Quiero** ver un mapa interactivo con MapLibre GL  
**Para** ubicar espacialmente los dispositivos

**Criterios de aceptación:**
- Implementar mapa usando MapLibre GL
- Mostrar marcadores de lugares y dispositivos
- Permitir navegación y zoom en el mapa

#### HU-09: Gestión de Geocercas
**Como** usuario  
**Quiero** visualizar geocercas de los lugares  
**Para** delimitar áreas de monitoreo

**Criterios de aceptación:**
- Mostrar geocercas como polígonos en el mapa
- Visualizar dispositivos dentro de sus geocercas correspondientes
- Implementar colores según estado de dispositivos

#### HU-10: Creación de Dispositivos
**Como** usuario  
**Quiero** un formulario para crear dispositivos asociados a lugares  
**Para** registrar nuevos sensores en el sistema

**Criterios de aceptación:**
- Implementar modal con formulario de creación
- Permitir seleccionar tipo de dispositivo
- Mostrar información de certificados generados

## Tareas Técnicas

### TT-01: Configuración AWS
- Implementar servicio para crear recursos AWS mediante SDK
- Configurar retención de logs en CloudWatch a 3 días
- Crear políticas IoT por tipo de dispositivo

### TT-02: Esquema de Base de Datos
- Implementar modelos de datos con Prisma
- Crear índices para optimizar consultas frecuentes
- Configurar relaciones entre entidades

### TT-03: Simulador de Dispositivos
- Desarrollar un módulo de simulación de dispositivos
- Implementar generación de datos según tipo de dispositivo
- Crear función para simular múltiples dispositivos simultáneamente

### TT-04: Optimizaciones
- Implementar consultas SQL optimizadas para Prisma
- Configurar particionamiento para datos históricos
- Establecer estrategias para reducir costos en AWS

## Requisitos No Funcionales

### RNF-01: Seguridad
- Toda comunicación debe ser encriptada mediante TLS
- Implementar autenticación JWT para APIs REST
- Utilizar políticas específicas por tipo de dispositivo

### RNF-02: Rendimiento
- El sistema debe soportar al menos 1000 dispositivos simultáneos
- Las respuestas de API deben ser menores a 500ms
- El mapa debe cargar en menos de 3 segundos

### RNF-03: Disponibilidad
- El sistema debe tener una disponibilidad de 99.9%
- Implementar manejo de errores y reintentos en toda comunicación
- Documentar procedimientos de recuperación

### RNF-04: Escalabilidad
- La arquitectura debe permitir escalado horizontal
- Implementar particionamiento para datos históricos
- Optimizar consultas para grandes volúmenes de datos