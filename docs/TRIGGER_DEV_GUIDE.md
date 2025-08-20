# Complete Trigger.dev Guide for Production-Ready Background Jobs & Cron

## Overview

This comprehensive guide covers the implementation of Trigger.dev v3 for production-ready background job processing, cron scheduling, and task management based on Papermark's real-world implementation. Trigger.dev provides reliable, scalable background job processing with advanced features like retry logic, concurrency control, and monitoring.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Setup & Configuration](#setup--configuration)
3. [Cron Jobs & Scheduling](#cron-jobs--scheduling)
4. [Background Tasks](#background-tasks)
5. [Queue Management](#queue-management)
6. [Error Handling & Retries](#error-handling--retries)
7. [File Processing Tasks](#file-processing-tasks)
8. [Email & Notification Tasks](#email--notification-tasks)
9. [Data Export & Processing](#data-export--processing)
10. [Task Chaining & Workflows](#task-chaining--workflows)
11. [Monitoring & Logging](#monitoring--logging)
12. [Performance Optimization](#performance-optimization)
13. [Deployment & Production](#deployment--production)
14. [Testing Strategies](#testing-strategies)
15. [Best Practices](#best-practices)

---

## Architecture Overview

### Trigger.dev v3 Architecture

Trigger.dev v3 provides a robust background job processing system with the following key components:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Your App      │    │   Trigger.dev   │    │   Workers       │
│                 │    │   Platform      │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │ Task Trigger│ ├────┤ │ Task Queue  │ ├────┤ │ Task Runner │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
│                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │ Cron Jobs   │ ├────┤ │ Scheduler   │ ├────┤ │ Extensions  │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Key Features

- **Reliable Execution**: Built-in retry logic and error handling
- **Scalable**: Automatic scaling based on workload
- **Scheduled Tasks**: Cron-based scheduling with timezone support
- **Queue Management**: Concurrency limits and prioritization
- **Monitoring**: Real-time task monitoring and logging
- **Extensions**: Built-in support for databases, file processing, etc.

---

## Setup & Configuration

### 1. Installation

```bash
npm install @trigger.dev/sdk@beta
npm install @trigger.dev/build@beta
```

### 2. Trigger Configuration (`trigger.config.ts`)

```typescript
import { ffmpeg } from "@trigger.dev/build/extensions/core";
import { prismaExtension } from "@trigger.dev/build/extensions/prisma";
import { defineConfig, timeout } from "@trigger.dev/sdk/v3";

export default defineConfig({
  project: "proj_your_project_id", // Get from Trigger.dev dashboard
  dirs: ["./lib/trigger"], // Directory containing your tasks
  maxDuration: timeout.None, // No max duration limit
  
  // Retry configuration
  retries: {
    enabledInDev: false, // Disable retries in development
    default: {
      maxAttempts: 3,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 10000,
      factor: 2, // Exponential backoff
      randomize: true, // Add jitter to retries
    },
  },
  
  // Build extensions
  build: {
    extensions: [
      // Prisma extension for database access
      prismaExtension({
        schema: "prisma/schema/schema.prisma",
      }),
      // FFmpeg for video processing
      ffmpeg(),
    ],
  },
});
```

### 3. Environment Variables

```bash
# .env.local

# Trigger.dev
TRIGGER_SECRET_KEY=tr_dev_your_secret_key
TRIGGER_PUBLIC_API_KEY=tr_public_your_public_key

# For production
TRIGGER_DEPLOYMENT_ID=your_deployment_id

# Database (for Prisma extension)
DATABASE_URL="postgresql://user:password@localhost:5432/db"

# Other service integrations
INTERNAL_API_KEY=your_internal_api_key
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

### 4. Package.json Scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "trigger:dev": "npx trigger.dev@beta dev",
    "trigger:deploy": "npx trigger.dev@beta deploy",
    "trigger:build": "npx trigger.dev@beta build"
  }
}
```

---

## Cron Jobs & Scheduling

### Basic Cron Task Implementation

```typescript
// lib/trigger/cleanup-expired-exports.ts
import { logger, schedules } from "@trigger.dev/sdk/v3";
import { del } from "@vercel/blob";
import { jobStore } from "@/lib/redis-job-store";

export const cleanupExpiredExports = schedules.task({
  id: "cleanup-expired-exports",
  // Run daily at 2 AM UTC
  cron: "0 2 * * *",
  run: async (payload) => {
    logger.info("Starting cleanup of expired export blobs", {
      timestamp: payload.timestamp,
      scheduledTime: payload.scheduledTime,
    });

    try {
      // Get all blob URLs that are due for cleanup
      const blobsToCleanup = await jobStore.getBlobsForCleanup();

      if (blobsToCleanup.length === 0) {
        logger.info("No blobs due for cleanup");
        return { deletedCount: 0 };
      }

      logger.info(`Found ${blobsToCleanup.length} blobs to delete`);

      // Delete blobs with error handling
      const deletionResults = await Promise.allSettled(
        blobsToCleanup.map(async (blob) => {
          try {
            await del(blob.blobUrl);
            await jobStore.removeBlobFromCleanupQueue(blob.blobUrl, blob.jobId);
            
            logger.info("Successfully deleted blob", {
              blobUrl: blob.blobUrl,
              jobId: blob.jobId,
            });

            return { blob, success: true };
          } catch (error) {
            logger.error("Failed to delete blob", {
              blobUrl: blob.blobUrl,
              jobId: blob.jobId,
              error: error instanceof Error ? error.message : String(error),
            });
            return { blob, success: false, error };
          }
        }),
      );

      const successCount = deletionResults.filter(
        (result) => result.status === "fulfilled" && result.value.success,
      ).length;

      const failureCount = deletionResults.length - successCount;

      logger.info("Cleanup completed", {
        totalBlobs: blobsToCleanup.length,
        successCount,
        failureCount,
      });

      return {
        deletedCount: successCount,
        failureCount,
        totalProcessed: blobsToCleanup.length,
      };
    } catch (error) {
      logger.error("Cleanup task failed", {
        error: error instanceof Error ? error.message : String(error),
      });
      throw error; // Re-throw to trigger retry
    }
  },
});
```

### Advanced Cron Patterns

```typescript
// lib/trigger/scheduled-tasks.ts
import { logger, schedules } from "@trigger.dev/sdk/v3";

// Multiple cron schedules
export const hourlyHealthCheck = schedules.task({
  id: "hourly-health-check",
  cron: "0 * * * *", // Every hour
  run: async (payload) => {
    // Health check logic
    const healthChecks = await performHealthChecks();
    return { status: "healthy", checks: healthChecks };
  },
});

export const weeklyReports = schedules.task({
  id: "weekly-reports",
  cron: "0 9 * * 1", // Every Monday at 9 AM
  timezone: "America/New_York", // Specify timezone
  run: async (payload) => {
    // Generate weekly reports
    const reportData = await generateWeeklyReport();
    await sendWeeklyReport(reportData);
    return { reportGenerated: true, timestamp: payload.timestamp };
  },
});

export const monthlyCleanup = schedules.task({
  id: "monthly-cleanup",
  cron: "0 3 1 * *", // First day of month at 3 AM
  run: async (payload) => {
    // Monthly data cleanup
    const deletedRecords = await cleanupOldData();
    return { deletedRecords };
  },
});

// Dynamic scheduling based on conditions
export const conditionalTask = schedules.task({
  id: "conditional-task",
  cron: "*/10 * * * *", // Every 10 minutes
  run: async (payload) => {
    // Check if task should run
    const shouldRun = await checkConditions();
    
    if (!shouldRun) {
      logger.info("Conditions not met, skipping task");
      return { skipped: true };
    }

    // Execute task
    await performConditionalTask();
    return { executed: true };
  },
});
```

### Cron Expression Examples

```typescript
// Common cron patterns
const cronPatterns = {
  // Every minute
  everyMinute: "* * * * *",
  
  // Every 5 minutes
  every5Minutes: "*/5 * * * *",
  
  // Every hour at minute 0
  everyHour: "0 * * * *",
  
  // Every day at 2:30 AM
  daily: "30 2 * * *",
  
  // Every Monday at 9 AM
  weeklyMonday: "0 9 * * 1",
  
  // First day of every month at midnight
  monthly: "0 0 1 * *",
  
  // Every 15 minutes during business hours (9-17)
  businessHours: "*/15 9-17 * * 1-5",
  
  // Every Sunday at 3 AM
  weeklySunday: "0 3 * * 0",
};
```

---

## Background Tasks

### Basic Task Definition

```typescript
// lib/trigger/email-tasks.ts
import { logger, task } from "@trigger.dev/sdk/v3";
import { sendDataroomInfoEmail } from "@/lib/emails/send-dataroom-info";
import { sendDataroomTrialEndEmail } from "@/lib/emails/send-dataroom-trial-end";
import prisma from "@/lib/prisma";

export const sendDataroomTrialInfoEmailTask = task({
  id: "send-dataroom-trial-info-email",
  retry: { maxAttempts: 3 },
  run: async (payload: { to: string; name?: string }) => {
    logger.info("Sending dataroom trial info email", { to: payload.to });
    
    await sendDataroomInfoEmail({ 
      user: { 
        email: payload.to, 
        name: payload.name || "User" 
      } 
    });
    
    logger.info("Email sent successfully", { to: payload.to });
    return { success: true, emailSent: true };
  },
});

export const sendDataroomTrialExpiredEmailTask = task({
  id: "send-dataroom-trial-expired-email",
  retry: { maxAttempts: 3 },
  run: async (payload: { to: string; name: string; teamId: string }) => {
    logger.info("Processing trial expiration", { teamId: payload.teamId });

    const team = await prisma.team.findUnique({
      where: { id: payload.teamId },
      select: { plan: true },
    });

    if (!team) {
      logger.error("Team not found", { teamId: payload.teamId });
      throw new Error("Team not found");
    }

    // Check if team still has trial
    if (team.plan.includes("drtrial")) {
      // Send expiration email
      await sendDataroomTrialEndEmail({
        email: payload.to,
        name: payload.name,
      });
      
      logger.info("Trial expiration email sent", { 
        to: payload.to, 
        teamId: payload.teamId 
      });

      // Update team plan (remove trial)
      const updatedTeam = await prisma.team.update({
        where: { id: payload.teamId },
        data: { plan: team.plan.replace("+drtrial", "") },
      });

      const isPaid = [
        "pro",
        "business", 
        "datarooms",
        "datarooms-plus",
      ].includes(updatedTeam.plan);

      if (!isPaid) {
        // Clean up trial features
        await prisma.brand.deleteMany({
          where: { teamId: payload.teamId },
        });

        // Block non-admin users
        const blockedUsers = await prisma.userTeam.updateMany({
          where: {
            teamId: payload.teamId,
            role: { not: "ADMIN" },
          },
          data: {
            status: "BLOCKED_TRIAL_EXPIRED",
            blockedAt: new Date(),
          },
        });

        logger.info("Trial cleanup completed", {
          teamId: payload.teamId,
          blockedUsers: blockedUsers.count,
        });
      }

      return { 
        success: true, 
        action: "trial_expired",
        emailSent: true,
        teamUpdated: true 
      };
    }

    logger.info("Team already upgraded - no action needed", {
      teamId: payload.teamId,
      plan: team.plan,
    });
    
    return { 
      success: true, 
      action: "already_upgraded",
      emailSent: false 
    };
  },
});
```

### Task Triggering from API Routes

```typescript
// pages/api/teams/[teamId]/trial-expiry.ts
import { NextApiRequest, NextApiResponse } from "next";
import { sendDataroomTrialExpiredEmailTask } from "@/lib/trigger/email-tasks";

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse,
) {
  if (req.method !== "POST") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const { teamId } = req.query;
  const { userEmail, userName } = req.body;

  try {
    // Trigger the background task
    const handle = await sendDataroomTrialExpiredEmailTask.trigger(
      {
        to: userEmail,
        name: userName,
        teamId: teamId as string,
      },
      {
        // Optional: Set idempotency key to prevent duplicate runs
        idempotencyKey: `trial-expiry-${teamId}-${Date.now()}`,
        
        // Optional: Add tags for filtering/monitoring
        tags: [
          `team:${teamId}`,
          `action:trial-expiry`,
          `user:${userEmail}`,
        ],
        
        // Optional: Delay execution
        delay: "5m", // Run in 5 minutes
      }
    );

    logger.info("Trial expiry task triggered", {
      taskId: handle.id,
      teamId,
      userEmail,
    });

    return res.status(200).json({
      success: true,
      taskId: handle.id,
      message: "Trial expiry process initiated",
    });
  } catch (error) {
    logger.error("Failed to trigger trial expiry task", {
      error: error instanceof Error ? error.message : String(error),
      teamId,
      userEmail,
    });

    return res.status(500).json({
      error: "Failed to initiate trial expiry process",
    });
  }
}
```

---

## Queue Management

### Concurrency Control

```typescript
// lib/trigger/file-processing.ts
import { logger, task } from "@trigger.dev/sdk/v3";

export const convertFilesToPdfTask = task({
  id: "convert-files-to-pdf",
  retry: { maxAttempts: 3 },
  // Limit concurrent executions
  queue: {
    concurrencyLimit: 10, // Maximum 10 tasks running simultaneously
  },
  run: async (payload: ConvertPayload) => {
    // File conversion logic
  },
});

export const convertCadToPdfTask = task({
  id: "convert-cad-to-pdf",
  retry: { maxAttempts: 3 },
  // Lower concurrency for resource-intensive tasks
  queue: {
    concurrencyLimit: 2,
  },
  run: async (payload: ConvertPayload) => {
    // CAD conversion logic
  },
});

export const processVideo = task({
  id: "process-video",
  // Use specific machine preset for video processing
  machine: {
    preset: "medium-1x", // or "large-1x", "small-1x"
  },
  queue: {
    concurrencyLimit: 1, // One video at a time
  },
  run: async (payload: VideoProcessingPayload) => {
    // Video processing logic
  },
});
```

### Priority Queues

```typescript
// lib/trigger/priority-tasks.ts
import { logger, task } from "@trigger.dev/sdk/v3";

// High priority task
export const urgentNotificationTask = task({
  id: "urgent-notification",
  queue: {
    name: "urgent", // Separate queue for urgent tasks
    concurrencyLimit: 50,
  },
  run: async (payload: NotificationPayload) => {
    // Urgent notification logic
  },
});

// Normal priority task
export const regularProcessingTask = task({
  id: "regular-processing",
  queue: {
    name: "regular",
    concurrencyLimit: 10,
  },
  run: async (payload: ProcessingPayload) => {
    // Regular processing logic
  },
});

// Low priority task
export const batchReportTask = task({
  id: "batch-report",
  queue: {
    name: "low-priority",
    concurrencyLimit: 2,
  },
  run: async (payload: ReportPayload) => {
    // Batch report generation
  },
});
```

### Queue Monitoring

```typescript
// lib/trigger/queue-monitoring.ts
import { logger, task } from "@trigger.dev/sdk/v3";

export const queueHealthCheck = task({
  id: "queue-health-check",
  run: async () => {
    // Monitor queue health
    const queueStats = await getQueueStatistics();
    
    if (queueStats.urgentQueue.pending > 100) {
      logger.warn("Urgent queue backlog detected", {
        pending: queueStats.urgentQueue.pending,
      });
      
      // Send alert
      await sendQueueAlert("urgent", queueStats.urgentQueue.pending);
    }

    return queueStats;
  },
});
```

---

## Error Handling & Retries

### Retry Configuration

```typescript
// lib/trigger/resilient-tasks.ts
import { logger, task, retry } from "@trigger.dev/sdk/v3";

export const resilientApiCallTask = task({
  id: "resilient-api-call",
  // Custom retry configuration
  retry: {
    maxAttempts: 5,
    minTimeoutInMs: 1000,
    maxTimeoutInMs: 30000,
    factor: 2,
    randomize: true,
  },
  run: async (payload: ApiCallPayload) => {
    try {
      // Use built-in retry.fetch for external API calls
      const response = await retry.fetch(
        `https://api.external-service.com/convert`,
        {
          method: "POST",
          body: JSON.stringify(payload.data),
          headers: {
            "Content-Type": "application/json",
            "Authorization": `Bearer ${process.env.API_KEY}`,
          },
          retry: {
            byStatus: {
              // Retry on specific status codes
              "500-599": {
                strategy: "backoff",
                maxAttempts: 3,
                factor: 2,
                minTimeoutInMs: 1000,
                maxTimeoutInMs: 30000,
                randomize: false,
              },
              "429": {
                // Handle rate limiting
                strategy: "backoff",
                maxAttempts: 5,
                factor: 2,
                minTimeoutInMs: 5000,
                maxTimeoutInMs: 60000,
                randomize: true,
              },
            },
          },
        }
      );

      if (!response.ok) {
        throw new Error(`API call failed: ${response.status}`);
      }

      const result = await response.json();
      
      logger.info("API call successful", {
        url: response.url,
        status: response.status,
      });

      return result;
    } catch (error) {
      logger.error("API call failed after retries", {
        error: error instanceof Error ? error.message : String(error),
        payload,
      });
      throw error;
    }
  },
});
```

### Error Classification

```typescript
// lib/trigger/error-handling.ts
import { logger, task } from "@trigger.dev/sdk/v3";

// Custom error classes
export class RetryableError extends Error {
  constructor(message: string, public code?: string) {
    super(message);
    this.name = "RetryableError";
  }
}

export class PermanentError extends Error {
  constructor(message: string, public code?: string) {
    super(message);
    this.name = "PermanentError";
  }
}

export const smartRetryTask = task({
  id: "smart-retry-task",
  retry: {
    maxAttempts: 3,
    minTimeoutInMs: 1000,
    maxTimeoutInMs: 10000,
    factor: 2,
  },
  run: async (payload: ProcessingPayload) => {
    try {
      await performProcessing(payload);
      return { success: true };
    } catch (error) {
      if (error instanceof Error) {
        // Classify errors
        if (error.message.includes("rate limit")) {
          // Temporary error - will retry
          throw new RetryableError("Rate limit exceeded, will retry", "RATE_LIMIT");
        }
        
        if (error.message.includes("invalid input")) {
          // Permanent error - won't retry
          throw new PermanentError("Invalid input data", "INVALID_INPUT");
        }
        
        if (error.message.includes("network")) {
          // Temporary network error - will retry
          throw new RetryableError("Network error, will retry", "NETWORK_ERROR");
        }
      }
      
      // Unknown error - let default retry logic handle it
      throw error;
    }
  },
});
```

### Dead Letter Queue Handling

```typescript
// lib/trigger/dlq-handler.ts
import { logger, task } from "@trigger.dev/sdk/v3";

export const handleFailedTask = task({
  id: "handle-failed-task",
  run: async (payload: {
    originalTaskId: string;
    originalPayload: any;
    errorDetails: string;
    attemptCount: number;
  }) => {
    logger.warn("Handling failed task", {
      originalTaskId: payload.originalTaskId,
      attemptCount: payload.attemptCount,
      error: payload.errorDetails,
    });

    // Store failed task information
    await prisma.failedTask.create({
      data: {
        taskId: payload.originalTaskId,
        payload: payload.originalPayload,
        error: payload.errorDetails,
        attempts: payload.attemptCount,
        failedAt: new Date(),
      },
    });

    // Notify administrators
    await notifyAdmins({
      subject: `Task Failed: ${payload.originalTaskId}`,
      details: payload.errorDetails,
      payload: payload.originalPayload,
    });

    // Optionally, try alternative processing
    if (payload.originalTaskId === "critical-process") {
      await triggerFallbackProcess(payload.originalPayload);
    }

    return { handled: true, notified: true };
  },
});
```

---

## File Processing Tasks

### Video Processing

```typescript
// lib/trigger/optimize-video-files.ts
import { logger, task } from "@trigger.dev/sdk/v3";
import ffmpeg from "fluent-ffmpeg";
import { createReadStream, createWriteStream } from "fs";
import fs from "fs/promises";
import fetch from "node-fetch";
import os from "os";
import path from "path";
import { pipeline } from "stream/promises";

export const processVideo = task({
  id: "process-video",
  machine: {
    preset: "medium-1x", // Dedicated machine for video processing
  },
  run: async (payload: {
    videoUrl: string;
    teamId: string;
    docId: string;
    documentVersionId: string;
    fileSize: number;
  }) => {
    const { videoUrl, teamId, docId, documentVersionId, fileSize } = payload;

    try {
      const fileUrl = await getFile({
        data: videoUrl,
        type: "S3_PATH",
      });

      logger.info("Starting video optimization", { fileUrl });

      // Create temp directory for processing
      const tempDirectory = path.join(os.tmpdir(), `video_${Date.now()}`);
      await fs.mkdir(tempDirectory, { recursive: true });
      const inputPath = path.join(tempDirectory, "input.mp4");
      const outputPath = path.join(tempDirectory, "output.mp4");

      // Stream video to temporary file
      const response = await fetch(fileUrl);
      if (!response.body) {
        throw new Error("Failed to fetch video stream");
      }

      await pipeline(response.body, createWriteStream(inputPath));

      // Get video metadata using FFprobe
      const metadata = await new Promise<{
        width: number;
        height: number;
        fps: number;
        duration: number;
      }>((resolve, reject) => {
        ffmpeg.ffprobe(inputPath, (err, metadata) => {
          if (err) {
            logger.error("Probe error:", { error: err.message });
            reject(err);
            return;
          }
          
          const videoStream = metadata.streams.find(
            (s) => s.codec_type === "video",
          );
          if (!videoStream) {
            reject(new Error("No video stream found"));
            return;
          }

          const fps = (() => {
            const fpsStr = videoStream.r_frame_rate || videoStream.avg_frame_rate;
            const [num, den] = fpsStr?.split("/").map(Number) || [0, 1];
            return num / (den || 1);
          })();

          resolve({
            width: videoStream.width || 1920,
            height: videoStream.height || 1080,
            fps,
            duration: Math.round(metadata.format.duration || 0),
          });
        });
      });

      // Update document version with metadata
      await prisma.documentVersion.update({
        where: { id: documentVersionId },
        data: { length: metadata.duration },
      });

      // Skip optimization for large files (>500MB)
      if (fileSize > 500 * 1024 * 1024) {
        logger.info(
          `File size is ${fileSize / 1024 / 1024} MB, skipping optimization`,
        );
        await fs.rm(tempDirectory, { recursive: true });
        return {
          success: true,
          message: "File size is too large, skipping optimization",
        };
      }

      // Calculate encoding parameters
      const keyframeInterval = Math.round(metadata.fps * 2);
      const bitrate = "6000k";
      const maxBitrate = parseInt(bitrate.replace("k", "")) * 2;
      const scaleFilter = metadata.width > 1920 ? "-vf scale=1920:-2" : null;

      // Process video with FFmpeg
      await new Promise<void>((resolve, reject) => {
        const ffmpegCommand = ffmpeg(inputPath)
          .inputOptions(["-y"])
          .outputOptions([
            ...(scaleFilter ? [scaleFilter] : []),
            "-c:v libx264",
            "-profile:v high", 
            "-level:v 4.1",
            "-c:a aac",
            "-ar 48000",
            "-b:a 128k",
            `-b:v ${bitrate}`,
            `-maxrate ${maxBitrate}k`,
            `-bufsize ${maxBitrate}k`,
            "-preset medium",
            `-g ${keyframeInterval}`,
            `-keyint_min ${keyframeInterval}`,
            "-sc_threshold 0",
            "-movflags +faststart",
          ])
          .output(outputPath)
          .on("start", (cmd) => {
            logger.info("FFmpeg started:", { cmd });
          })
          .on("error", (err, stdout, stderr) => {
            logger.error("FFmpeg error:", {
              error: err.message,
              stdout,
              stderr,
            });
            reject(err);
          })
          .on("end", () => {
            logger.info("FFmpeg completed");
            resolve();
          });

        ffmpegCommand.run();
      });

      // Upload optimized video
      const fileStream = createReadStream(outputPath);
      const { type, data } = await streamFileServer({
        file: {
          name: "optimized.mp4",
          type: "video/mp4",
          stream: fileStream,
        },
        teamId,
        docId,
      });

      // Update document version with optimized file
      await prisma.documentVersion.update({
        where: { id: documentVersionId },
        data: { file: data },
      });

      // Cleanup temporary files
      await fs.rm(tempDirectory, { recursive: true });

      return {
        success: true,
        message: "Successfully optimized video",
      };
    } catch (error) {
      logger.error("Failed to optimize video:", {
        error: error instanceof Error ? error.message : String(error),
      });
      throw error;
    }
  },
});
```

### Document Conversion

```typescript
// lib/trigger/convert-files.ts
import { logger, retry, task } from "@trigger.dev/sdk/v3";
import { getFile } from "@/lib/files/get-file";
import { putFileServer } from "@/lib/files/put-file-server";
import { updateStatus } from "@/lib/utils/generate-trigger-status";
import { getExtensionFromContentType } from "@/lib/utils/get-content-type";
import { convertPdfToImageRoute } from "./pdf-to-image-route";
import prisma from "@/lib/prisma";

export type ConvertPayload = {
  documentId: string;
  documentVersionId: string;
  teamId: string;
};

export const convertFilesToPdfTask = task({
  id: "convert-files-to-pdf",
  retry: { maxAttempts: 3 },
  queue: { concurrencyLimit: 10 },
  run: async (payload: ConvertPayload) => {
    updateStatus({ progress: 0, text: "Initializing..." });

    const team = await prisma.team.findUnique({
      where: { id: payload.teamId },
    });

    if (!team) {
      logger.error("Team not found", { teamId: payload.teamId });
      return;
    }

    const document = await prisma.document.findUnique({
      where: { id: payload.documentId },
      select: {
        name: true,
        versions: {
          where: { id: payload.documentVersionId },
          select: {
            file: true,
            originalFile: true,
            contentType: true,
            storageType: true,
          },
        },
      },
    });

    if (
      !document ||
      !document.versions[0] ||
      !document.versions[0].originalFile ||
      !document.versions[0].contentType
    ) {
      updateStatus({ progress: 0, text: "Document not found" });
      logger.error("Document not found", payload);
      return;
    }

    updateStatus({ progress: 10, text: "Retrieving file..." });

    const fileUrl = await getFile({
      data: document.versions[0].originalFile,
      type: document.versions[0].storageType,
    });

    // Prepare form data for LibreOffice conversion
    const formData = new FormData();
    formData.append(
      "downloadFrom",
      JSON.stringify([{ url: fileUrl }]),
    );
    formData.append("quality", "75");

    updateStatus({ progress: 20, text: "Converting document..." });

    // Make the conversion request with retry logic
    const conversionResponse = await retry.fetch(
      `${process.env.NEXT_PRIVATE_CONVERSION_BASE_URL}/forms/libreoffice/convert`,
      {
        method: "POST",
        body: formData,
        headers: {
          Authorization: `Basic ${process.env.NEXT_PRIVATE_INTERNAL_AUTH_TOKEN}`,
        },
        retry: {
          byStatus: {
            "500-599": {
              strategy: "backoff",
              maxAttempts: 3,
              factor: 2,
              minTimeoutInMs: 1_000,
              maxTimeoutInMs: 30_000,
              randomize: false,
            },
          },
        },
      },
    );

    if (!conversionResponse.ok) {
      updateStatus({ progress: 0, text: "Conversion failed" });
      const body = await conversionResponse.json();
      throw new Error(
        `Conversion failed: ${body.message} ${conversionResponse.status}`,
      );
    }

    const conversionBuffer = Buffer.from(
      await conversionResponse.arrayBuffer(),
    );

    // Extract docId from original file path
    const match = document.versions[0].originalFile.match(/(doc_[^\\/]+)\\/)/);
    const docId = match ? match[1] : undefined;

    updateStatus({ progress: 30, text: "Saving converted file..." });

    // Save the converted PDF
    const { type: storageType, data } = await putFileServer({
      file: {
        name: `${document.name}.pdf`,
        type: "application/pdf",
        buffer: conversionBuffer,
      },
      teamId: payload.teamId,
      docId: docId,
    });

    if (!data || !storageType) {
      updateStatus({ progress: 0, text: "Failed to save converted file" });
      logger.error("Failed to save converted file", payload);
      return;
    }

    // Update document version with converted PDF
    const { versionNumber } = await prisma.documentVersion.update({
      where: { id: payload.documentVersionId },
      data: {
        file: data,
        type: "pdf",
        storageType: storageType,
      },
      select: { versionNumber: true },
    });

    updateStatus({ progress: 40, text: "Initiating document processing..." });

    // Trigger PDF to image conversion
    await convertPdfToImageRoute.trigger(
      {
        documentId: payload.documentId,
        documentVersionId: payload.documentVersionId,
        teamId: payload.teamId,
        versionNumber: versionNumber,
      },
      {
        idempotencyKey: `${payload.teamId}-${payload.documentVersionId}`,
        tags: [
          `team_${payload.teamId}`,
          `document_${payload.documentId}`,
          `version:${payload.documentVersionId}`,
        ],
      },
    );

    logger.info("Document converted", payload);
    return;
  },
});

// CAD file conversion to PDF
export const convertCadToPdfTask = task({
  id: "convert-cad-to-pdf",
  retry: { maxAttempts: 3 },
  queue: { concurrencyLimit: 2 }, // Lower concurrency for CAD files
  run: async (payload: ConvertPayload) => {
    const document = await prisma.document.findUnique({
      where: { id: payload.documentId },
      select: {
        name: true,
        versions: {
          where: { id: payload.documentVersionId },
          select: {
            originalFile: true,
            contentType: true,
            storageType: true,
          },
        },
      },
    });

    if (!document?.versions[0]) {
      logger.error("Document not found", payload);
      return;
    }

    const fileUrl = await getFile({
      data: document.versions[0].originalFile,
      type: document.versions[0].storageType,
    });

    // Create payload for CAD conversion service
    const tasksPayload = {
      tasks: {
        "import-file-v1": {
          operation: "import/url",
          url: fileUrl,
          filename: document.name,
        },
        "convert-file-v1": {
          operation: "convert",
          input: ["import-file-v1"],
          input_format: getExtensionFromContentType(
            document.versions[0].contentType,
          ),
          output_format: "pdf",
          engine: "cadconverter",
          all_layouts: true,
          auto_zoom: false,
        },
        "export-file-v1": {
          operation: "export/url",
          input: ["convert-file-v1"],
          inline: false,
          archive_multiple_files: false,
        },
      },
      redirect: true,
    };

    // Make conversion request to CAD conversion API
    const conversionResponse = await retry.fetch(
      `${process.env.NEXT_PRIVATE_CONVERT_API_URL}`,
      {
        method: "POST",
        body: JSON.stringify(tasksPayload),
        headers: {
          Authorization: `Bearer ${process.env.NEXT_PRIVATE_CONVERT_API_KEY}`,
          "Content-Type": "application/json",
        },
        retry: {
          byStatus: {
            "500-599": {
              strategy: "backoff",
              maxAttempts: 3,
              factor: 2,
              minTimeoutInMs: 1_000,
              maxTimeoutInMs: 30_000,
              randomize: false,
            },
          },
        },
      },
    );

    if (!conversionResponse.ok) {
      const body = await conversionResponse.json();
      throw new Error(
        `CAD conversion failed: ${body.message} ${conversionResponse.status}`,
      );
    }

    const conversionBuffer = Buffer.from(
      await conversionResponse.arrayBuffer(),
    );

    // Save converted PDF and trigger image conversion
    const match = document.versions[0].originalFile.match(/(doc_[^\\/]+)\\/)/);
    const docId = match ? match[1] : undefined;

    const { type: storageType, data } = await putFileServer({
      file: {
        name: `${document.name}.pdf`,
        type: "application/pdf",
        buffer: conversionBuffer,
      },
      teamId: payload.teamId,
      docId: docId,
    });

    if (!data || !storageType) {
      logger.error("Failed to save converted CAD file", payload);
      return;
    }

    await prisma.documentVersion.update({
      where: { id: payload.documentVersionId },
      data: {
        file: data,
        type: "pdf",
        storageType: storageType,
      },
    });

    // Trigger PDF to image conversion
    await convertPdfToImageRoute.trigger(
      {
        documentId: payload.documentId,
        documentVersionId: payload.documentVersionId,
        teamId: payload.teamId,
      },
      {
        idempotencyKey: `${payload.teamId}-${payload.documentVersionId}`,
        tags: [
          `team_${payload.teamId}`,
          `document_${payload.documentId}`,
          `version:${payload.documentVersionId}`,
        ],
      },
    );

    logger.info("CAD document converted", payload);
    return;
  },
});
```

### PDF to Image Conversion

```typescript
// lib/trigger/pdf-to-image-route.ts
import { logger, task } from "@trigger.dev/sdk/v3";
import { getFile } from "@/lib/files/get-file";
import { updateStatus } from "@/lib/utils/generate-trigger-status";
import prisma from "@/lib/prisma";

type ConvertPdfToImagePayload = {
  documentId: string;
  documentVersionId: string;
  teamId: string;
  versionNumber?: number;
};

export const convertPdfToImageRoute = task({
  id: "convert-pdf-to-image-route",
  run: async (payload: ConvertPdfToImagePayload) => {
    const { documentVersionId, teamId, documentId, versionNumber } = payload;

    updateStatus({ progress: 0, text: "Initializing..." });

    // Get document version
    const documentVersion = await prisma.documentVersion.findUnique({
      where: { id: documentVersionId },
      select: {
        file: true,
        storageType: true,
        numPages: true,
      },
    });

    if (!documentVersion) {
      logger.error("File not found", { payload });
      updateStatus({ progress: 0, text: "Document not found" });
      return;
    }

    logger.info("Document version", { documentVersion });
    updateStatus({ progress: 10, text: "Retrieving file..." });

    // Get signed URL from file
    const signedUrl = await getFile({
      type: documentVersion.storageType,
      data: documentVersion.file,
    });

    if (!signedUrl) {
      logger.error("Failed to get signed url", { payload });
      updateStatus({ progress: 0, text: "Failed to retrieve document" });
      return;
    }

    let numPages = documentVersion.numPages;

    // Get number of pages if not already defined
    if (!numPages || numPages === 1) {
      logger.info("Sending file to api/get-pages endpoint");

      const response = await fetch(
        `${process.env.NEXT_PUBLIC_BASE_URL}/api/mupdf/get-pages`,
        {
          method: "POST",
          body: JSON.stringify({ url: signedUrl }),
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${process.env.INTERNAL_API_KEY}`,
          },
        },
      );

      if (!response.ok) {
        logger.error("Failed to get number of pages", {
          signedUrl,
          response,
        });
        throw new Error("Failed to get number of pages");
      }

      const { numPages: numPagesResult } = (await response.json()) as {
        numPages: number;
      };

      if (numPagesResult < 1) {
        logger.error("Failed to get number of pages", { payload });
        updateStatus({ progress: 0, text: "Failed to get number of pages" });
        return;
      }

      numPages = numPagesResult;
    }

    updateStatus({ progress: 20, text: "Converting document..." });

    // Iterate through pages and convert to images
    let currentPage = 0;
    let conversionWithoutError = true;
    
    for (var i = 0; i < numPages; ++i) {
      if (!conversionWithoutError) {
        break;
      }

      currentPage = i + 1;
      logger.info(`Converting page ${currentPage}`, {
        currentPage,
        numPages,
      });

      try {
        // Convert each page to image
        const response = await fetch(
          `${process.env.NEXT_PUBLIC_BASE_URL}/api/mupdf/convert-page`,
          {
            method: "POST",
            body: JSON.stringify({
              documentVersionId: documentVersionId,
              pageNumber: currentPage,
              url: signedUrl,
              teamId: teamId,
            }),
            headers: {
              "Content-Type": "application/json",
              Authorization: `Bearer ${process.env.INTERNAL_API_KEY}`,
            },
          },
        );

        if (!response.ok) {
          throw new Error("Failed to convert page");
        }

        const { documentPageId } = (await response.json()) as {
          documentPageId: string;
        };

        logger.info(`Created document page for page ${currentPage}:`, {
          documentPageId,
          payload,
        });
      } catch (error: unknown) {
        conversionWithoutError = false;
        if (error instanceof Error) {
          logger.error("Failed to convert page", {
            error: error.message,
          });
        }
      }

      updateStatus({
        progress: (currentPage / numPages) * 100,
        text: `${currentPage} / ${numPages} pages processed`,
      });
    }

    if (!conversionWithoutError) {
      logger.error("Failed to process pages", { payload });
      updateStatus({
        progress: (currentPage / numPages) * 100,
        text: `Error processing page ${currentPage} of ${numPages}`,
      });
      return;
    }

    // Update document version to enable pages
    await prisma.documentVersion.update({
      where: { id: documentVersionId },
      data: {
        numPages: numPages,
        hasPages: true,
        isPrimary: true,
      },
    });

    logger.info("Enabling pages");
    updateStatus({ progress: 90, text: "Enabling pages..." });

    if (versionNumber) {
      // Set all other versions to not primary
      await prisma.documentVersion.updateMany({
        where: {
          documentId: documentId,
          versionNumber: { not: versionNumber },
        },
        data: { isPrimary: false },
      });
    }

    logger.info("Revalidating link");
    updateStatus({ progress: 95, text: "Revalidating link..." });

    // Revalidate document links
    await fetch(
      `${process.env.NEXTAUTH_URL}/api/revalidate?secret=${process.env.REVALIDATE_TOKEN}&documentId=${documentId}`,
    );

    updateStatus({ progress: 100, text: "Processing complete" });

    logger.info("Processing complete");
    return {
      success: true,
      message: "Successfully converted PDF to images",
      totalPages: numPages,
    };
  },
});
```

---

## Cleanup and Maintenance Tasks

### Scheduled Cleanup Operations

```typescript
// lib/trigger/cleanup-expired-exports.ts
import { logger, schedules } from "@trigger.dev/sdk/v3";
import { del } from "@vercel/blob";
import { jobStore } from "@/lib/redis-job-store";

export const cleanupExpiredExports = schedules.task({
  id: "cleanup-expired-exports",
  // Run daily at 2 AM UTC
  cron: "0 2 * * *",
  run: async (payload) => {
    logger.info("Starting cleanup of expired export blobs", {
      timestamp: payload.timestamp,
    });

    try {
      // Get all blob URLs that are due for cleanup
      const blobsToCleanup = await jobStore.getBlobsForCleanup();

      if (blobsToCleanup.length === 0) {
        logger.info("No blobs due for cleanup");
        return { deletedCount: 0 };
      }

      logger.info(`Found ${blobsToCleanup.length} blobs to delete`);

      // Delete blobs from Vercel Blob with error handling
      const deletionResults = await Promise.allSettled(
        blobsToCleanup.map(async (blob) => {
          try {
            await del(blob.blobUrl);

            // Remove from cleanup queue after successful deletion
            await jobStore.removeBlobFromCleanupQueue(blob.blobUrl, blob.jobId);

            logger.info("Successfully deleted blob", {
              blobUrl: blob.blobUrl,
              jobId: blob.jobId,
            });

            return { blob, success: true };
          } catch (error) {
            logger.error("Failed to delete blob", {
              blobUrl: blob.blobUrl,
              jobId: blob.jobId,
              error: error instanceof Error ? error.message : String(error),
            });
            return { blob, success: false, error };
          }
        }),
      );

      const successCount = deletionResults.filter(
        (result) => result.status === "fulfilled" && result.value.success,
      ).length;

      const failureCount = deletionResults.length - successCount;

      logger.info("Cleanup completed", {
        totalBlobs: blobsToCleanup.length,
        successCount,
        failureCount,
      });

      return {
        deletedCount: successCount,
        failureCount,
        totalProcessed: blobsToCleanup.length,
      };
    } catch (error) {
      logger.error("Cleanup task failed", {
        error: error instanceof Error ? error.message : String(error),
      });
      throw error;
    }
  },
});
```

---

## Best Practices Summary

### 1. **Task Design Principles**
- **Single Responsibility**: Each task should have one clear purpose
- **Idempotency**: Tasks should be safe to retry and produce same results
- **Resource Management**: Always clean up temporary files and connections
- **Error Boundaries**: Handle errors gracefully with proper logging

### 2. **Performance Optimization**
- **Queue Management**: Use appropriate concurrency limits for different task types
- **Resource Allocation**: Choose the right machine presets for CPU/memory intensive tasks
- **Batching**: Process multiple items together when possible
- **Streaming**: Use streams for large file operations to reduce memory usage

### 3. **Monitoring and Observability**
- **Structured Logging**: Include relevant context in all log messages
- **Progress Tracking**: Update status for long-running operations
- **Error Tracking**: Log errors with full context for debugging
- **Metrics**: Track task duration, success rates, and resource usage

### 4. **Production Deployment**
- **Environment Configuration**: Use proper environment variables for different stages
- **Secret Management**: Store sensitive data securely in environment variables
- **Health Checks**: Implement monitoring for task execution health
- **Graceful Degradation**: Handle external service failures appropriately

### 5. **Testing Strategy**
- **Unit Tests**: Test individual task functions with mocked dependencies
- **Integration Tests**: Test tasks with real external services in staging
- **Load Testing**: Verify performance under expected production loads
- **Error Simulation**: Test retry logic and error handling scenarios

---

## Conclusion

This comprehensive guide demonstrates how to implement production-ready Trigger.dev v3 integration for background processing, cron jobs, and task orchestration. The patterns shown here from Papermark's implementation provide:

- **Scalable Architecture**: Queue-based processing with proper concurrency controls
- **Robust Error Handling**: Comprehensive retry strategies and error recovery
- **Flexible Scheduling**: Cron-based and event-driven task execution  
- **Resource Optimization**: Efficient file processing and memory management
- **Production Readiness**: Monitoring, logging, and deployment strategies

Use these patterns as a foundation for implementing similar background processing systems in your own projects, adapting the specific business logic while maintaining the core architectural principles demonstrated here.
