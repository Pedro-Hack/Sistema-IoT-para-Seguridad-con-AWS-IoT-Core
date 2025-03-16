# Requerimientos del Sistema IoT de Seguridad

## Épicas

### EP-01: Infraestructura AWS IoT
**Como** administrador del sistema  
**Quiero** tener una infraestructura cloud escalable  
**Para** gestionar dispositivos IoT de seguridad de manera confiable
**Nota** esto se puede hacer con terraform, incluso uno puede pedirle a una ia que haga eso, no toma mucho tiempo, ejemplos aqui https://github.com/alfonsof/terraform-aws-examples

### EP-02: Backend de Gestión
**Como** operador de seguridad  
**Quiero** un backend que administre dispositivos y datos  
**Para** centralizar la información de seguridad
**Nota** cuales son los datos?, que dispositivos se usan?, cual es la informacion de seguridad?, esta definida esa información o es algo que puede ir cambiando sin previo aviso?

### EP-03: Visualización y Control
**Como** usuario del sistema  
**Quiero** ver los dispositivos en un mapa con geocercas  
**Para** monitorear la seguridad espacialmente
**Nota** se puede hacer con google maps, fuera hay pocas alternativas pero todas son servicios de pago, esto si es mas o menos engorroso y puede llevar una semana en hacerse

## Historias de Usuario

### AWS IoT (EP-01)

#### HU-01: Configuración de AWS IoT Core
**Como** desarrollador  
**Quiero** configurar AWS IoT Core mediante SDK  
**Para** gestionar las conexiones MQTT de dispositivos
**Nota** no es buena idea porque al final se reduce a tener scripts o un servicio donde el error humano por implementacion, por operacion o por diseño es grande, es mejor hacer esto con terraform

**Criterios de aceptación:**
- El sistema debe usar AWS SDK para crear y configurar todos los recursos AWS (Nota: esto lo hace terraform)
- Debe implementarse un servicio que centralice la creación de recursos AWS (Nota: esto lo hace terraform)
- Los recursos deben crearse solo si no existen previamente (nota: esto es complejo, terraform lo resuelve solo)

#### HU-02: Gestión de Certificados
**Como** administrador  
**Quiero** crear y gestionar certificados para dispositivos  
**Para** garantizar conexiones seguras
**Nota** si van a usar aws, esto lo hace IAM y se puede hacer en el CI/CD o con terraform, se puede usar vault o los secretos de google para hacerlo de manera segura, este es un punto de falla muy serio y es mejor usar las herramientas que ya son maduras haciendo esto, aqui hay un ejemplo https://gist.github.com/syntaqx/da18e4c74d96e6764b2a3d14be7db0c6

**Criterios de aceptación:**
- Cada dispositivo debe tener un certificado único o compartir uno por tipo
- Los certificados deben almacenarse de forma segura (Nota: segura ante que tipo de amenaza? puede guardarlos encriptados en una base de datos, en un servicio como vault, en las github secrets...)
- El sistema debe asociar automáticamente certificados a políticas y things (Esto lo hace terraform, ver ejemplo previo)

#### HU-03: Reglas IoT y CloudWatch
**Como** administrador  
**Quiero** configurar reglas IoT y CloudWatch con retención de 3 días  
**Para** procesar mensajes y mantener logs optimizados
**Nota** esto tambien se puede hacer en terraform, el administrador es alguien tecnico o un operario?

**Criterios de aceptación:**
- Configurar retención de logs de CloudWatch a 3 días exactamente
- Crear reglas IoT que procesen mensajes según patrones específicos
- Implementar manejo de errores en las reglas IoT (Nota: que significa esto?, es algo que funciona en el hardware del dispositivo o en el servicio backend?, es una configuracion?)

### Backend (EP-02)

#### HU-04: Modelo de Datos PostgreSQL
**Como** desarrollador  
**Quiero** implementar un esquema PostgreSQL con Prisma  
**Para** almacenar información de dispositivos y lugares
**Nota** es bien, toma un par de dias hacer esto

**Criterios de aceptación:**
- Implementar el esquema definido con Prisma
- Crear índices para optimizar consultas frecuentes (Nota: realment es necesario?, si las tablas no tienen mas de 10.000 registros y si la concurrencia no es de mas de 100 consultas no creo que valga la pena)
- Incluir relaciones entre lugares, dispositivos y topics (Nota: esto es relacionado con el modelo de bases de datos?)

#### HU-05: API REST de Lugares
**Como** operador  
**Quiero** APIs para gestionar lugares con geocercas  
**Para** definir zonas de monitoreo
**Nota** es medianamente engorroso, sale en un par de dias

