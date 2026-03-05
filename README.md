1️⃣ Sequelize Models
models/Flight.ts
Ts
Copy code
import { Model, DataTypes } from 'sequelize';
import { sequelize } from '../db';

export class Flight extends Model {
  declare id: string;
  declare flight_number: string;
  declare status: string;
}

Flight.init({
  id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
  flight_number: { type: DataTypes.STRING, allowNull: false },
  status: { type: DataTypes.STRING, allowNull: false, defaultValue: 'scheduled' },
}, {
  sequelize,
  modelName: 'Flight',
  tableName: 'flights',
});
models/Notification.ts
Ts
Copy code
import { Model, DataTypes } from 'sequelize';
import { sequelize } from '../db';
import { Flight } from './Flight';

export class Notification extends Model {
  declare id: string;
  declare flight_id: string;
  declare message: string;
  declare read: boolean;
}

Notification.init({
  id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
  flight_id: { type: DataTypes.UUID, allowNull: false },
  message: { type: DataTypes.TEXT, allowNull: false },
  read: { type: DataTypes.BOOLEAN, defaultValue: false },
}, {
  sequelize,
  modelName: 'Notification',
  tableName: 'notifications',
});

// Association
Flight.hasMany(Notification, { foreignKey: 'flight_id' });
Notification.belongsTo(Flight, { foreignKey: 'flight_id' });
✅ This links notifications to a flight.
2️⃣ RabbitMQ Setup
worker/rabbitmq/connection.ts
Ts
Copy code
import amqp, { Channel } from 'amqplib';

let channel: Channel;

export async function connectRabbitMQ(): Promise<Channel> {
  if (channel) return channel;
  const connection = await amqp.connect('amqp://guest:guest@rabbitmq:5672');
  channel = await connection.createChannel();
  return channel;
}

export function getChannel(): Channel {
  if (!channel) throw new Error('RabbitMQ not connected');
  return channel;
}
worker/rabbitmq/publisher.ts
Ts
Copy code
import { getChannel } from './connection';

export async function publishNotification(flightId: string, message: string) {
  const channel = getChannel();
  const queue = 'notifications';
  await channel.assertQueue(queue, { durable: true });
  channel.sendToQueue(queue, Buffer.from(JSON.stringify({ flightId, message })), { persistent: true });
}
worker/rabbitmq/consumer.ts
Ts
Copy code
import { getChannel } from './connection';
import { Notification } from '../../models/Notification';

export async function consumeNotifications() {
  const channel = getChannel();
  const queue = 'notifications';
  await channel.assertQueue(queue, { durable: true });

  channel.consume(queue, async (msg) => {
    if (!msg) return;
    const data = JSON.parse(msg.content.toString());
    console.log('Received notification:', data);

    await Notification.create({
      flight_id: data.flightId,
      message: data.message,
    });

    channel.ack(msg);
  });
}
3️⃣ Worker Service Entry
worker/index.ts
Ts
Copy code
import { connectRabbitMQ } from './rabbitmq/connection';
import { consumeNotifications } from './rabbitmq/consumer';
import { sequelize } from '../db';

async function startWorker() {
  try {
    await sequelize.authenticate();
    console.log('DB connected');
    await connectRabbitMQ();
    console.log('RabbitMQ connected');
    await consumeNotifications();
    console.log('Worker consuming notifications...');
  } catch (err) {
    console.error(err);
  }
}

startWorker();
4️⃣ Notification Service
services/notificationService.ts
Ts
Copy code
import { Notification } from '../models/Notification';

export async function getAllNotifications() {
  return Notification.findAll({ order: [['createdAt', 'DESC']] });
}

export async function getUnreadCount() {
  return Notification.count({ where: { read: false } });
}

export async function markAsRead(id: string) {
  return Notification.update({ read: true }, { where: { id } });
}
5️⃣ Notification Controller
controllers/notificationController.ts
Ts
Copy code
import { Request, Response } from 'express';
import * as notificationService from '../services/notificationService';

export async function getNotifications(req: Request, res: Response) {
  const notifications = await notificationService.getAllNotifications();
  res.json(notifications);
}

