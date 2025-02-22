# NestJS에서 Kafka와 Avro를 이용한 데이터 스트리밍 구축하기

데이터가 분산되고 실시간으로 처리가 필요한 시스템에서 **Kafka**와 **Avro**를 결합해 메시징 시스템을 구축하는 것은 효율적이고 확장 가능한 선택입니다. 이 글에서는 **NestJS**와 **Kafka**, 그리고 **Confluent Schema Registry**를 사용하여 Avro 직렬화를 구현하고, 이를 통해 다양한 데이터 구조를 안전하고 일관되게 처리하는 방법을 설명합니다.

NestJS는 직관적인 구조와 모듈화를 지원하는 Node.js 프레임워크로, Kafka와 결합하여 데이터 스트리밍 시스템을 쉽게 확장할 수 있습니다. **Schema Registry**를 통해 Avro 스키마를 중앙에서 관리하고, 다양한 데이터 형식을 다루는 사례들을 소개합니다.

## 1. 왜 Avro인가?

Kafka는 JSON, XML 등 다양한 포맷을 지원하지만, **Avro**는 빠르고, 바이너리 포맷으로 크기가 작아 **네트워크 효율성**이 높습니다. 또한, 스키마 기반으로 직렬화되므로 데이터 일관성을 보장합니다. **Confluent Schema Registry**와 함께 Avro를 사용하면 각 메시지가 동일한 스키마를 따르도록 관리하고, 변경이 필요할 때에도 기존 데이터를 안전하게 호환할 수 있습니다.

## 2. 시스템 구성 요소

아래의 세 가지 서비스가 주요 구성 요소입니다.

- **SchemaRegistryService**: Avro 스키마를 관리하고, 데이터 직렬화와 역직렬화를 담당합니다.
- **KafkaProducerService**: Avro로 직렬화된 데이터를 Kafka로 전송하는 역할을 합니다.
- **KafkaConsumerService**: Kafka에서 데이터를 수신하고, 역직렬화하여 데이터를 처리합니다.

## 3. 다양한 Avro 스키마 정의 예시

### 사용자 등록 정보 (User Registration)

사용자 정보에는 필수 데이터(이메일, 비밀번호 등)와 선택적 데이터(프로필)가 포함됩니다. 아래는 해당 데이터를 정의한 Avro 스키마입니다.

```json
{
  "type": "record",
  "name": "UserRegistration",
  "fields": [
    { "name": "userId", "type": "string" },
    { "name": "email", "type": "string" },
    { "name": "password", "type": "string" },
    { "name": "createdAt", "type": "long" },
    {
      "name": "profile",
      "type": [
        "null",
        {
          "type": "record",
          "name": "UserProfile",
          "fields": [
            { "name": "firstName", "type": "string" },
            { "name": "lastName", "type": "string" },
            { "name": "age", "type": ["null", "int"], "default": null },
            { "name": "address", "type": ["null", "string"], "default": null }
          ]
        }
      ],
      "default": null
    }
  ]
}
```

### 주문 데이터 (Order Data)

주문 데이터는 사용자 ID, 주문 날짜, 제품 목록을 포함하며, 각 제품에는 상품 ID, 이름, 수량, 가격이 포함됩니다.

```json
{
  "type": "record",
  "name": "Order",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "userId", "type": "string" },
    { "name": "orderDate", "type": "long" },
    {
      "name": "products",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "Product",
          "fields": [
            { "name": "productId", "type": "string" },
            { "name": "productName", "type": "string" },
            { "name": "quantity", "type": "int" },
            { "name": "price", "type": "double" }
          ]
        }
      }
    },
    { "name": "totalAmount", "type": "double" }
  ]
}
```

### 센서 데이터 (Sensor Data)

실시간으로 전송되는 센서 데이터는 센서 ID, 측정 시간, 온도와 습도를 포함하며, 센서 상태는 `NORMAL`, `WARNING`, `CRITICAL`로 구분됩니다.

```json
{
  "type": "record",
  "name": "SensorData",
  "fields": [
    { "name": "sensorId", "type": "string" },
    { "name": "timestamp", "type": "long" },
    { "name": "temperature", "type": ["null", "float"], "default": null },
    { "name": "humidity", "type": ["null", "float"], "default": null },
    {
      "name": "status",
      "type": {
        "type": "enum",
        "name": "SensorStatus",
        "symbols": ["NORMAL", "WARNING", "CRITICAL"]
      }
    }
  ]
}
```

## 4. SchemaRegistryService 구현

Schema Registry를 통해 Avro 스키마를 관리하는 서비스를 생성합니다.