**Criterios de aceptación:**
- Implementar CRUD completo para lugares
- Permitir definir geocercas en formato GeoJSON (Nota: no es claro, quiere subir un archivo en ese formato?, va a mandar puntos con ese formato?)
- Optimizar consultas de dispositivos por lugar (Nota: si todas las entidades asociadas a un lugar no pueden estar en otro y no van a ser usadas por otros lugares, mejor usar un campo BSON con esa info en lugar de hacer tablas separadas que solo van a agregar complejidad y desempeño lento)

#### HU-06: API REST de Dispositivos
**Como** operador  
**Quiero** APIs para registrar y administrar dispositivos  
**Para** mantener control del inventario de sensores
**Nota** es bien, toma un par de dias

**Criterios de aceptación:**
- Implementar CRUD completo para dispositivos
- Registrar dispositivos en AWS IoT automáticamente (Nota: es peligroso, muy peligroso esto, yo no lo haria porque estan dando rienda a que se creen recursos sin supervision)
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
**Nota** es bastante engorroso, lleva como una semana solo esto

**Criterios de aceptación:**
- Implementar mapa usando MapLibre GL
- Mostrar marcadores de lugares y dispositivos
- Permitir navegación y zoom en el mapa

#### HU-09: Gestión de Geocercas
**Como** usuario  
**Quiero** visualizar geocercas de los lugares  
**Para** delimitar áreas de monitoreo
**Nota** usa lo anterior pero igual de un par de dias no baja

**Criterios de aceptación:**
- Mostrar geocercas como polígonos en el mapa
- Visualizar dispositivos dentro de sus geocercas correspondientes
- Implementar colores según estado de dispositivos (Nota: cuales colores?, cuales estados?)

#### HU-10: Creación de Dispositivos
**Como** usuario  
**Quiero** un formulario para crear dispositivos asociados a lugares  
**Para** registrar nuevos sensores en el sistema
**Nota** esto toma tiempo, como una semana

**Criterios de aceptación:**
- Implementar modal con formulario de creación
- Permitir seleccionar tipo de dispositivo
- Mostrar información de certificados generados (Nota: esto es peligroso, estas mandando por web algo que no deberia salir de aws y de los dispositivos)

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
- Desarrollar un módulo de simulación de dispositivos (Nota: si no hay documentación esto es super largo, como para una semana)
- Implementar generación de datos según tipo de dispositivo
- Crear función para simular múltiples dispositivos simultáneamente

### TT-04: Optimizaciones
- Implementar consultas SQL optimizadas para Prisma (Nota: si no tienes miles de registros y una concurrencia mayor a 100 simplemente diseña bien la base de datos y usa en aurora un nodo dedicado a consultas y no pierdas tiempo en esto)
- Configurar particionamiento para datos históricos (Nota: guarda en S3 logs)
- Establecer estrategias para reducir costos en AWS (Nota: guarda en S3 logs)

## Requisitos No Funcionales

### RNF-01: Seguridad
- Toda comunicación debe ser encriptada mediante TLS
- Implementar autenticación JWT para APIs REST (Nota: no es suficiente, necesita configurar por cors y poner csp)
- Utilizar políticas específicas por tipo de dispositivo

### RNF-02: Rendimiento
- El sistema debe soportar al menos 1000 dispositivos simultáneos (Nota: eso no me dice nada, son concurrentes?)
- Las respuestas de API deben ser menores a 500ms (Nota: no es algo que usted pueda controlar porque usted no maneja la latencia de la red)
- El mapa debe cargar en menos de 3 segundos (Nota: no es algo que usted pueda controlar porque eso depende de su proveedor de servicio de mapa)

### RNF-03: Disponibilidad
- El sistema debe tener una disponibilidad de 99.9% (no va a tener esa disponibilidad, necesita hacer cosas como despliegues canarios y orquestación de multiple sector de aws)
- Implementar manejo de errores y reintentos en toda comunicación (Nota: no es claro a que se refiere esto)
- Documentar procedimientos de recuperación (Nota: no es claro a que se refiere esto)

### RNF-04: Escalabilidad
- La arquitectura debe permitir escalado horizontal
- Implementar particionamiento para datos históricos (Nota: guarde en s3 y no se complique)
- Optimizar consultas para grandes volúmenes de datos (Nota: sea claro con que son grandes volumenes, para una base de datos como AuroraDB (postgres de aws) si no pasa de un millon de registros usted no esta manejando grandes volumenes)