export async function markNotificationRead(req: Request, res: Response) {
  const { id } = req.params;
  await notificationService.markAsRead(id);
  res.json({ success: true });
}
6️⃣ Notification Routes
routes/notificationRoutes.ts
Ts
Copy code
import { Router } from 'express';
import { getNotifications, markNotificationRead } from '../controllers/notificationController';

const router = Router();

router.get('/', getNotifications);
router.patch('/:id/read', markNotificationRead);

export default router;
In your main app.ts / server.ts:
Ts
Copy code
import notificationRoutes from './routes/notificationRoutes';
app.use('/notifications', notificationRoutes);
7️⃣ Publishing Notifications in Operational Service
Example in operationService.ts:
Ts
Copy code
import { publishNotification } from '../worker/rabbitmq/publisher';
import * as repo from '../repositories/operation-repository';

if (data.event_type === 'delay') {
  await repo.updateFlightStatus(data.flight_id, 'delayed');
  await publishNotification(data.flight_id, `Flight delayed by ${data.delay_minutes} minutes`);
}

if (data.event_type === 'cancellation') {
  await repo.updateFlightStatus(data.flight_id, 'cancelled');
  await publishNotification(data.flight_id, 'Flight cancelled');
}

notification-repo.ts:
Ts
Copy code
import { Notification } from '@/models'; // Sequelize model

export async function createNotification(flightId: string, message: string) {
  return Notification.create({ flight_id: flightId, message });
}
Step 8: Integrate with Operational Event service
In operationService.ts:
Ts
Copy code
import { publishNotification } from '@/worker-service/rabbitmq/publisher';

if (data.event_type === 'delay') {
  await repo.updateFlightStatus(data.flight_id, 'delayed');
  await publishNotification(data.flight_id, `Flight delayed by ${data.delay_minutes} mins`);
}

if (data.event_type === 'cancellation') {
  await repo.updateFlightStatus(data.flight_id, 'cancelled');
  await publishNotification(data.flight_id, 'Flight cancelled');
}
✅ This keeps your status update synchronous, but notifications are handled asynchronously in the queue.
Step 9: Frontend (React) Notification Bell
Create NotificationBell.tsx using shadcn components:
Tsx
Copy code
import { useEffect, useState } from 'react';
import { Button } from '@/components/ui/button';
import { DropdownMenu, DropdownMenuTrigger, DropdownMenuContent, DropdownMenuItem } from '@/components/ui/dropdown-menu';
import { api } from '@/api/api';

interface Notification {
  id: string;
  message: string;
  read: boolean;
}

