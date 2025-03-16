# Sistema IoT para Seguridad con AWS IoT Core

Sistema de monitoreo de dispositivos IoT para seguridad que integra AWS IoT Core, NestJS, PostgreSQL y MapLibre GL.

## Descripción del Proyecto
Sistema integrado de dispositivos IoT para seguridad conectados mediante MQTT, con una plataforma web en Preact que permite la gestión de dispositivos, visualización en tiempo real y almacenamiento de históricos.

## Arquitectura del Sistema

```mermaid
flowchart TD
    subgraph "Dispositivos IoT"
        D1[Sensor Movimiento]
        D2[Sensor Apertura]
        D3[Cámara IP]
        D4[Alarma]
    end

    subgraph "AWS Cloud"
        AWS_IOT[AWS IoT Core]
        AWS_RULE[AWS IoT Rules Engine]
        AWS_CW[CloudWatch\n3 días retención]
    end

    subgraph "Backend"
        NEST[NestJS API]
        PRISMA[Prisma ORM]
        PG[(PostgreSQL)]
        WS_CLI[WebSocket Client]
        AWS_SDK[AWS SDK]
    end

    subgraph "Comunicación"
        ERLANG[Servidor Erlang\nWebSocket]
    end

    subgraph "Frontend"
        PREACT[Preact App]
        MAPLIBRE[MapLibre GL]
    end

    D1 -->|MQTT/TLS| AWS_IOT
    D2 -->|MQTT/TLS| AWS_IOT
    D3 -->|MQTT/TLS| AWS_IOT
    D4 -->|MQTT/TLS| AWS_IOT

    AWS_IOT -->|Logs| AWS_CW
    AWS_IOT --> AWS_RULE
    AWS_RULE -->|Reenvía eventos| NEST

    NEST <-->|CRUD| PRISMA
    PRISMA <-->|SQL| PG
    NEST -->|Crea reglas\ny topics| AWS_SDK
    AWS_SDK -->|Administra| AWS_IOT
    NEST -->|Notifica eventos| WS_CLI
    WS_CLI -->|WebSocket| ERLANG

    ERLANG -.->|No hay conexión\ndirecta| PREACT
    PREACT -->|Renderiza| MAPLIBRE

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E,stroke-width:2px
    classDef backend fill:#69b3a2,stroke:#333,stroke-width:2px
    classDef frontend fill:#5d87fe,stroke:#333,stroke-width:2px
    classDef comm fill:#9d65ff,stroke:#333,stroke-width:2px
    
    class AWS_IOT,AWS_RULE,AWS_CW aws
    class NEST,PRISMA,PG,WS_CLI,AWS_SDK backend
    class PREACT,MAPLIBRE frontend
    class ERLANG comm
```

## Planificación del proyecto (4 semanas)

### Semana 1: Infraestructura y Base
- **Días 1-2:** Configuración de AWS y esquema de base de datos
  - Configurar AWS IoT Core, IAM, CloudWatch
  - Implementar esquema PostgreSQL con Prisma
  - Configurar NestJS con módulos base
- **Días 3-5:** Implementación de servicios core
  - Servicio AWS IoT para gestión de dispositivos y topics
  - Servicio Gateway para comunicación con Erlang
  - Controladores REST para lugares y dispositivos

### Semana 2: Servicios y lógica de negocio
- **Días 1-2:** Implementación de topics y reglas AWS IoT
  - Creación de topics a través de AWS SDK
  - Configuración de reglas para reenvío de mensajes
  - Integración con CloudWatch (retención 3 días)
- **Días 3-5:** Sistema de dispositivos y certificados
  - Flujo de registro de dispositivos
  - Generación y gestión de certificados
  - Pruebas con dispositivos simulados

### Semana 3: Frontend y visualización
- **Días 1-3:** Implementación del mapa con MapLibre GL
  - Configuración de mapa base
  - Visualización de geocercas
  - Marcadores para dispositivos
- **Días 4-5:** Interface de usuario
  - Modal para crear dispositivos
  - Modal para crear topics
  - Visualización de datos de dispositivos

### Semana 4: Integración y pruebas
- **Días 1-2:** Integración Backend-Frontend
  - Conexión de APIs
  - Manejo de eventos y alertas
- **Días 3-5:** Pruebas y optimización
  - Simulación de dispositivos
  - Optimización de consultas
  - Finalización de documentación

## Configuración de la Base de Datos

