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
