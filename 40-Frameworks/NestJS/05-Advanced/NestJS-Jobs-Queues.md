---
created: 2026-02-17
tags:
  - nestjs
  - jobs
  - queues
  - bull
  - workers
aliases:
  - NestJS Jobs Queues
  - NestJS Bull
related:
  - NestJS-MOC
  - NestJS-Events
  - NestJS-Microservices
---

# NestJS — Jobs & Queues

> [!SUMMARY] Обзор
> Очереди задач в NestJS: Bull, workers, delayed jobs, repeatable jobs, job processing.

---

## 📦 Установка

```bash
npm install @nestjs/bull bull
npm install -D @types/bull
```

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                  NestJS App                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Producer │  │  Queue   │  │ Consumer │          │
│  │          │→ │ (Redis)  │→ │(Worker)  │          │
│  └──────────┘  └──────────┘  └──────────┘          │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                    Redis                            │
│              (Queue Storage)                        │
└─────────────────────────────────────────────────────┘
```

---

## 🔧 Module Setup

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.forRoot({
      redis: {
        host: process.env.REDIS_HOST || 'localhost',
        port: Number(process.env.REDIS_PORT) || 6379,
        password: process.env.REDIS_PASSWORD,
      },
      defaultJobOptions: {
        removeOnComplete: 100,    // Keep 100 completed jobs
        removeOnFail: 1000,       // Keep 1000 failed jobs
        attempts: 3,              // Retry 3 times
        backoff: {
          type: 'exponential',
          delay: 1000,            // Start with 1s delay
        },
      },
    }),
  ],
})
export class AppModule {}
```

---

## 📬 Queue Configuration

```typescript
// mail/mail.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { MailService } from './mail.service';
import { SendEmailProcessor } from './send-email.processor';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'email',
      defaultJobOptions: {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 1000,
        },
        removeOnComplete: 100,
        removeOnFail: 1000,
      },
    }),
    BullModule.registerQueue({
      name: 'notifications',
      defaultJobOptions: {
        attempts: 5,
        backoff: {
          type: 'fixed',
          delay: 5000,
        },
      },
    }),
  ],
  providers: [MailService, SendEmailProcessor],
  exports: [BullModule],
})
export class MailModule {}
```

---

## 📥 Producer (Adding Jobs)

```typescript
// mail/mail.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { Queue } from 'bull';
import { InjectQueue } from '@nestjs/bull';

@Injectable()
export class MailService {
  constructor(
    @InjectQueue('email') private emailQueue: Queue,
    @InjectQueue('notifications') private notificationsQueue: Queue,
  ) {}

  // Add job to queue
  async sendWelcomeEmail(email: string, name: string) {
    await this.emailQueue.add(
      'welcome',  // Job type
      { email, name },  // Job data
      {
        priority: 1,  // High priority
        delay: 0,     // Immediate
      },
    );
  }

  // Delayed job
  async sendPasswordReset(email: string, token: string) {
    await this.emailQueue.add(
      'password-reset',
      { email, token },
      {
        delay: 5000,  // Send after 5 seconds
        priority: 2,
      },
    );
  }

  // Repeatable job (cron)
  async scheduleNewsletter() {
    await this.emailQueue.add(
      'newsletter',
      { type: 'weekly' },
      {
        repeat: {
          cron: '0 9 * * 1',  // Every Monday at 9 AM
          timeZone: 'UTC',
        },
      },
    );
  }

  // Bulk jobs
  async sendBulkEmails(emails: string[], subject: string, html: string) {
    const jobs = emails.map((email, index) => ({
      name: 'bulk',
      data: { email, subject, html },
      opts: {
        delay: index * 1000,  // Stagger by 1 second
        priority: 3,
      },
    }));

    await this.emailQueue.addBulk(jobs);
  }

  // Job with custom options
  async sendTransactionalEmail(data: any) {
    const job = await this.emailQueue.add('transactional', data, {
      attempts: 5,
      backoff: {
        type: 'exponential',
        delay: 2000,
      },
      timeout: 30000,  // 30 second timeout
      removeOnComplete: true,
      removeOnFail: true,
    });

    console.log(`Job ${job.id} added`);
    return job;
  }

  // Get job status
  async getJobStatus(jobId: string) {
    const job = await this.emailQueue.getJob(jobId);
    
    if (!job) {
      return { status: 'not_found' };
    }

    const state = await job.getState();
    return {
      id: job.id,
      name: job.name,
      state,
      data: job.data,
      failedReason: job.failedReason,
      attemptsMade: job.attemptsMade,
      processedOn: job.processedOn,
      finishedOn: job.finishedOn,
    };
  }
}
```