### Esquema Prisma

```prisma
// schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

// Modelo para lugares (edificios, casas, conjuntos, empresas)
model Place {
  id          Int      @id @default(autoincrement())
  name        String   @db.VarChar(100)
  type        String   @db.VarChar(50) // 'home', 'business', 'complex', 'building'
  address     String?  @db.VarChar(200)
  // Coordenadas del centro del lugar
  latitude    Decimal  @db.Decimal(10, 7)
  longitude   Decimal  @db.Decimal(10, 7)
  // Definición de la geocerca (polígono en formato GeoJSON)
  geofence    Json?    
  // Relaciones
  devices     Device[]
  // Timestamps
  createdAt   DateTime @default(now()) @db.Timestamptz(6)
  updatedAt   DateTime @updatedAt @db.Timestamptz(6)

  // Índices para búsqueda geoespacial
  @@index([latitude, longitude])
}

// Modelo para dispositivos IoT
model Device {
  id              Int       @id @default(autoincrement())
  uuid            String    @unique @db.VarChar(50)
  name            String    @db.VarChar(100)
  description     String?   @db.Text
  type            String    @db.VarChar(50) // 'motion-sensor', 'door-sensor', 'camera', etc.
  // Ubicación del dispositivo
  latitude        Decimal?  @db.Decimal(10, 7)
  longitude       Decimal?  @db.Decimal(10, 7)
  // Estado actual
  status          String    @default("disconnected") @db.VarChar(50)
  lastReport      DateTime? @db.Timestamptz(6)
  // AWS IoT
  awsThingName    String?   @db.VarChar(128)
  certificateArn  String?   @db.VarChar(255)
  // Relaciones
  placeId         Int
  place           Place     @relation(fields: [placeId], references: [id], onDelete: Cascade)
  topic           Topic?    @relation(fields: [topicId], references: [id])
  topicId         Int?
  data            DeviceData[]
  alerts          Alert[]
  // Timestamps
  createdAt       DateTime  @default(now()) @db.Timestamptz(6)
  updatedAt       DateTime  @updatedAt @db.Timestamptz(6)

  // Índices
  @@index([placeId])
  @@index([topicId])
  @@index([status])
  @@index([lastReport])
}

// Modelo para topics MQTT
model Topic {
  id          Int       @id @default(autoincrement())
  pattern     String    @unique @db.VarChar(200)
  description String?   @db.Text
  qos         Int       @default(0)
  retain      Boolean   @default(false)
  // AWS IoT Specific
  ruleArn     String?   @db.VarChar(255)
  // Relaciones
  devices     Device[]
  // Timestamps
  createdAt   DateTime  @default(now()) @db.Timestamptz(6)
  updatedAt   DateTime  @updatedAt @db.Timestamptz(6)
}

// Modelo para datos históricos de dispositivos
model DeviceData {
  id          Int       @id @default(autoincrement())
  deviceId    Int
  device      Device    @relation(fields: [deviceId], references: [id], onDelete: Cascade)
  data        Json
  timestamp   DateTime  @default(now()) @db.Timestamptz(6)

  // Índices
  @@index([deviceId])
  @@index([timestamp])
}

// Modelo para alertas
model Alert {
  id          Int       @id @default(autoincrement())
  deviceId    Int
  device      Device    @relation(fields: [deviceId], references: [id], onDelete: Cascade)
  type        String    @db.VarChar(50)
  message     String    @db.Text
  read        Boolean   @default(false)
  readAt      DateTime? @db.Timestamptz(6)
  timestamp   DateTime  @default(now()) @db.Timestamptz(6)

  // Índices
  @@index([deviceId])
  @@index([read])
  @@index([timestamp])
}
```

## Implementación Backend (NestJS)

### Servicio AWS IoT

