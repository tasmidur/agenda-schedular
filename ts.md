To convert the Node.js application into a **TypeScript package**, we need to:

1. Add TypeScript support.
2. Define types for jobs, events, and services.
3. Compile TypeScript to JavaScript.
4. Package the application for reuse.

Here’s the step-by-step process:

---

### Step 1: Initialize the Project
1. Create a new folder and initialize the project:
   ```bash
   mkdir nodejs-scheduler-ts
   cd nodejs-scheduler-ts
   npm init -y
   ```

2. Install TypeScript and required dependencies:
   ```bash
   npm install typescript ts-node @types/node @types/mongoose @types/agenda agenda mongoose dotenv
   ```

3. Initialize TypeScript configuration:
   ```bash
   npx tsc --init
   ```

4. Update `tsconfig.json` for better TypeScript settings:
   ```json
   {
     "compilerOptions": {
       "target": "ES2020",
       "module": "CommonJS",
       "outDir": "./dist",
       "rootDir": "./src",
       "strict": true,
       "esModuleInterop": true,
       "skipLibCheck": true,
       "forceConsistentCasingInFileNames": true
     },
     "include": ["src/**/*"],
     "exclude": ["node_modules"]
   }
   ```

---

### Step 2: Folder Structure
```
nodejs-scheduler-ts/
├── src/
│   ├── config/              # Configuration files
│   │   ├── db.ts
│   │   └── jobs.ts          # Job configurations and handlers
│   ├── core/                # Core application logic
│   │   ├── events/          # Event definitions
│   │   │   └── userEvents.ts
│   │   └── services/        # Service classes
│   │       ├── agendaService.ts
│   │       ├── eventService.ts
│   │       └── jobService.ts
│   ├── infrastructure/      # Infrastructure setup
│   │   └── agendaSetup.ts   # Agenda initialization
│   ├── types/               # Custom types
│   │   └── index.ts
│   └── index.ts             # Entry point
├── .env                     # Environment variables
├── package.json
├── tsconfig.json
└── README.md
```

---

### Step 3: Environment Variables (`.env`)
Add MongoDB connection details:
```env
MONGO_URI=mongodb://localhost:27017/agenda
```

---

### Step 4: Database Configuration (`src/config/db.ts`)
Set up MongoDB connection:
```typescript
import mongoose from 'mongoose';
import 'dotenv/config';

const connectDB = async (): Promise<void> => {
  try {
    await mongoose.connect(process.env.MONGO_URI as string, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB connected...');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

export default connectDB;
```

---

### Step 5: Job Configuration and Handlers (`src/config/jobs.ts`)
Define jobs, their handlers, and schedules:
```typescript
import { Job } from 'agenda';
import { JobHandler } from '../types';

// Job Handlers
const processTask: JobHandler = async (job: Job) => {
  const { data } = job.attrs;
  console.log('Processing task with data:', data);
  // Add your job logic here
};

const sendEmail: JobHandler = async (job: Job) => {
  const { data } = job.attrs;
  console.log('Sending email to:', data.email);
  // Add your email sending logic here
};

// Job Configuration
export const jobs = [
  {
    name: 'process-task',
    handler: processTask, // Direct reference to the handler
    schedule: '0 12 * * *', // Daily at 12 PM (noon)
  },
  {
    name: 'send-email',
    handler: sendEmail, // Direct reference to the handler
    schedule: '2025-02-01T00:00:00', // Specific date: 01-02-2025 at midnight
  },
];
```

---

### Step 6: Custom Types (`src/types/index.ts`)
Define types for job handlers and other reusable types:
```typescript
import { Job } from 'agenda';

export type JobHandler = (job: Job) => Promise<void>;

export interface JobConfig {
  name: string;
  handler: JobHandler;
  schedule?: string | Date;
}
```

---

