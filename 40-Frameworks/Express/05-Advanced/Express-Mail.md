---
created: 2026-02-17
tags:
  - express
  - mail
  - email
  - nodemailer
aliases:
  - Express Mail
  - Express Email Nodemailer
related:
  - Express-MOC
  - Express-Deployment
---

# Express.js — Mail Service

> [!SUMMARY] Обзор
> Отправка email в Express.js: Nodemailer, templates, queues.

---

## Установка

```bash
npm install nodemailer
npm install -D @types/nodemailer
```

---

## Mail Config

```typescript
// config/mail.ts
import nodemailer from 'nodemailer';

export const transporter = nodemailer.createTransport({
  host: process.env.MAIL_HOST || 'smtp.mailtrap.io',
  port: Number(process.env.MAIL_PORT) || 587,
  secure: false, // true for 465, false for other ports
  auth: {
    user: process.env.MAIL_USER,
    pass: process.env.MAIL_PASSWORD,
  },
  tls: {
    rejectUnauthorized: process.env.NODE_ENV === 'production',
  },
});

// Verify connection
transporter.verify((error, success) => {
  if (error) {
    console.error('Mail connection failed:', error);
  } else {
    console.log('Mail server ready');
  }
});
```

---

## Mail Service

```typescript
// services/mail.service.ts
import { transporter } from '../config/mail';

export interface MailOptions {
  to: string | string[];
  subject: string;
  template?: string;
  context?: any;
  html?: string;
  text?: string;
  attachments?: any[];
}

export class MailService {
  // Send email
  async send(options: MailOptions): Promise<void> {
    const mailOptions = {
      from: process.env.MAIL_FROM || '"No Reply" <noreply@example.com>',
      to: options.to,
      subject: options.subject,
      html: options.html,
      text: options.text,
      attachments: options.attachments,
    };

    await transporter.sendMail(mailOptions);
  }

  // Welcome email
  async sendWelcomeEmail(email: string, name: string): Promise<void> {
    await this.send({
      to: email,
      subject: 'Welcome to our platform!',
      html: `
        <h1>Welcome, ${name}!</h1>
        <p>Thank you for joining us.</p>
        <p>Get started by <a href="https://example.com/login">logging in</a>.</p>
      `,
      text: `Welcome, ${name}! Thank you for joining us.`,
    });
  }

  // Password reset
  async sendPasswordReset(email: string, token: string): Promise<void> {
    const resetUrl = `https://example.com/reset-password?token=${token}`;
    
    await this.send({
      to: email,
      subject: 'Password Reset Request',
      html: `
        <h1>Password Reset</h1>
        <p>Click the link below to reset your password:</p>
        <a href="${resetUrl}">${resetUrl}</a>
        <p>This link expires in 1 hour.</p>
        <p>If you didn't request this, ignore this email.</p>
      `,
      text: `Reset your password: ${resetUrl}`,
    });
  }

  // Email verification
  async sendVerificationEmail(email: string, code: string): Promise<void> {
    await this.send({
      to: email,
      subject: 'Verify your email',
      html: `
        <h1>Email Verification</h1>
        <p>Your verification code: <strong>${code}</strong></p>
        <p>Enter this code in the app to verify your email.</p>
      `,
      text: `Your verification code: ${code}`,
    });
  }

  // Newsletter
  async sendNewsletter(emails: string[], subject: string, html: string): Promise<void> {
    // Send in batches to avoid rate limits
    const batchSize = 50;
    
    for (let i = 0; i < emails.length; i += batchSize) {
      const batch = emails.slice(i, i + batchSize);
      
      await Promise.all(
        batch.map(email =>
          this.send({
            to: email,
            subject,
            html,
          })
        )
      );
      
      // Delay between batches
      if (i + batchSize < emails.length) {
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
    }
  }

  // With attachment
  async sendWithAttachment(
    email: string,
    subject: string,
    html: string,
    filePath: string,
    filename: string,
  ): Promise<void> {
    await this.send({
      to: email,
      subject,
      html,
      attachments: [
        {
          filename,
          path: filePath,
        },
      ],
    });
  }
}
```

---

## Usage in Controller

```typescript
// controllers/auth.controller.ts
import { Request, Response } from 'express';
import { MailService } from '../services/mail.service';
import { AuthService } from '../services/auth.service';

export class AuthController {
  constructor(
    private authService: AuthService,
    private mailService: MailService,
  ) {}

  async register(req: Request, res: Response) {
    const { email, password, name } = req.body;

    // Create user
    const user = await this.authService.register({ email, password, name });

    // Send welcome email (non-blocking)
    this.mailService.sendWelcomeEmail(user.email, user.name).catch(err => {
      console.error('Failed to send welcome email:', err);
      // Don't fail the request, just log the error
    });

    res.status(201).json({ message: 'User created', data: user });
  }

  async forgotPassword(req: Request, res: Response) {
    const { email } = req.body;

    const user = await this.authService.findByEmail(email);
    
    if (!user) {
      // Don't reveal if email exists
      return res.json({ message: 'If email exists, reset link sent' });
    }

    // Generate reset token
    const token = await this.authService.generateResetToken(user.id);

    // Send reset email
    await this.mailService.sendPasswordReset(user.email, token);

    res.json({ message: 'If email exists, reset link sent' });
  }

  async verifyEmail(req: Request, res: Response) {
    const { email } = req.body;

    const user = await this.authService.findByEmail(email);
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    // Generate verification code
    const code = Math.random().toString(36).substring(2, 8).toUpperCase();

    // Send verification email
    await this.mailService.sendVerificationEmail(user.email, code);

    res.json({ message: 'Verification code sent' });
  }
}
```

---

## Environment

```bash
# .env
# Mailtrap (for development)
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=587
MAIL_USER=your-mailtrap-user
MAIL_PASSWORD=your-mailtrap-password
MAIL_FROM="No Reply" <noreply@example.com>

# Production (Gmail example)
# MAIL_HOST=smtp.gmail.com
# MAIL_PORT=465
# MAIL_USER=your-gmail@gmail.com
# MAIL_PASSWORD=your-app-password
# MAIL_FROM="Your App" <your-gmail@gmail.com>

# Production (SendGrid example)
# MAIL_HOST=smtp.sendgrid.net
# MAIL_PORT=587
# MAIL_USER=apikey
# MAIL_PASSWORD=your-sendgrid-api-key
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Deployment]] — Deployment