```typescript
// aws-iot.service.ts - Implementación principal
import { Injectable, OnModuleInit, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PrismaService } from '../prisma/prisma.service';
import { ErlangGatewayService } from '../erlang/erlang-gateway.service';
import * as AWS from 'aws-sdk';

@Injectable()
export class AwsIotService implements OnModuleInit {
  private iot: AWS.Iot;
  private iotData: AWS.IotData;
  private logger = new Logger('AwsIotService');

  constructor(
    private configService: ConfigService,
    private prisma: PrismaService,
    private erlangGateway: ErlangGatewayService,
  ) {
    // Configurar AWS SDK
    AWS.config.update({
      region: this.configService.get<string>('AWS_REGION'),
      accessKeyId: this.configService.get<string>('AWS_ACCESS_KEY_ID'),
      secretAccessKey: this.configService.get<string>('AWS_SECRET_ACCESS_KEY'),
    });

    this.iot = new AWS.Iot();
    this.iotData = new AWS.IotData({
      endpoint: this.configService.get<string>('AWS_IOT_ENDPOINT'),
    });
  }

  async onModuleInit() {
    this.logger.log('Inicializando servicio AWS IoT...');
  }

  /**
   * Registra un nuevo dispositivo en AWS IoT y lo asocia a un topic
   */
  async registerDevice(deviceData: {
    uuid: string;
    name: string;
    type: string;
    placeId: number;
  }) {
    try {
      // 1. Crear una cosa (thing) en AWS IoT
      const thingName = `device-${deviceData.uuid}`;
      
      // TODO: Implementar createThing()
      
      // 2. Crear certificado
      // TODO: Implementar createCertificate()
      
      // 3. Asociar certificado a la cosa
      // TODO: Implementar attachCertificateToThing()
      
      // 4. Crear y asociar política
      // TODO: Implementar createAndAttachPolicy()
      
      // 5. Generar un topic para el dispositivo
      const topicPattern = `seguridad/${deviceData.type}/${deviceData.placeId}/${deviceData.uuid}`;
      
      // 6. Crear topic en la base de datos
      const topic = await this.prisma.topic.create({
        data: {
          pattern: topicPattern,
          description: `Topic para ${deviceData.name}`,
          qos: 1,
          retain: true,
        },
      });

      // 7. Crear regla para el topic
      // TODO: Implementar createTopicRule()
      
      // 8. Registrar el dispositivo en la base de datos
      const device = await this.prisma.device.create({
        data: {
          uuid: deviceData.uuid,
          name: deviceData.name,
          type: deviceData.type,
          placeId: deviceData.placeId,
          awsThingName: thingName,
          topicId: topic.id,
        },
        include: {
          place: true,
          topic: true,
        },
      });

      return {
        device,
        // Devolver información de certificados
      };
    } catch (error) {
      this.logger.error(`Error al registrar dispositivo: ${error.message}`);
      throw error;
    }
  }

  /**
   * Simula envío de datos desde un dispositivo para pruebas
   */
  async simulateDeviceData(deviceId: number, data: any) {
    try {
      const device = await this.prisma.device.findUnique({
        where: { id: deviceId },
        include: { topic: true },
      });

      if (!device || !device.topic) {
        throw new Error('Dispositivo no encontrado o sin topic asociado');
      }

      // Simular publicación MQTT
      await this.publishMessage(device.topic.pattern, {
        timestamp: new Date().toISOString(),
        deviceId: device.uuid,
        ...data,
      });

      return true;
    } catch (error) {
      this.logger.error(`Error al simular datos: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Publica un mensaje MQTT en AWS IoT Core
   */
  async publishMessage(topic: string, payload: any, options: { qos?: number; retain?: boolean } = {}) {
    try {
      const params = {
        topic,
        payload: JSON.stringify(payload),
        qos: options.qos || 0,
        retain: options.retain || false,
      };

      await this.iotData.publish(params).promise();
      this.logger.log(`Mensaje publicado en topic ${topic}`);
      return true;
    } catch (error) {
      this.logger.error(`Error al publicar mensaje: ${error.message}`);
      throw error;
    }
  }
  
  // Implementar métodos:
  // - createThing
  // - createCertificate
  // - attachCertificateToThing
  // - createAndAttachPolicy
  // - createTopicRule
  // - processIotMessage (para procesamiento de mensajes recibidos)
}
```

### Servicio Gateway Erlang

```typescript
// erlang-gateway.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as WebSocket from 'ws';

@Injectable()
export class ErlangGatewayService implements OnModuleInit, OnModuleDestroy {
  private ws: WebSocket;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private reconnectInterval = 3000; // 3 segundos
  private reconnectTimeoutId: NodeJS.Timeout;
  private messageQueue: any[] = [];
  private isConnected = false;
  private logger = new Logger('ErlangGatewayService');

  constructor(private configService: ConfigService) {}

  async onModuleInit() {
    this.connect();
  }

  onModuleDestroy() {
    this.cleanUp();
  }