---

## 🛠️ Processor (Consumer)

```typescript
// mail/send-email.processor.ts
import { Process, Processor } from '@nestjs/bull';
import { Job } from 'bull';
import { MailService } from './mail.service';
import { Logger } from '@nestjs/common';

@Processor('email')
export class SendEmailProcessor {
  private readonly logger = new Logger(SendEmailProcessor.name);

  constructor(private mailService: MailService) {}

  @Process('welcome')
  async handleWelcome(job: Job) {
    this.logger.log(`Processing welcome email job ${job.id}`);
    
    try {
      await this.mailService.sendEmail({
        to: job.data.email,
        subject: 'Welcome!',
        html: `<h1>Welcome, ${job.data.name}!</h1>`,
      });
      
      this.logger.log(`Welcome email sent to ${job.data.email}`);
    } catch (error) {
      this.logger.error(`Failed to send welcome email: ${error.message}`);
      throw error;  // Will trigger retry
    }
  }

  @Process('password-reset')
  async handlePasswordReset(job: Job) {
    this.logger.log(`Processing password reset job ${job.id}`);
    
    await this.mailService.sendEmail({
      to: job.data.email,
      subject: 'Password Reset',
      html: `<a href="https://example.com/reset/${job.data.token}">Reset Password</a>`,
    });
  }

  @Process('newsletter')
  async handleNewsletter(job: Job) {
    this.logger.log(`Processing newsletter job ${job.id}`);
    
    // Get all subscribers
    const subscribers = await this.getSubscribers();
    
    // Send in batches
    for (const subscriber of subscribers) {
      await this.mailService.sendEmail({
        to: subscriber.email,
        subject: 'Weekly Newsletter',
        html: job.data.html,
      });
    }
  }

  @Process('bulk')
  async handleBulk(job: Job) {
    this.logger.log(`Processing bulk email job ${job.id}`);
    
    await this.mailService.sendEmail({
      to: job.data.email,
      subject: job.data.subject,
      html: job.data.html,
    });
  }

  @Process('transactional')
  async handleTransactional(job: Job) {
    this.logger.log(`Processing transactional email job ${job.id}`);
    
    await this.mailService.sendEmail({
      to: job.data.to,
      subject: job.data.subject,
      html: job.data.html,
    });
  }

  private async getSubscribers() {
    // Get subscribers from database
    return [];
  }

  private async sendEmail(options: any) {
    // Send email via SMTP/SNS
  }
}
```

---

## 📊 Queue Management

```typescript
// mail/mail.controller.ts
import { Controller, Get, Post, Param, Delete } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

@Controller('queues/email')
export class MailQueueController {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  // Get queue stats
  @Get('stats')
  async getStats() {
    const [waiting, active, completed, failed, delayed] = await Promise.all([
      this.emailQueue.getWaitingCount(),
      this.emailQueue.getActiveCount(),
      this.emailQueue.getCompletedCount(),
      this.emailQueue.getFailedCount(),
      this.emailQueue.getDelayedCount(),
    ]);

    return { waiting, active, completed, failed, delayed };
  }

  // Get jobs by state
  @Get('jobs/:state')
  async getJobs(@Param('state') state: string) {
    const jobs = await this.emailQueue.getJobs(state.split(','), 0, 100);
    
    return jobs.map(job => ({
      id: job.id,
      name: job.name,
      data: job.data,
      attemptsMade: job.attemptsMade,
      failedReason: job.failedReason,
    }));
  }

  // Get specific job
  @Get('job/:id')
  async getJob(@Param('id') id: string) {
    const job = await this.emailQueue.getJob(id);
    
    if (!job) {
      return { error: 'Job not found' };
    }

    const state = await job.getState();
    return {
      id: job.id,
      name: job.name,
      state,
      data: job.data,
      attemptsMade: job.attemptsMade,
      failedReason: job.failedReason,
      processedOn: job.processedOn,
      finishedOn: job.finishedOn,
    };
  }

  // Retry failed job
  @Post('job/:id/retry')
  async retryJob(@Param('id') id: string) {
    const job = await this.emailQueue.getJob(id);
    
    if (!job) {
      return { error: 'Job not found' };
    }

    await job.retry();
    return { message: 'Job retried' };
  }

  // Remove job
  @Delete('job/:id')
  async removeJob(@Param('id') id: string) {
    const job = await this.emailQueue.getJob(id);
    
    if (!job) {
      return { error: 'Job not found' };
    }

    await job.remove();
    return { message: 'Job removed' };
  }

  // Clean queue
  @Delete('clean')
  async cleanQueue() {
    await this.emailQueue.clean(1000, 'completed');
    await this.emailQueue.clean(1000, 'failed');
    return { message: 'Queue cleaned' };
  }

  // Pause queue
  @Post('pause')
  async pauseQueue() {
    await this.emailQueue.pause();
    return { message: 'Queue paused' };
  }

  // Resume queue
  @Post('resume')
  async resumeQueue() {
    await this.emailQueue.resume();
    return { message: 'Queue resumed' };
  }

  // Drain queue
  @Delete('drain')
  async drainQueue() {
    await this.emailQueue.obliterate({ force: true });
    return { message: 'Queue drained' };
  }
}
```