export function NotificationBell() {
  const [notifications, setNotifications] = useState<Notification[]>([]);

  useEffect(() => {
    fetchNotifications();
  }, []);

  async function fetchNotifications() {
    const res = await api<{ data: Notification[] }>('/notifications/today');
    setNotifications(res.data);
  }

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button>🔔 {notifications.filter(n => !n.read).length}</Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        {notifications.map(n => (
          <DropdownMenuItem key={n.id}>{n.message}</DropdownMenuItem>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
Add this bell icon to your dashboard.
✅ Step 10: Run everything
Start all containers:
Bash
Copy code
docker compose up -d
Start worker service:
Bash
Copy code
cd apps/worker-service
ts-node index.ts
Test:
Add an operational event → status updates → notification appears in DB → frontend bell shows it.











++++++++++(((+((++++((+++(+(
[05/03, 13:51] Ashminita: Perfect — let’s do this step by step, clearly, with everything you need for RabbitMQ integration, notifications, and TypeScript. I’ll write it in a way that’s copy-paste ready. I’ll separate it by services, show folder structure, models, publisher, consumer, and routes.
1️⃣ Folder Structure
Copy code

/apps
  /operational-service
    /src
      /controllers
        operationalEventController.ts
      /services
        operationalEventService.ts
        rabbitmqPublisher.ts
      /routes
        operationalEventRoutes.ts
      /models
        OperationalEvent.ts
        Flight.ts
      /utils
        db.ts
        logger.ts
      server.ts
  /worker-service
    /src
      /consumers
        notificationConsumer.ts
      /services
        notificationService.ts
      /models
        Notification.ts
        Flight.ts
      /utils
        db.ts
        rabbitmq.ts
      server.ts
/docker
  docker-compose.yml
2️⃣ Notification Table Model (Worker Service)
Ts
Copy code
// /worker-service/src/models/Notification.ts
import { DataTypes, Model } from "sequelize";
import { sequelize } from "../utils/db";
import { Flight } from "./Flight";

export class Notification extends Model {
  id!: string;
  flight_id!: string;
  message!: string;
  created_at!: Date;
}

Notification.init(
  {
    id: {
      type: DataTypes.UUID,
      primaryKey: true,
      defaultValue: DataTypes.UUIDV4,
    },
    flight_id: {
      type: DataTypes.UUID,
      allowNull: false,
      references: { model: Flight, key: "id" },
    },
    message: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    created_at: {
      type: DataTypes.DATE,
      defaultValue: DataTypes.NOW,
    },
  },
  {
    sequelize,
    tableName: "notifications",
    timestamps: false,
  }
);

// association
Flight.hasMany(Notification, { foreignKey: "flight_id" });
Notification.belongsTo(Flight, { foreignKey: "flight_id" });
3️⃣ Publisher Code (Operational Service)
Ts
Copy code
// /operational-service/src/services/rabbitmqPublisher.ts
import amqp from "amqplib";

const QUEUE_NAME = "flight_status_notifications";

export async function publishNotification(message: any) {
  const connection = await amqp.connect(process.env.RABBITMQ_URL || "amqp://localhost");
  const channel = await connection.createChannel();
  await channel.assertQueue(QUEUE_NAME, { durable: true });
  channel.sendToQueue(QUEUE_NAME, Buffer.from(JSON.stringify(message)), { persistent: true });
  console.log("Published message:", message);
  await channel.close();
  await connection.close();
}
4️⃣ Trigger Publisher in Operational Event Service
Ts
Copy code
// /operational-service/src/services/operationalEventService.ts
import { publishNotification } from "./rabbitmqPublisher";
import { Flight } from "../models/Flight";
import { OperationalEvent } from "../models/OperationalEvent";

export async function createOperationalEvent(data: any) {
  const flight = await Flight.findByPk(data.flight_id);
  if (!flight) throw new Error("Flight not found");

  // Update status synchronously
  if (data.event_type === "delay") flight.status = "delayed";
  if (data.event_type === "cancellation") flight.status = "cancelled";
  if (data.event_type === "diversion") flight.status = "diverted";

  await flight.save();

  // Create operational event
  const event = await OperationalEvent.create(data);

  // Publish notification to queue
  await publishNotification({
    flight_id: flight.id,
    status: flight.status,
    message: `Flight ${flight.flight_number} is ${flight.status}`,
    created_at: new Date(),
  });

  return event;
}
5️⃣ Consumer Code (Worker Service)
Ts
Copy code
// /worker-service/src/consumers/notificationConsumer.ts
import amqp from "amqplib";
import { Notification } from "../models/Notification";

const QUEUE_NAME = "flight_status_notifications";

export async function startConsumer() {
  const connection = await amqp.connect(process.env.RABBITMQ_URL || "amqp://localhost");
  const channel = await connection.createChannel();
  await channel.assertQueue(QUEUE_NAME, { durable: true });

  channel.consume(
    QUEUE_NAME,
    async (msg) => {
      if (msg) {
        const data = JSON.parse(msg.content.toString());
        console.log("Received message:", data);

        // Save to notification table
        await Notification.create({
          flight_id: data.flight_id,
          message: data.message,
          created_at: new Date(data.created_at),
        });

        channel.ack(msg);
      }
    },
    { noAck: false }
  );
}
6️⃣ Start Consumer in Worker Service
Ts
Copy code
// /worker-service/src/server.ts
import { startConsumer } from "./consumers/notificationConsumer";

async function main() {
  console.log("Worker service started");
  await startConsumer();
}

main().catch(console.error);
7️⃣ Frontend Fetch Example (Shadcn UI / React)
Tsx
Copy code
// /frontend/src/components/NotificationBell.tsx
import { useEffect, useState } from "react";
import { Button } from "@/components/ui/button";
import { api } from "@/api/api";

export function NotificationBell() {
  const [notifications, setNotifications] = useState<any[]>([]);

  useEffect(() => {
    fetchNotifications();
  }, []);

  async function fetchNotifications() {
    const res = await api<{ success: boolean; data: any[] }>("/notifications/today");
    setNotifications(res.data);
  }

  return (
    <div className="relative">
      <Button>
        🔔 {notifications.length}
      </Button>
      <div className="absolute right-0 mt-2 w-64 bg-white text-black shadow-lg rounded">
        {notifications.map((n) => (
          <div key={n.id} className="p-2 border-b">
            {n.message}
          </div>
        ))}
      </div>
    </div>
  );
}
8️⃣ Docker Compose
Yaml
Copy code
version: "3.8"
services:
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq
    ports:
      - "5672:5672"   # messaging port
      - "15672:15672" # management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest

  operational-service:
    build:
      context: .
      dockerfile: apps/operational-service/Dockerfile
    env_file: .env
    depends_on:
      - rabbitmq

  worker-service:
    build:
      context: .
      dockerfile: apps/worker-service/Dockerfile
    env_file: .env
    depends_on:
      - rabbitmq
✅ Summary of Flow
User creates an operational event in Operational Service → status changes synchronously.
Operational Service publishes a notification message to RabbitMQ.
Worker Service consumes the message, saves it in notifications table.
Frontend fetches notifications for dashboard bell icon.
If you want, I can also give you the ready-to-use Sequelize models for Flight, OperationalEvent, and Notification, along with migration files, so you literally just copy-paste and run.
Do you want me to do that next?
[05/03, 13:52] Ashminita: Perfect! Now that you have the RabbitMQ setup, publisher, consumer, and notifications table, here’s how to test everything step by step in your environment. I’ll keep it simple and actionable.
1️⃣ Start Services
Make sure all services are up. From your project root (where docker-compose.yml is):
Bash
Copy code
docker-compose up -d
Check logs:
Bash
Copy code
docker-compose logs -f rabbitmq
docker-compose logs -f worker-service
docker-compose logs -f operational-service
RabbitMQ Management UI will be at http://localhost:15672⁠�
Default user/pass: guest / guest
✅ You should see the queue flight_status_notifications in RabbitMQ.
2️⃣ Test Publisher
You can test the publisher manually in Operational Service:
Use Postman or cURL to create an operational event:
Http
Copy code
POST http://localhost:3004/operational-events
Content-Type: application/json

{
  "flight_id": "your-flight-id",
  "event_type": "delay",
  "delay_minutes": 30,
  "severity": "medium",
  "description": "Delayed due to weather"
}
After the request:
The flight status should update immediately in your database (delayed).
You should see a log in worker-service console: Received message: {…}
Check your database notifications table:
Sql
Copy code
SELECT * FROM notifications;
You should see a row with flight_id and message like:
Copy code

Flight ABC123 is delayed
3️⃣ Test Consumer
The consumer is running in worker-service. Every time a new message is published to RabbitMQ:
It gets consumed automatically
It inserts a notification into the table.
You can verify in the console logs of the worker-service:
Bash
Copy code
docker-compose logs -f worker-service
Look for:
Copy code

Received message: { flight_id: "123", status: "delayed", message: "Flight ABC123 is delayed" }
4️⃣ Test Frontend
Make sure your frontend is running (docker-compose or npm run dev).
Go to the dashboard where you added the Notification Bell.
The bell icon should show the number of notifications.
Click it → see all messages from the notifications table.
5️⃣ Test Multiple Messages
Publish 2–3 events quickly (delays, cancellations) for different flights.
Worker service will queue them in RabbitMQ → consume → insert into notifications.
Frontend should show updated notification count each time you refresh or poll.
This demonstrates asynchronous processing: the front-end does not wait for RabbitMQ, the operational service does not block while sending email/notifications.
6️⃣ Optional: Testing via RabbitMQ UI
Go to http://localhost:15672⁠�
Login with guest/guest
Go to Queues → flight_status_notifications
You can see messages ready/acknowledged.
Try Publish message manually and see if the worker-service consumes it.
Tips
If messages are not being consumed, check RabbitMQ URL in .env. It should be like:
Copy code

RABBITMQ_URL=amqp://guest:guest@localhost:5672
Make sure worker-service is running before publishing events.
Logs are your friend: watch both operational-service and worker-service logs.
If you want, I can also give you a ready Postman collection to test create operational events → see notifications → check flight status so you can literally run all scenarios in one go.
Do you want me to do that?