  private cleanUp() {
    if (this.reconnectTimeoutId) {
      clearTimeout(this.reconnectTimeoutId);
    }
    
    if (this.ws) {
      this.ws.removeAllListeners();
      this.ws.close();
    }
  }

  private connect() {
    // Implementar conexión WebSocket al servidor Erlang
    // con manejo de reconexión y cola de mensajes
  }
  
  /**
   * Envía un mensaje al servidor Erlang
   * Si no hay conexión, encola el mensaje para envío posterior
   */
  async sendMessage(message: any): Promise<boolean> {
    // Implementar lógica de envío/encolamiento de mensajes
    return false;
  }
}
```

### Configuración CloudWatch

```typescript
// cloudwatch.config.ts
import { AWS } from 'aws-sdk';

export const setupCloudWatchRetention = async (logGroupName: string, retentionInDays: number = 3) => {
  const cloudwatchLogs = new AWS.CloudWatchLogs({
    region: process.env.AWS_REGION,
  });
  
  try {
    // Buscar si el grupo de logs existe
    try {
      await cloudwatchLogs.describeLogGroups({
        logGroupNamePrefix: logGroupName,
      }).promise();
    } catch (error) {
      // Si no existe, crearlo
      await cloudwatchLogs.createLogGroup({
        logGroupName,
      }).promise();
    }
    
    // Establecer la retención
    await cloudwatchLogs.putRetentionPolicy({
      logGroupName,
      retentionInDays,
    }).promise();
    
    console.log(`Política de retención de ${retentionInDays} días establecida para ${logGroupName}`);
    return true;
  } catch (error) {
    console.error('Error configurando retención de CloudWatch:', error);
    throw error;
  }
};
```

## Implementación Frontend (Solo representación visual)

### Componente Mapa con MapLibre GL

```jsx
// Map.jsx - Estructura básica del componente de mapa
import { h } from 'preact';
import { useEffect, useRef, useState } from 'preact/hooks';
import maplibregl from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css';

const Map = () => {
  const mapContainer = useRef(null);
  const map = useRef(null);
  const [places, setPlaces] = useState([]);
  const [selectedPlace, setSelectedPlace] = useState(null);
  const [devices, setDevices] = useState([]);
  const [modalOpen, setModalOpen] = useState(false);
  const [modalType, setModalType] = useState(null);

  // Inicializar mapa
  useEffect(() => {
    if (map.current) return;

    map.current = new maplibregl.Map({
      container: mapContainer.current,
      style: 'https://api.maptiler.com/maps/streets/style.json?key=YOUR_MAPTILER_KEY',
      center: [-74.0817, 4.6097], // Bogotá como ejemplo
      zoom: 11,
    });

    // Controles de navegación
    map.current.addControl(new maplibregl.NavigationControl(), 'top-right');

    // Cargar lugares cuando se inicia el mapa
    map.current.on('load', () => {
      fetchPlaces();
    });

    return () => {
      if (map.current) {
        map.current.remove();
      }
    };
  }, []);

  // Cargar lugares desde la API
  const fetchPlaces = async () => {
    try {
      const response = await fetch('http://localhost:3000/api/places');
      const data = await response.json();
      setPlaces(data);
      
      // Añadir marcadores de lugares al mapa
      addPlacesToMap(data);
    } catch (error) {
      console.error('Error fetching places:', error);
    }
  };

  // Añadir lugares al mapa
  const addPlacesToMap = (placesData) => {
    // Implementar código para añadir marcadores de lugares al mapa
  };

  // Seleccionar un lugar
  const selectPlace = async (place) => {
    setSelectedPlace(place);
    
    // Centrar mapa en el lugar
    map.current.flyTo({
      center: [parseFloat(place.longitude), parseFloat(place.latitude)],
      zoom: 15,
    });

    // Dibujar geocerca si existe
    drawGeofence(place);
    
    // Cargar dispositivos del lugar
    fetchDevicesByPlace(place.id);
  };

  // Dibujar geocerca
  const drawGeofence = (place) => {
    // Implementar código para dibujar geocerca en el mapa
  };

  // Cargar dispositivos de un lugar
  const fetchDevicesByPlace = async (placeId) => {
    try {
      const response = await fetch(`http://localhost:3000/api/devices?placeId=${placeId}`);
      const data = await response.json();
      setDevices(data);
      
      // Añadir dispositivos al mapa
      addDevicesToMap(data);
    } catch (error) {
      console.error('Error fetching devices:', error);
    }
  };

  // Añadir dispositivos al mapa
  const addDevicesToMap = (devicesData) => {
    // Implementar código para añadir marcadores de dispositivos al mapa
  };

  // Abrir modal para crear dispositivo
  const openCreateDeviceModal = () => {
    setModalType('createDevice');
    setModalOpen(true);
  };

  return (
    <div className="map-container">
      <div className="map-sidebar">
        <h2>Lugares</h2>
        <ul className="places-list">
          {places.map((place) => (
            <li
              key={place.id}
              className={selectedPlace?.id === place.id ? 'selected' : ''}
              onClick={() => selectPlace(place)}
            >
              {place.name}
            </li>
          ))}
        </ul>
        
        {selectedPlace && (
          <div className="place-actions">
            <h3>Dispositivos en {selectedPlace.name}</h3>
            <ul className="devices-list">
              {devices.map((device) => (
                <li key={device.id} className={`status-${device.status}`}>
                  {device.name}
                  <span className="device-type">{device.type}</span>
                </li>
              ))}
            </ul>
            <button onClick={openCreateDeviceModal}>Añadir Dispositivo</button>
          </div>
        )}
      </div>
      
      <div ref={mapContainer} className="map-content" />
      
      {modalOpen && (
        <Modal
          type={modalType}
          place={selectedPlace}
          onClose={() => setModalOpen(false)}
          onDeviceCreated={() => {
            setModalOpen(false);
            if (selectedPlace) {
              fetchDevicesByPlace(selectedPlace.id);
            }
          }}
        />
      )}
    </div>
  );
};