---

## 🎯 Event Listeners

```typescript
// mail/mail.events.ts
import { Injectable } from '@nestjs/common';
import { OnQueueEvent, OnQueueGlobalEvent } from '@nestjs/bull';
import { Job } from 'bull';
import { Logger } from '@nestjs/common';

@Injectable()
export class MailQueueEvents {
  private readonly logger = new Logger(MailQueueEvents.name);

  // Global events
  @OnQueueGlobalEvent('error')
  onError(error: Error) {
    this.logger.error(`Queue error: ${error.message}`, error.stack);
  }

  @OnQueueGlobalEvent('paused')
  onPaused() {
    this.logger.log('Queue paused');
  }

  @OnQueueGlobalEvent('resumed')
  onResumed() {
    this.logger.log('Queue resumed');
  }

  // Job events
  @OnQueueEvent('waiting')
  onWaiting(job: Job) {
    this.logger.debug(`Job ${job.id} waiting`);
  }

  @OnQueueEvent('active')
  onActive(job: Job) {
    this.logger.log(`Job ${job.id} active`);
  }

  @OnQueueEvent('completed')
  onCompleted(job: Job, result: any) {
    this.logger.log(`Job ${job.id} completed: ${result}`);
  }

  @OnQueueEvent('failed')
  onFailed(job: Job, error: Error) {
    this.logger.error(`Job ${job.id} failed: ${error.message}`, error.stack);
  }

  @OnQueueEvent('stalled')
  onStalled(job: Job) {
    this.logger.warn(`Job ${job.id} stalled`);
  }

  @OnQueueEvent('progress')
  onProgress(job: Job, progress: number | object) {
    this.logger.debug(`Job ${job.id} progress: ${progress}%`);
  }
}
```

---

## 🔄 Delayed & Repeatable Jobs

```typescript
// Scheduled jobs example
@Injectable()
export class ScheduledJobsService {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  // Delayed job
  async sendDelayedEmail(email: string, delayMs: number) {
    await this.emailQueue.add(
      'delayed',
      { email },
      { delay: delayMs },
    );
  }

  // Repeatable job (cron)
  async scheduleDailyReport() {
    await this.emailQueue.add(
      'daily-report',
      { type: 'daily' },
      {
        repeat: {
          cron: '0 8 * * *',  // Every day at 8 AM
          timeZone: 'UTC',
        },
      },
    );
  }

  // Repeatable with custom key
  async scheduleWeeklyReport(userId: string) {
    await this.emailQueue.add(
      'weekly-report',
      { userId },
      {
        repeat: {
          cron: '0 9 * * 1',  // Every Monday at 9 AM
          jobId: `weekly-${userId}`,  // Custom job ID for uniqueness
        },
      },
    );
  }

  // Remove repeatable job
  async cancelWeeklyReport(userId: string) {
    await this.emailQueue.removeRepeatableByKey(`weekly-${userId}`);
  }
}
```

---

## 📋 Environment

```bash
# .env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# Queue settings
QUEUE_DEFAULT_ATTEMPTS=3
QUEUE_DEFAULT_BACKOFF_DELAY=1000
QUEUE_DEFAULT_REMOVE_ON_COMPLETE=100
QUEUE_DEFAULT_REMOVE_ON_FAIL=1000
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Events]] — Event emitter
- [[NestJS-Microservices]] — Микросервисы
- [[NestJS-Redis]] — Redis integration