```typescript
// schema-registry.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { SchemaRegistry } from "@kafkajs/confluent-schema-registry";

@Injectable()
export class SchemaRegistryService {
  private registry: SchemaRegistry;
  private readonly logger = new Logger(SchemaRegistryService.name);

  constructor() {
    this.registry = new SchemaRegistry({ host: "http://localhost:8081" });
  }

  async encode(data: any, subject: string) {
    try {
      const { id } = await this.registry.getLatestSchemaId(subject);
      return await this.registry.encode(id, data);
    } catch (error) {
      this.logger.error(`Encoding failed: ${error.message}`);
      throw error;
    }
  }

  async decode(encodedData: Buffer) {
    try {
      return await this.registry.decode(encodedData);
    } catch (error) {
      this.logger.error(`Decoding failed: ${error.message}`);
      throw error;
    }
  }
}
```

## 5. Kafka Producer 구현

아래는 사용자 등록 정보를 Kafka 주제(`user-registration`)로 전송하는 예제입니다. `KafkaProducerService`는 데이터를 직렬화한 후 전송합니다.

```typescript
// kafka-producer.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { ClientKafka } from "@nestjs/microservices";
import { SchemaRegistryService } from "./schema-registry.service";

@Injectable()
export class KafkaProducerService {
  private readonly logger = new Logger(KafkaProducerService.name);

  constructor(
    private readonly kafkaClient: ClientKafka,
    private readonly schemaRegistryService: SchemaRegistryService
  ) {}

  async sendUserRegistration(topic: string, data: any) {
    try {
      const serializedData = await this.schemaRegistryService.encode(
        data,
        "UserRegistration-value"
      );
      await this.kafkaClient.emit(topic, { value: serializedData });
      this.logger.log(`User registration data sent to topic ${topic}`);
    } catch (error) {
      this.logger.error(`Failed to send data: ${error.message}`);
      throw error;
    }
  }
}
```

## 6. Kafka Consumer 구현

Kafka로부터 `user-registration` 주제를 구독하여 데이터를 수신하고, Avro를 통해 역직렬화하여 처리합니다.

```typescript
// kafka-consumer.service.ts
import { Injectable, OnModuleInit, Logger } from "@nestjs/common";
import { Consumer, Kafka } from "kafkajs";
import { SchemaRegistryService } from "./schema-registry.service";

@Injectable()
export class KafkaConsumerService implements OnModuleInit {
  private readonly logger = new Logger(KafkaConsumerService.name);
  private consumer: Consumer;

  constructor(private readonly schemaRegistryService: SchemaRegistryService) {
    const kafka = new Kafka({ brokers: ["localhost:9092"] });
    this.consumer = kafka.consumer({ groupId: "user-group" });
  }

  async onModuleInit() {
    await this.consumer.connect();
    await this.consumer.subscribe({
      topic: "user-registration",
      fromBeginning: true
    });

    await this.consumer.run({
      eachMessage: async ({ message }) => {
        try {
          const decodedData = await this.schemaRegistryService.decode(
            message.value
          );
          this.logger.log(
            `Received user registration data: ${JSON.stringify(decodedData)}`
          );
          this.processUserRegistration(decodedData);
        } catch (error) {
          this.logger.error(`Failed to process message: ${error.message}`);
        }
      }
    });
  }

  processUserRegistration(userData: any) {
    this.logger.log(
      `Processing user registration: ${JSON.stringify(userData)}`
    );
  }
}
```

## 7. 모듈 구성

마지막으로 Kafka 클라이언트를 NestJS 모듈에 설정하고, 프로듀서와 컨슈머를 모듈로 구성합니다.

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { KafkaProducerService } from './kafka-producer.service';
import { KafkaConsumerService } from './kafka-cons

umer.service';
import { SchemaRegistryService } from './schema-registry.service';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'KAFKA_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            brokers: ['localhost:9092'],
          },
          consumer: {
            groupId: 'user-group',
          },
        },
      },
    ]),
  ],
  providers: [KafkaProducerService, KafkaConsumerService, SchemaRegistryService],
})
export class AppModule {}
```

---

NestJS에서 Kafka와 Avro를 사용해 데이터 스트리밍 시스템을 구축하는 방법을 살펴보았습니다. **Schema Registry**를 이용해 다양한 데이터를 일관된 스키마로 관리하고, Kafka를 통해 실시간으로 데이터를 전송, 수신하는 시스템을 구현할 수 있습니다.

각 `acks` 옵션에 따른 플로우차트를 Markdown의 Mermaid 형식으로 도식화해 보겠습니다.