export default Map;
```

### Componente Modal para Crear Dispositivos

```jsx
// Modal.jsx - Estructura básica 
import { h } from 'preact';
import { useState, useEffect } from 'preact/hooks';

const Modal = ({ type, place, onClose, onDeviceCreated }) => {
  const [formData, setFormData] = useState({});
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [success, setSuccess] = useState(false);

  // Inicializar datos del formulario según tipo de modal
  useEffect(() => {
    if (type === 'createDevice' && place) {
      setFormData({
        name: '',
        type: 'motion-sensor',
        description: '',
        placeId: place.id,
        latitude: '',
        longitude: '',
      });
    }
  }, [type, place]);

  // Manejar cambios en el formulario
  const handleChange = (e) => {
    const { name, value, type, checked } = e.target;
    setFormData({
      ...formData,
      [name]: type === 'checkbox' ? checked : value,
    });
  };

  // Enviar formulario
  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      if (type === 'createDevice') {
        const response = await fetch('http://localhost:3000/api/devices', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(formData),
        });

        if (!response.ok) {
          throw new Error('Error al crear el dispositivo');
        }

        setSuccess(true);
        setTimeout(() => {
          onDeviceCreated();
        }, 2000);
      }
    } catch (error) {
      setError(error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="modal-overlay">
      <div className="modal">
        <div className="modal-header">
          <h2>{type === 'createDevice' ? 'Crear Nuevo Dispositivo' : 'Modal'}</h2>
          <button className="close-button" onClick={onClose}>×</button>
        </div>
        
        <div className="modal-content">
          {success ? (
            <div className="success-message">¡Operación exitosa!</div>
          ) : (
            <form onSubmit={handleSubmit}>
              {type === 'createDevice' && (
                <>
                  <div className="form-group">
                    <label htmlFor="name">Nombre</label>
                    <input
                      type="text"
                      id="name"
                      name="name"
                      value={formData.name}
                      onChange={handleChange}
                      required
                    />
                  </div>
                  
                  <div className="form-group">
                    <label htmlFor="type">Tipo</label>
                    <select
                      id="type"
                      name="type"
                      value={formData.type}
                      onChange={handleChange}
                      required
                    >
                      <option value="motion-sensor">Sensor de Movimiento</option>
                      <option value="door-sensor">Sensor de Puerta</option>
                      <option value="camera">Cámara</option>
                      <option value="alarm">Alarma</option>
                    </select>
                  </div>
                  
                  <div className="form-group">
                    <label htmlFor="description">Descripción</label>
                    <textarea
                      id="description"
                      name="description"
                      value={formData.description}
                      onChange={handleChange}
                    />
                  </div>
                  
                  <div className="form-group">
                    <label htmlFor="latitude">Latitud</label>
                    <input
                      type="number"
                      id="latitude"
                      name="latitude"
                      step="0.0000001"
                      value={formData.latitude}
                      onChange={handleChange}
                    />
                  </div>
                  
                  <div className="form-group">
                    <label htmlFor="longitude">Longitud</label>
                    <input
                      type="number"
                      id="longitude"
                      name="longitude"
                      step="0.0000001"
                      value={formData.longitude}
                      onChange={handleChange}
                    />
                  </div>
                </>
              )}
              
              {error && <div className="error-message">{error}</div>}
              
              <div className="form-actions">
                <button type="button" onClick={onClose}>Cancelar</button>
                <button type="submit" disabled={loading}>
                  {loading ? 'Procesando...' : 'Guardar'}
                </button>
              </div>
            </form>
          )}
        </div>
      </div>
    </div>
  );
};