### Step 7: Agenda Setup (`src/infrastructure/agendaSetup.ts`)
Initialize Agenda and register jobs:
```typescript
import Agenda from 'agenda';
import 'dotenv/config';
import { jobs } from '../config/jobs';
import { JobConfig } from '../types';

const agenda = new Agenda({ db: { address: process.env.MONGO_URI as string } });

// Register jobs
jobs.forEach((job: JobConfig) => {
  agenda.define(job.name, job.handler);

  if (job.schedule) {
    if (typeof job.schedule === 'string' && job.schedule.split(' ').length === 5) {
      // Cron syntax
      agenda.every(job.schedule, job.name);
    } else {
      // Specific date
      agenda.schedule(new Date(job.schedule), job.name);
    }
  }
});

export default agenda;
```

---

### Step 8: Agenda Service (`src/core/services/agendaService.ts`)
Create a service to interact with Agenda:
```typescript
import agenda from '../../infrastructure/agendaSetup';

class AgendaService {
  async start(): Promise<void> {
    await agenda.start();
    console.log('Agenda scheduler started...');
  }

  queueJob(name: string, data: Record<string, unknown> = {}): void {
    agenda.now(name, data);
  }
}

export default new AgendaService();
```

---

### Step 9: Event Service (`src/core/services/eventService.ts`)
Create a service to handle event emission and listening:
```typescript
import { EventEmitter } from 'events';
import agendaService from './agendaService';

class EventService extends EventEmitter {
  constructor() {
    super();
    this.setupListeners();
  }

  setupListeners(): void {
    // Listen for user.signup event
    this.on('user.signup', (userData: Record<string, unknown>) => {
      console.log('User signed up:', userData);
      // Queue a job
      agendaService.queueJob('process-task', { data: userData });
    });

    // Listen for send.email event
    this.on('send.email', (emailData: Record<string, unknown>) => {
      console.log('Email requested:', emailData);
      // Queue a job
      agendaService.queueJob('send-email', { data: emailData });
    });
  }
}

export default new EventService();
```

---

### Step 10: Event Definitions (`src/core/events/userEvents.ts`)
Emit events to test the system:
```typescript
import eventService from '../services/eventService';

// Simulate a user signing up
const userData = { id: 1, name: 'John Doe', email: 'john@example.com' };
eventService.emit('user.signup', userData);

// Simulate sending an email
const emailData = { email: 'john@example.com', subject: 'Welcome!', body: 'Thank you for signing up.' };
eventService.emit('send.email', emailData);
```

---

### Step 11: Entry Point (`src/index.ts`)
Initialize the application:
```typescript
import connectDB from './config/db';
import agendaService from './core/services/agendaService';
import './core/events/userEvents'; // Emit events

// Connect to MongoDB
connectDB();

// Start Agenda
agendaService.start();
```

---

### Step 12: Compile and Run
1. Compile TypeScript to JavaScript:
   ```bash
   npx tsc
   ```

2. Run the application:
   ```bash
   node dist/index.js
   ```

---

### Step 13: Package for Reuse
1. Add a `main` entry point to `package.json`:
   ```json
   "main": "dist/index.js"
   ```

2. Add a `build` script to `package.json`:
   ```json
   "scripts": {
     "build": "tsc",
     "start": "node dist/index.js"
   }
   ```

3. Publish the package to npm (optional):
   ```bash
   npm publish
   ```

---

### Final Output
When you run the application, you should see the following in the console:
```
MongoDB connected...
Agenda scheduler started...
User signed up: { id: 1, name: 'John Doe', email: 'john@example.com' }
Processing task with data: { id: 1, name: 'John Doe', email: 'john@example.com' }
Email requested: { email: 'john@example.com', subject: 'Welcome!', body: 'Thank you for signing up.' }
```

---

### Key Features
1. **TypeScript Support**:
   - Strong typing for better code quality and maintainability.

2. **Modular Design**:
   - Separation of concerns using services, configurations, and infrastructure.

3. **Reusable Package**:
   - Compiled into a reusable npm package.

This TypeScript implementation adheres to **SOLID principles** and **best practices**, making it scalable and maintainable. Let me know if you need further assistance!