export default Modal;
```

## Optimizaciones y Recomendaciones

### 1. Optimización de Consultas SQL (Prisma)

```typescript
// Ejemplo de consulta optimizada para dispositivos por lugar
// Incluye solo los datos necesarios y limita resultados
const getDevicesByPlace = async (placeId: number) => {
  return prisma.device.findMany({
    where: { placeId },
    select: {
      id: true,
      name: true,
      type: true,
      status: true,
      latitude: true,
      longitude: true,
      // No incluir datos innecesarios como certificados
    },
    orderBy: { 
      name: 'asc' 
    },
    // Usar paginación para grandes conjuntos de datos
    take: 100,
  });
};

// Uso de raw queries para casos de rendimiento crítico
const getDeviceStats = async () => {
  return prisma.$queryRaw`
    SELECT 
      p.id, 
      p.name, 
      COUNT(d.id) as device_count,
      SUM(CASE WHEN d.status = 'active' THEN 1 ELSE 0 END) as active_devices
    FROM "Place" p
    LEFT JOIN "Device" d ON d."placeId" = p.id
    GROUP BY p.id, p.name
    ORDER BY device_count DESC
  `;
};
```

### 2. Estrategias para reducir costos en AWS

1. **CloudWatch Logs:**
   - Configurar retención de 3 días como se especificó
   - Usar filtros métricos específicos en lugar de registrar todo

2. **AWS IoT Core:**
   - Usar Basic Ingest para mensajes de solo entrada (más económico)
   - Implementar reglas eficientes que filtren mensajes innecesarios
   - Agregar mensajes cuando sea posible para reducir el número total

3. **Certificados:**
   - Crear certificados que sirvan para múltiples dispositivos del mismo tipo
   - Usar rotación de certificados programada para seguridad

### 3. Estrategia de almacenamiento de datos históricos

```typescript
// Implementar particionamiento por tiempo
// Crear función para migrar datos antiguos a tablas particionadas
const migrateHistoricalData = async () => {
  // Mover datos de más de 30 días a tabla histórica
  await prisma.$executeRaw`
    INSERT INTO "DeviceDataHistorical" (id, "deviceId", data, timestamp)
    SELECT id, "deviceId", data, timestamp 
    FROM "DeviceData"
    WHERE timestamp < NOW() - INTERVAL '30 days'
  `;
  
  // Eliminar datos migrados
  await prisma.$executeRaw`
    DELETE FROM "DeviceData"
    WHERE timestamp < NOW() - INTERVAL '30 days'
  `;
};

// Programar esta función para ejecutarse periódicamente
// Implementar como tarea cron en NestJS
```

### 4. Seguridad

1. **Certificados TLS:** Utilizar autenticación mutua TLS para todos los dispositivos.

2. **Políticas de IoT:** Crear políticas específicas por tipo de dispositivo para limitar las acciones que pueden realizar.

3. **Validación de datos:** Implementar esquemas de validación para los mensajes MQTT recibidos.

4. **API Gateway:** Proteger las APIs REST con autenticación JWT.

### 5. Simulación de dispositivos para pruebas

```typescript
// Función para simular múltiples dispositivos
const simulateMultipleDevices = async (
  placeId: number, 
  count: number, 
  interval: number = 5000
) => {
  // Obtener dispositivos del lugar
  const devices = await prisma.device.findMany({
    where: { placeId },
    include: { topic: true },
  });
  
  if (devices.length === 0) {
    console.log('No hay dispositivos para simular');
    return;
  }
  
  // Crear simuladores para cada dispositivo
  const simulators = devices.map(device => {
    // Generar datos según tipo de dispositivo
    const generateData = () => {
      switch(device.type) {
        case 'motion-sensor':
          return {
            status: Math.random() > 0.7 ? 'active' : 'inactive',
            motion: Math.random() > 0.7,
            battery: Math.floor(70 + Math.random() * 30),
          };
        case 'door-sensor':
          return {
            status: 'active',
            open: Math.random() > 0.8,
            battery: Math.floor(70 + Math.random() * 30),
          };
        case 'camera':
          return {
            status: 'active',
            recording: Math.random() > 0.5,
            battery: Math.floor(80 + Math.random() * 20),
          };
        case 'alarm':
          const isAlarm = Math.random() > 0.95;
          return {
            status: isAlarm ? 'alarm' : 'active',
            alarm: isAlarm,
            battery: Math.floor(60 + Math.random() * 40),
          };
        default:
          return {
            status: 'active',
            value: Math.floor(Math.random() * 100),
          };
      }
    };
    
    return {
      device,
      start: () => {
        const intervalId = setInterval(async () => {
          try {
            const data = generateData();
            await awsIotService.publishMessage(device.topic.pattern, {
              timestamp: new Date().toISOString(),
              deviceId: device.uuid,
              ...data,
            });
            console.log(`Datos simulados enviados para ${device.name}`);
          } catch (error) {
            console.error(`Error simulando ${device.name}:`, error);
          }
        }, interval);
        
        return () => clearInterval(intervalId);
      }
    };
  });
  
  // Iniciar simulación para la cantidad especificada
  const activeSimulators = simulators.slice(0, count).map(sim => ({
    device: sim.device,
    stop: sim.start(),
  }));
  
  console.log(`Simulando ${activeSimulators.length} dispositivos cada ${interval/1000} segundos`);
  
  // Devolver función para detener la simulación
  return () => {
    activeSimulators.forEach(sim => sim.stop());
    console.log('Simulación detenida');
  };
};
```

## Comandos de Inicio Rápido

Configuración del entorno:

```bash
# Instalar dependencias
npm install

# Generar cliente Prisma
npx prisma generate

# Aplicar migraciones a la base de datos
npx prisma migrate dev

# Iniciar la aplicación en modo desarrollo
npm run start:dev
```

Configuración de AWS:

```bash
# Configurar credenciales AWS
aws configure

# Crear grupo de CloudWatch con retención de 3 días
aws logs create-log-group --log-group-name "/aws/iot/security-devices"
aws logs put-retention-policy --log-group-name "/aws/iot/security-devices" --retention-in-days 3
```

## Variables de Entorno (.env)

```
# Base de datos
DATABASE_URL="postgresql://username:password@localhost:5432/iot_security?schema=public"

# AWS
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_access_key_id
AWS_SECRET_ACCESS_KEY=your_secret_access_key
AWS_ACCOUNT_ID=your_account_id
AWS_IOT_ENDPOINT=your_iot_endpoint.iot.us-east-1.amazonaws.com
AWS_IOT_ROLE_ARN=arn:aws:iam::your_account_id:role/iot_role
AWS_LAMBDA_ARN=arn:aws:lambda:us-east-1:your_account_id:function:iot-processor
AWS_CLOUDWATCH_LOG_GROUP=/aws/iot/security-devices

# Erlang WebSocket
ERLANG_WS_URL=ws://localhost:8080/socket

# JWT
JWT_SECRET=your_jwt_secret
JWT_EXPIRATION=86400

# Puerto del servidor
PORT=3000
```

## Recomendaciones para producción

1. **Alta disponibilidad**:
   - Utilizar AWS IoT Core en lugar de un broker MQTT autogestionado
   - Configurar múltiples instancias del backend NestJS
   - Implementar una base de datos PostgreSQL con replicación

2. **Monitoreo**:
   - Configurar alarmas en CloudWatch para métricas clave
   - Implementar health checks para el backend
   - Utilizar AWS X-Ray para rastreo distribuido

3. **Seguridad**:
   - Rotar certificados y credenciales periódicamente
   - Implementar cifrado de datos sensibles en reposo
   - Revisar y ajustar políticas IAM regularmente

4. **Escalabilidad**:
   - Utilizar particionamiento de tablas para datos históricos
   - Implementar caching para consultas frecuentes
   - Considerar soluciones de time-series database para métricas

5. **Recuperación ante desastres**:
   - Configurar backups automáticos de la base de datos
   - Documentar procedimientos de recuperación
   - Realizar pruebas periódicas de recuperación