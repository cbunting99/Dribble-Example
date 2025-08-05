# Dribbble Replica Build Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Tech Stack](#tech-stack)
3. [Project Setup](#project-setup)
4. [Database Design](#database-design)
5. [Backend Development](#backend-development)
6. [Frontend Development](#frontend-development)
7. [Core Features Implementation](#core-features-implementation)
8. [Advanced Features](#advanced-features)
9. [Testing](#testing)
10. [Deployment](#deployment)
11. [Performance Optimization](#performance-optimization)

## Project Overview

This guide will walk you through building a complete Dribbble replica with the following features:

### Core Features
- User authentication and profiles
- Shot (design) upload and management
- Like and save functionality
- Comments system
- Search and filtering
- Following/followers system
- Collections
- Responsive design

### Advanced Features
- Real-time notifications
- Advanced search with filters
- Image processing and optimization
- Email notifications
- Analytics dashboard
- Pro membership system
- Job board (optional)

## Tech Stack

### Frontend
- **Framework**: React 18 with TypeScript
- **Styling**: Tailwind CSS
- **State Management**: Zustand or Redux Toolkit
- **Routing**: React Router v6
- **Forms**: React Hook Form with Zod validation
- **Image Handling**: React Dropzone + Image optimization
- **Icons**: Lucide React
- **Animations**: Framer Motion

### Backend
- **Runtime**: Node.js with Express.js
- **Language**: TypeScript
- **Database**: PostgreSQL with Prisma ORM
- **Authentication**: JWT with refresh tokens
- **File Storage**: AWS S3 or Cloudinary
- **Image Processing**: Sharp
- **Email**: SendGrid or Nodemailer
- **Caching**: Redis
- **WebSockets**: Socket.io (for real-time features)

### DevOps & Tools
- **Build Tool**: Vite
- **Testing**: Jest + React Testing Library
- **API Testing**: Supertest
- **Code Quality**: ESLint + Prettier
- **Git Hooks**: Husky
- **Deployment**: Docker + AWS/Vercel/Railway

## Project Setup

### 1. Initialize the Project

```bash
# Create project directory
mkdir dribbble-clone
cd dribbble-clone

# Initialize backend
mkdir backend
cd backend
npm init -y
npm install express cors helmet morgan compression dotenv bcryptjs jsonwebtoken
npm install -D typescript @types/node @types/express @types/cors @types/bcryptjs @types/jsonwebtoken ts-node nodemon

# Initialize frontend
cd ..
npx create-react-app frontend --template typescript
cd frontend
npm install @tanstack/react-query axios react-router-dom react-hook-form @hookform/resolvers zod
npm install -D tailwindcss postcss autoprefixer
```

### 2. Setup TypeScript Configuration

**Backend `tsconfig.json`:**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 3. Setup Environment Variables

**Backend `.env`:**
```env
NODE_ENV=development
PORT=5000
DATABASE_URL="postgresql://username:password@localhost:5432/dribbble_clone"
JWT_SECRET=your-super-secret-jwt-key
JWT_REFRESH_SECRET=your-refresh-secret-key
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret
REDIS_URL=redis://localhost:6379
EMAIL_SERVICE_API_KEY=your-email-service-key
```

## Database Design

### Prisma Schema (`schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id          String   @id @default(cuid())
  email       String   @unique
  username    String   @unique
  name        String
  bio         String?
  avatar      String?
  website     String?
  location    String?
  isVerified  Boolean  @default(false)
  isPro       Boolean  @default(false)
  password    String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  // Relations
  shots       Shot[]
  likes       Like[]
  comments    Comment[]
  collections Collection[]
  followers   Follow[] @relation("UserFollowers")
  following   Follow[] @relation("UserFollowing")
  saves       Save[]

  @@map("users")
}

model Shot {
  id          String   @id @default(cuid())
  title       String
  description String?
  imageUrl    String
  tags        String[]
  color       String?
  views       Int      @default(0)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  // Relations
  userId      String
  user        User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  likes       Like[]
  comments    Comment[]
  saves       Save[]
  collections CollectionShot[]

  @@map("shots")
}

model Like {
  id     String @id @default(cuid())
  userId String
  shotId String
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  shot   Shot   @relation(fields: [shotId], references: [id], onDelete: Cascade)

  @@unique([userId, shotId])
  @@map("likes")
}

model Comment {
  id        String   @id @default(cuid())
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  userId String
  shotId String
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  shot   Shot   @relation(fields: [shotId], references: [id], onDelete: Cascade)

  @@map("comments")
}

model Follow {
  id          String @id @default(cuid())
  followerId  String
  followingId String

  follower  User @relation("UserFollowers", fields: [followerId], references: [id], onDelete: Cascade)
  following User @relation("UserFollowing", fields: [followingId], references: [id], onDelete: Cascade)

  @@unique([followerId, followingId])
  @@map("follows")
}

model Collection {
  id          String   @id @default(cuid())
  name        String
  description String?
  isPrivate   Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  userId String
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  shots  CollectionShot[]

  @@map("collections")
}

model CollectionShot {
  id           String @id @default(cuid())
  collectionId String
  shotId       String

  collection Collection @relation(fields: [collectionId], references: [id], onDelete: Cascade)
  shot       Shot       @relation(fields: [shotId], references: [id], onDelete: Cascade)

  @@unique([collectionId, shotId])
  @@map("collection_shots")
}

model Save {
  id     String @id @default(cuid())
  userId String
  shotId String
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  shot   Shot   @relation(fields: [shotId], references: [id], onDelete: Cascade)

  @@unique([userId, shotId])
  @@map("saves")
}
```

## Backend Development

### 1. Project Structure

```
backend/
├── src/
│   ├── controllers/
│   ├── middleware/
│   ├── routes/
│   ├── services/
│   ├── utils/
│   ├── types/
│   └── app.ts
├── prisma/
│   └── schema.prisma
└── package.json
```

### 2. Express App Setup (`src/app.ts`)

```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';
import { PrismaClient } from '@prisma/client';

import authRoutes from './routes/auth';
import userRoutes from './routes/users';
import shotRoutes from './routes/shots';
import { errorHandler } from './middleware/errorHandler';
import { rateLimiter } from './middleware/rateLimiter';

const app = express();
export const prisma = new PrismaClient();

// Middleware
app.use(helmet());
app.use(cors());
app.use(compression());
app.use(morgan('combined'));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));
app.use(rateLimiter);

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/shots', shotRoutes);

// Error handling
app.use(errorHandler);

export default app;
```

### 3. Authentication Controller

```typescript
// src/controllers/authController.ts
import { Request, Response } from 'express';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { prisma } from '../app';
import { generateTokens, verifyRefreshToken } from '../utils/tokenUtils';

export const register = async (req: Request, res: Response) => {
  try {
    const { email, username, name, password } = req.body;

    // Check if user exists
    const existingUser = await prisma.user.findFirst({
      where: {
        OR: [{ email }, { username }]
      }
    });

    if (existingUser) {
      return res.status(400).json({ 
        error: 'User with this email or username already exists' 
      });
    }

    // Hash password
    const hashedPassword = await bcrypt.hash(password, 12);

    // Create user
    const user = await prisma.user.create({
      data: {
        email,
        username,
        name,
        password: hashedPassword
      },
      select: {
        id: true,
        email: true,
        username: true,
        name: true,
        avatar: true,
        createdAt: true
      }
    });

    // Generate tokens
    const { accessToken, refreshToken } = generateTokens(user.id);

    res.status(201).json({
      user,
      accessToken,
      refreshToken
    });
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
};

export const login = async (req: Request, res: Response) => {
  try {
    const { email, password } = req.body;

    // Find user
    const user = await prisma.user.findUnique({
      where: { email }
    });

    if (!user || !await bcrypt.compare(password, user.password)) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Generate tokens
    const { accessToken, refreshToken } = generateTokens(user.id);

    res.json({
      user: {
        id: user.id,
        email: user.email,
        username: user.username,
        name: user.name,
        avatar: user.avatar
      },
      accessToken,
      refreshToken
    });
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
};
```

### 4. Shot Controller

```typescript
// src/controllers/shotController.ts
import { Request, Response } from 'express';
import { prisma } from '../app';
import { uploadImage } from '../services/imageService';

export const createShot = async (req: Request, res: Response) => {
  try {
    const { title, description, tags, color } = req.body;
    const userId = req.user!.id;
    const imageFile = req.file;

    if (!imageFile) {
      return res.status(400).json({ error: 'Image is required' });
    }

    // Upload image
    const imageUrl = await uploadImage(imageFile);

    // Create shot
    const shot = await prisma.shot.create({
      data: {
        title,
        description,
        imageUrl,
        tags: tags ? tags.split(',').map((tag: string) => tag.trim()) : [],
        color,
        userId
      },
      include: {
        user: {
          select: {
            id: true,
            username: true,
            name: true,
            avatar: true,
            isPro: true
          }
        },
        _count: {
          select: {
            likes: true,
            comments: true,
            saves: true
          }
        }
      }
    });

    res.status(201).json(shot);
  } catch (error) {
    res.status(500).json({ error: 'Failed to create shot' });
  }
};

export const getShots = async (req: Request, res: Response) => {
  try {
    const { page = 1, limit = 20, search, tags, color, sort = 'recent' } = req.query;
    const skip = (Number(page) - 1) * Number(limit);

    const where: any = {};

    if (search) {
      where.OR = [
        { title: { contains: search as string, mode: 'insensitive' } },
        { description: { contains: search as string, mode: 'insensitive' } }
      ];
    }

    if (tags) {
      where.tags = {
        hasSome: (tags as string).split(',').map(tag => tag.trim())
      };
    }

    if (color) {
      where.color = color;
    }

    const orderBy: any = {};
    switch (sort) {
      case 'popular':
        orderBy.likes = { _count: 'desc' };
        break;
      case 'views':
        orderBy.views = 'desc';
        break;
      default:
        orderBy.createdAt = 'desc';
    }

    const shots = await prisma.shot.findMany({
      where,
      skip,
      take: Number(limit),
      orderBy,
      include: {
        user: {
          select: {
            id: true,
            username: true,
            name: true,
            avatar: true,
            isPro: true
          }
        },
        _count: {
          select: {
            likes: true,
            comments: true,
            saves: true
          }
        }
      }
    });

    const total = await prisma.shot.count({ where });

    res.json({
      shots,
      pagination: {
        page: Number(page),
        limit: Number(limit),
        total,
        pages: Math.ceil(total / Number(limit))
      }
    });
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch shots' });
  }
};

export const likeShot = async (req: Request, res: Response) => {
  try {
    const { shotId } = req.params;
    const userId = req.user!.id;

    const existingLike = await prisma.like.findUnique({
      where: {
        userId_shotId: {
          userId,
          shotId
        }
      }
    });

    if (existingLike) {
      await prisma.like.delete({
        where: { id: existingLike.id }
      });
      res.json({ liked: false });
    } else {
      await prisma.like.create({
        data: { userId, shotId }
      });
      res.json({ liked: true });
    }
  } catch (error) {
    res.status(500).json({ error: 'Failed to toggle like' });
  }
};
```

## Frontend Development

### 1. Project Structure

```
frontend/
├── src/
│   ├── components/
│   │   ├── common/
│   │   ├── auth/
│   │   ├── shots/
│   │   └── user/
│   ├── pages/
│   ├── hooks/
│   ├── services/
│   ├── store/
│   ├── types/
│   ├── utils/
│   └── App.tsx
└── public/
```

### 2. Main App Component

```tsx
// src/App.tsx
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Toaster } from 'react-hot-toast';

import Header from './components/common/Header';
import Home from './pages/Home';
import Login from './pages/Login';
import Register from './pages/Register';
import Profile from './pages/Profile';
import ShotDetail from './pages/ShotDetail';
import Upload from './pages/Upload';
import { AuthProvider } from './contexts/AuthContext';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <Router>
          <div className="min-h-screen bg-gray-50">
            <Header />
            <Routes>
              <Route path="/" element={<Home />} />
              <Route path="/login" element={<Login />} />
              <Route path="/register" element={<Register />} />
              <Route path="/upload" element={<Upload />} />
              <Route path="/shots/:id" element={<ShotDetail />} />
              <Route path="/:username" element={<Profile />} />
            </Routes>
            <Toaster position="top-right" />
          </div>
        </Router>
      </AuthProvider>
    </QueryClientProvider>
  );
}

export default App;
```

### 3. Shot Grid Component

```tsx
// src/components/shots/ShotGrid.tsx
import React from 'react';
import { Heart, MessageCircle, Bookmark } from 'lucide-react';
import { Shot } from '../types';

interface ShotGridProps {
  shots: Shot[];
  onLike: (shotId: string) => void;
  onSave: (shotId: string) => void;
}

const ShotGrid: React.FC<ShotGridProps> = ({ shots, onLike, onSave }) => {
  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
      {shots.map((shot) => (
        <div
          key={shot.id}
          className="group bg-white rounded-lg overflow-hidden shadow-sm hover:shadow-xl transition-all duration-300"
        >
          <div className="relative overflow-hidden">
            <img
              src={shot.imageUrl}
              alt={shot.title}
              className="w-full h-64 object-cover group-hover:scale-105 transition-transform duration-300"
            />
            <div className="absolute inset-0 bg-black bg-opacity-0 group-hover:bg-opacity-30 transition-all duration-300 flex items-center justify-center">
              <div className="opacity-0 group-hover:opacity-100 transition-opacity duration-300 flex space-x-4">
                <button
                  onClick={() => onLike(shot.id)}
                  className="bg-white rounded-full p-3 hover:bg-gray-100 transition-colors"
                >
                  <Heart className="w-5 h-5 text-gray-700" />
                </button>
                <button
                  onClick={() => onSave(shot.id)}
                  className="bg-white rounded-full p-3 hover:bg-gray-100 transition-colors"
                >
                  <Bookmark className="w-5 h-5 text-gray-700" />
                </button>
              </div>
            </div>
          </div>
          
          <div className="p-4">
            <div className="flex items-center justify-between mb-2">
              <h3 className="font-semibold text-gray-900 truncate">
                {shot.title}
              </h3>
              <div className="flex items-center space-x-1 text-sm text-gray-500">
                <Heart className="w-4 h-4" />
                <span>{shot._count.likes}</span>
              </div>
            </div>
            
            <div className="flex items-center justify-between">
              <div className="flex items-center space-x-2">
                <img
                  src={shot.user.avatar || '/default-avatar.png'}
                  alt={shot.user.name}
                  className="w-6 h-6 rounded-full"
                />
                <span className="text-sm text-gray-600">{shot.user.name}</span>
                {shot.user.isPro && (
                  <span className="bg-pink-100 text-pink-800 text-xs px-2 py-1 rounded-full">
                    PRO
                  </span>
                )}
              </div>
              
              <div className="flex items-center space-x-3 text-sm text-gray-500">
                <div className="flex items-center space-x-1">
                  <MessageCircle className="w-4 h-4" />
                  <span>{shot._count.comments}</span>
                </div>
                <div className="flex items-center space-x-1">
                  <Bookmark className="w-4 h-4" />
                  <span>{shot._count.saves}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      ))}
    </div>
  );
};

export default ShotGrid;
```

### 4. Upload Form Component

```tsx
// src/components/shots/UploadForm.tsx
import React, { useState, useCallback } from 'react';
import { useDropzone } from 'react-dropzone';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Upload, X } from 'lucide-react';

const uploadSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100),
  description: z.string().max(500).optional(),
  tags: z.string().optional(),
  color: z.string().optional()
});

type UploadFormData = z.infer<typeof uploadSchema>;

const UploadForm: React.FC = () => {
  const [imageFile, setImageFile] = useState<File | null>(null);
  const [imagePreview, setImagePreview] = useState<string | null>(null);

  const { register, handleSubmit, formState: { errors } } = useForm<UploadFormData>({
    resolver: zodResolver(uploadSchema)
  });

  const onDrop = useCallback((acceptedFiles: File[]) => {
    const file = acceptedFiles[0];
    if (file) {
      setImageFile(file);
      const reader = new FileReader();
      reader.onload = () => setImagePreview(reader.result as string);
      reader.readAsDataURL(file);
    }
  }, []);

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: {
      'image/*': ['.png', '.jpg', '.jpeg', '.gif', '.webp']
    },
    maxFiles: 1,
    maxSize: 10 * 1024 * 1024 // 10MB
  });

  const removeImage = () => {
    setImageFile(null);
    setImagePreview(null);
  };

  const onSubmit = async (data: UploadFormData) => {
    if (!imageFile) return;

    const formData = new FormData();
    formData.append('image', imageFile);
    formData.append('title', data.title);
    if (data.description) formData.append('description', data.description);
    if (data.tags) formData.append('tags', data.tags);
    if (data.color) formData.append('color', data.color);

    try {
      // API call to upload shot
      console.log('Uploading shot...', data);
    } catch (error) {
      console.error('Upload failed:', error);
    }
  };

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-3xl font-bold text-gray-900 mb-8">Upload a Shot</h1>
      
      <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
        {/* Image Upload */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Image
          </label>
          {!imagePreview ? (
            <div
              {...getRootProps()}
              className={`border-2 border-dashed rounded-lg p-8 text-center cursor-pointer transition-colors ${
                isDragActive
                  ? 'border-pink-500 bg-pink-50'
                  : 'border-gray-300 hover:border-gray-400'
              }`}
            >
              <input {...getInputProps()} />
              <Upload className="mx-auto h-12 w-12 text-gray-400 mb-4" />
              <p className="text-gray-600">
                {isDragActive
                  ? 'Drop the image here...'
                  : 'Drag & drop an image, or click to select'}
              </p>
              <p className="text-sm text-gray-500 mt-2">
                PNG, JPG, GIF up to 10MB
              </p>
            </div>
          ) : (
            <div className="relative">
              <img
                src={imagePreview}
                alt="Preview"
                className="w-full h-64 object-cover rounded-lg"
              />
              <button
                type="button"
                onClick={removeImage}
                className="absolute top-2 right-2 bg-red-500 text-white rounded-full p-1 hover:bg-red-600"
              >
                <X className="w-4 h-4" />
              </button>
            </div>
          )}
        </div>

        {/* Title */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Title *
          </label>
          <input
            {...register('title')}
            type="text"
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-pink-500"
            placeholder="What would you like to call this shot?"
          />
          {errors.title && (
            <p className="text-red-500 text-sm mt-1">{errors.title.message}</p>
          )}
        </div>

        {/* Description */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Description
          </label>
          <textarea
            {...register('description')}
            rows={4}
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-pink-500"
            placeholder="Describe your shot..."
          />
          {errors.description && (
            <p className="text-red-500 text-sm mt-1">{errors.description.message}</p>
          )}
        </div>

        {/* Tags */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Tags
          </label>
          <input
            {...register('tags')}
            type="text"
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-pink-500"
            placeholder="web design, mobile, ui/ux (comma separated)"
          />
        </div>

        {/* Color */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Primary Color
          </label>
          <input
            {...register('color')}
            type="color"
            className="w-20 h-10 border border-gray-300 rounded-md"
          />
        </div>

        <button
          type="submit"
          disabled={!imageFile}
          className="w-full bg-pink-600 text-white py-3 px-4 rounded-md hover:bg-pink-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors"
        >
          Publish Shot
        </button>
      </form>
    </div>
  );
};

export default UploadForm;
```

## Core Features Implementation

### 1. Search and Filtering

```typescript
// src/hooks/useShots.ts
import { useQuery } from '@tanstack/react-query';
import { shotService } from '../services/shotService';

interface UseShotsParams {
  page?: number;
  search?: string;
  tags?: string[];
  color?: string;
  sort?: 'recent' | 'popular' | 'views';
}

export const useShots = (params: UseShotsParams) => {
  return useQuery({
    queryKey: ['shots', params],
    queryFn: () => shotService.getShots(params),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};
```

### 2. Real-time Features with Socket.io

```typescript
// Backend: src/services/socketService.ts
import { Server } from 'socket.io';
import jwt from 'jsonwebtoken';

export const initializeSocket = (server: any) => {
  const io = new Server(server, {
    cors: {
      origin: process.env.FRONTEND_URL,
      methods: ['GET', 'POST']
    }
  });

  io.use((socket, next) => {
    const token = socket.handshake.auth.token;
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET!) as any;
      socket.userId = decoded.userId;
      next();
    } catch (err) {
      next(new Error('Authentication error'));
    }
  });

  io.on('connection', (socket) => {
    console.log('User connected:', socket.userId);

    socket.on('join-shot', (shotId) => {
      socket.join(`shot-${shotId}`);
    });

    socket.on('leave-shot', (shotId) => {
      socket.leave(`shot-${shotId}`);
    });

    socket.on('disconnect', () => {
      console.log('User disconnected:', socket.userId);
    });
  });

  return io;
};

// Emit new comment to shot viewers
export const emitNewComment = (io: any, shotId: string, comment: any) => {
  io.to(`shot-${shotId}`).emit('new-comment', comment);
};

// Emit like notification
export const emitLikeNotification = (io: any, userId: string, notification: any) => {
  io.to(`user-${userId}`).emit('notification', notification);
};
```

### 3. Advanced Image Processing

```typescript
// Backend: src/services/imageService.ts
import sharp from 'sharp';
import { v2 as cloudinary } from 'cloudinary';

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

export const processAndUploadImage = async (imageBuffer: Buffer): Promise<string> => {
  try {
    // Process image with Sharp
    const processedImage = await sharp(imageBuffer)
      .resize(800, 600, { 
        fit: 'inside',
        withoutEnlargement: true 
      })
      .jpeg({ quality: 85 })
      .toBuffer();

    // Generate thumbnail
    const thumbnail = await sharp(imageBuffer)
      .resize(400, 300, { 
        fit: 'cover' 
      })
      .jpeg({ quality: 80 })
      .toBuffer();

    // Upload to Cloudinary
    const [originalUpload, thumbnailUpload] = await Promise.all([
      new Promise<any>((resolve, reject) => {
        cloudinary.uploader.upload_stream(
          { 
            folder: 'dribbble-clone/shots',
            format: 'jpg',
            quality: 'auto:good'
          },
          (error, result) => {
            if (error) reject(error);
            else resolve(result);
          }
        ).end(processedImage);
      }),
      new Promise<any>((resolve, reject) => {
        cloudinary.uploader.upload_stream(
          { 
            folder: 'dribbble-clone/thumbnails',
            format: 'jpg',
            quality: 'auto:low'
          },
          (error, result) => {
            if (error) reject(error);
            else resolve(result);
          }
        ).end(thumbnail);
      })
    ]);

    return originalUpload.secure_url;
  } catch (error) {
    throw new Error('Image processing failed');
  }
};

// Extract dominant color from image
export const extractDominantColor = async (imageBuffer: Buffer): Promise<string> => {
  try {
    const { dominant } = await sharp(imageBuffer).stats();
    const { r, g, b } = dominant;
    return `#${r.toString(16).padStart(2, '0')}${g.toString(16).padStart(2, '0')}${b.toString(16).padStart(2, '0')}`;
  } catch (error) {
    return '#000000'; // Default color
  }
};
```

### 4. Advanced Search with Elasticsearch (Optional)

```typescript
// Backend: src/services/searchService.ts
import { Client } from '@elastic/elasticsearch';

const client = new Client({
  node: process.env.ELASTICSEARCH_URL || 'http://localhost:9200'
});

export const indexShot = async (shot: any) => {
  try {
    await client.index({
      index: 'shots',
      id: shot.id,
      body: {
        title: shot.title,
        description: shot.description,
        tags: shot.tags,
        color: shot.color,
        userId: shot.userId,
        username: shot.user.username,
        userTags: shot.user.tags,
        createdAt: shot.createdAt,
        likesCount: shot._count.likes,
        viewsCount: shot.views
      }
    });
  } catch (error) {
    console.error('Failed to index shot:', error);
  }
};

export const searchShots = async (query: string, filters: any = {}) => {
  try {
    const searchQuery: any = {
      index: 'shots',
      body: {
        query: {
          bool: {
            must: [],
            filter: []
          }
        },
        sort: [
          { createdAt: { order: 'desc' } }
        ],
        from: filters.from || 0,
        size: filters.size || 20
      }
    };

    // Text search
    if (query) {
      searchQuery.body.query.bool.must.push({
        multi_match: {
          query,
          fields: ['title^2', 'description', 'tags^1.5', 'username'],
          fuzziness: 'AUTO'
        }
      });
    } else {
      searchQuery.body.query.bool.must.push({
        match_all: {}
      });
    }

    // Filters
    if (filters.tags && filters.tags.length > 0) {
      searchQuery.body.query.bool.filter.push({
        terms: { tags: filters.tags }
      });
    }

    if (filters.color) {
      searchQuery.body.query.bool.filter.push({
        term: { color: filters.color }
      });
    }

    // Sorting
    if (filters.sort === 'popular') {
      searchQuery.body.sort = [{ likesCount: { order: 'desc' } }];
    } else if (filters.sort === 'views') {
      searchQuery.body.sort = [{ viewsCount: { order: 'desc' } }];
    }

    const response = await client.search(searchQuery);
    return response.body.hits;
  } catch (error) {
    console.error('Search failed:', error);
    throw error;
  }
};
```

## Advanced Features

### 1. Collections System

```tsx
// Frontend: src/components/collections/CollectionModal.tsx
import React, { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Plus, Check } from 'lucide-react';
import { collectionService } from '../../services/collectionService';

interface CollectionModalProps {
  shotId: string;
  isOpen: boolean;
  onClose: () => void;
}

const CollectionModal: React.FC<CollectionModalProps> = ({ shotId, isOpen, onClose }) => {
  const [newCollectionName, setNewCollectionName] = useState('');
  const [showCreateForm, setShowCreateForm] = useState(false);
  const queryClient = useQueryClient();

  const { data: collections } = useQuery({
    queryKey: ['collections'],
    queryFn: collectionService.getUserCollections
  });

  const createCollectionMutation = useMutation({
    mutationFn: collectionService.createCollection,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['collections'] });
      setNewCollectionName('');
      setShowCreateForm(false);
    }
  });

  const addToCollectionMutation = useMutation({
    mutationFn: ({ collectionId }: { collectionId: string }) =>
      collectionService.addShotToCollection(collectionId, shotId),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['collections'] });
    }
  });

  const handleCreateCollection = () => {
    if (newCollectionName.trim()) {
      createCollectionMutation.mutate({
        name: newCollectionName,
        description: '',
        isPrivate: false
      });
    }
  };

  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg p-6 w-full max-w-md">
        <h3 className="text-lg font-semibold mb-4">Save to collection</h3>
        
        <div className="space-y-2 max-h-60 overflow-y-auto">
          {collections?.map((collection: any) => (
            <div
              key={collection.id}
              onClick={() => addToCollectionMutation.mutate({ collectionId: collection.id })}
              className="flex items-center justify-between p-3 border rounded-lg cursor-pointer hover:bg-gray-50"
            >
              <div>
                <p className="font-medium">{collection.name}</p>
                <p className="text-sm text-gray-500">{collection._count.shots} shots</p>
              </div>
              {collection.hasShotId && (
                <Check className="w-5 h-5 text-green-500" />
              )}
            </div>
          ))}
        </div>

        {showCreateForm ? (
          <div className="mt-4 p-3 border rounded-lg">
            <input
              type="text"
              value={newCollectionName}
              onChange={(e) => setNewCollectionName(e.target.value)}
              placeholder="Collection name"
              className="w-full px-3 py-2 border rounded-md mb-3"
              autoFocus
            />
            <div className="flex space-x-2">
              <button
                onClick={handleCreateCollection}
                disabled={!newCollectionName.trim() || createCollectionMutation.isPending}
                className="bg-pink-600 text-white px-4 py-2 rounded-md hover:bg-pink-700 disabled:bg-gray-300"
              >
                Create
              </button>
              <button
                onClick={() => setShowCreateForm(false)}
                className="border border-gray-300 px-4 py-2 rounded-md hover:bg-gray-50"
              >
                Cancel
              </button>
            </div>
          </div>
        ) : (
          <button
            onClick={() => setShowCreateForm(true)}
            className="w-full mt-4 flex items-center justify-center space-x-2 p-3 border-2 border-dashed border-gray-300 rounded-lg hover:border-gray-400"
          >
            <Plus className="w-5 h-5" />
            <span>Create new collection</span>
          </button>
        )}

        <div className="flex justify-end mt-6">
          <button
            onClick={onClose}
            className="px-4 py-2 text-gray-600 hover:text-gray-800"
          >
            Done
          </button>
        </div>
      </div>
    </div>
  );
};

export default CollectionModal;
```

### 2. Notification System

```typescript
// Backend: src/controllers/notificationController.ts
import { Request, Response } from 'express';
import { prisma } from '../app';
import { emitNotification } from '../services/socketService';

export const createNotification = async (
  userId: string,
  type: string,
  data: any,
  io?: any
) => {
  const notification = await prisma.notification.create({
    data: {
      userId,
      type,
      data,
      read: false
    },
    include: {
      user: {
        select: {
          id: true,
          username: true,
          name: true,
          avatar: true
        }
      }
    }
  });

  // Emit real-time notification
  if (io) {
    emitNotification(io, userId, notification);
  }

  return notification;
};

export const getNotifications = async (req: Request, res: Response) => {
  try {
    const userId = req.user!.id;
    const { page = 1, limit = 20 } = req.query;
    const skip = (Number(page) - 1) * Number(limit);

    const notifications = await prisma.notification.findMany({
      where: { userId },
      skip,
      take: Number(limit),
      orderBy: { createdAt: 'desc' },
      include: {
        user: {
          select: {
            id: true,
            username: true,
            name: true,
            avatar: true
          }
        }
      }
    });

    const unreadCount = await prisma.notification.count({
      where: { userId, read: false }
    });

    res.json({
      notifications,
      unreadCount,
      pagination: {
        page: Number(page),
        limit: Number(limit)
      }
    });
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch notifications' });
  }
};

export const markAsRead = async (req: Request, res: Response) => {
  try {
    const { notificationId } = req.params;
    const userId = req.user!.id;

    await prisma.notification.updateMany({
      where: {
        id: notificationId,
        userId
      },
      data: { read: true }
    });

    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: 'Failed to mark notification as read' });
  }
};
```

### 3. Analytics Dashboard

```tsx
// Frontend: src/components/analytics/AnalyticsDashboard.tsx
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, BarChart, Bar } from 'recharts';
import { Eye, Heart, MessageCircle, Users } from 'lucide-react';
import { analyticsService } from '../../services/analyticsService';

const AnalyticsDashboard: React.FC = () => {
  const { data: analytics } = useQuery({
    queryKey: ['analytics'],
    queryFn: analyticsService.getAnalytics
  });

  const { data: viewsData } = useQuery({
    queryKey: ['analytics-views'],
    queryFn: analyticsService.getViewsOverTime
  });

  if (!analytics) {
    return <div>Loading analytics...</div>;
  }

  const stats = [
    {
      label: 'Total Views',
      value: analytics.totalViews,
      icon: Eye,
      color: 'bg-blue-500'
    },
    {
      label: 'Total Likes',
      value: analytics.totalLikes,
      icon: Heart,
      color: 'bg-red-500'
    },
    {
      label: 'Comments',
      value: analytics.totalComments,
      icon: MessageCircle,
      color: 'bg-green-500'
    },
    {
      label: 'Followers',
      value: analytics.followerCount,
      icon: Users,
      color: 'bg-purple-500'
    }
  ];

  return (
    <div className="p-6">
      <h2 className="text-2xl font-bold text-gray-900 mb-6">Analytics</h2>
      
      {/* Stats Grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
        {stats.map((stat) => (
          <div key={stat.label} className="bg-white p-6 rounded-lg shadow-sm">
            <div className="flex items-center">
              <div className={`${stat.color} p-3 rounded-lg`}>
                <stat.icon className="w-6 h-6 text-white" />
              </div>
              <div className="ml-4">
                <p className="text-sm font-medium text-gray-600">{stat.label}</p>
                <p className="text-2xl font-bold text-gray-900">
                  {stat.value.toLocaleString()}
                </p>
              </div>
            </div>
          </div>
        ))}
      </div>

      {/* Views Over Time Chart */}
      <div className="bg-white p-6 rounded-lg shadow-sm mb-8">
        <h3 className="text-lg font-semibold text-gray-900 mb-4">Views Over Time</h3>
        <ResponsiveContainer width="100%" height={300}>
          <LineChart data={viewsData}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="date" />
            <YAxis />
            <Tooltip />
            <Line 
              type="monotone" 
              dataKey="views" 
              stroke="#ec4899" 
              strokeWidth={2}
              dot={{ fill: '#ec4899' }}
            />
          </LineChart>
        </ResponsiveContainer>
      </div>

      {/* Top Performing Shots */}
      <div className="bg-white p-6 rounded-lg shadow-sm">
        <h3 className="text-lg font-semibold text-gray-900 mb-4">Top Performing Shots</h3>
        <div className="space-y-4">
          {analytics.topShots?.map((shot: any) => (
            <div key={shot.id} className="flex items-center space-x-4">
              <img
                src={shot.imageUrl}
                alt={shot.title}
                className="w-16 h-16 object-cover rounded-lg"
              />
              <div className="flex-1">
                <h4 className="font-medium text-gray-900">{shot.title}</h4>
                <div className="flex space-x-4 text-sm text-gray-600">
                  <span className="flex items-center space-x-1">
                    <Eye className="w-4 h-4" />
                    <span>{shot.views}</span>
                  </span>
                  <span className="flex items-center space-x-1">
                    <Heart className="w-4 h-4" />
                    <span>{shot._count.likes}</span>
                  </span>
                  <span className="flex items-center space-x-1">
                    <MessageCircle className="w-4 h-4" />
                    <span>{shot._count.comments}</span>
                  </span>
                </div>
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default AnalyticsDashboard;
```

## Testing

### 1. Backend Testing Setup

```typescript
// Backend: src/tests/setup.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.TEST_DATABASE_URL
    }
  }
});

beforeAll(async () => {
  await prisma.$connect();
});

afterAll(async () => {
  await prisma.$disconnect();
});

beforeEach(async () => {
  // Clean database before each test
  await prisma.like.deleteMany();
  await prisma.comment.deleteMany();
  await prisma.shot.deleteMany();
  await prisma.user.deleteMany();
});

export { prisma };
```

### 2. API Tests

```typescript
// Backend: src/tests/auth.test.ts
import request from 'supertest';
import app from '../app';
import { prisma } from './setup';

describe('Authentication', () => {
  describe('POST /api/auth/register', () => {
    it('should register a new user', async () => {
      const userData = {
        email: 'test@example.com',
        username: 'testuser',
        name: 'Test User',
        password: 'password123'
      };

      const response = await request(app)
        .post('/api/auth/register')
        .send(userData)
        .expect(201);

      expect(response.body).toHaveProperty('user');
      expect(response.body).toHaveProperty('accessToken');
      expect(response.body.user.email).toBe(userData.email);
    });

    it('should not register user with existing email', async () => {
      const userData = {
        email: 'test@example.com',
        username: 'testuser1',
        name: 'Test User',
        password: 'password123'
      };

      // Create user first
      await request(app)
        .post('/api/auth/register')
        .send(userData);

      // Try to register with same email
      const response = await request(app)
        .post('/api/auth/register')
        .send({ ...userData, username: 'testuser2' })
        .expect(400);

      expect(response.body).toHaveProperty('error');
    });
  });

  describe('POST /api/auth/login', () => {
    beforeEach(async () => {
      await request(app)
        .post('/api/auth/register')
        .send({
          email: 'test@example.com',
          username: 'testuser',
          name: 'Test User',
          password: 'password123'
        });
    });

    it('should login with valid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'password123'
        })
        .expect(200);

      expect(response.body).toHaveProperty('user');
      expect(response.body).toHaveProperty('accessToken');
    });

    it('should not login with invalid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'wrongpassword'
        })
        .expect(401);

      expect(response.body).toHaveProperty('error');
    });
  });
});
```

### 3. Frontend Testing

```tsx
// Frontend: src/components/__tests__/ShotGrid.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import ShotGrid from '../shots/ShotGrid';

const mockShots = [
  {
    id: '1',
    title: 'Test Shot',
    imageUrl: 'https://example.com/image.jpg',
    user: {
      id: '1',
      username: 'testuser',
      name: 'Test User',
      avatar: null,
      isPro: false
    },
    _count: {
      likes: 5,
      comments: 2,
      saves: 1
    }
  }
];

const renderWithProviders = (component: React.ReactElement) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  });

  return render(
    <QueryClientProvider client={queryClient}>
      {component}
    </QueryClientProvider>
  );
};

describe('ShotGrid', () => {
  const mockOnLike = jest.fn();
  const mockOnSave = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders shots correctly', () => {
    renderWithProviders(
      <ShotGrid 
        shots={mockShots} 
        onLike={mockOnLike} 
        onSave={mockOnSave} 
      />
    );

    expect(screen.getByText('Test Shot')).toBeInTheDocument();
    expect(screen.getByText('Test User')).toBeInTheDocument();
    expect(screen.getByText('5')).toBeInTheDocument(); // likes count
  });

  it('calls onLike when like button is clicked', () => {
    renderWithProviders(
      <ShotGrid 
        shots={mockShots} 
        onLike={mockOnLike} 
        onSave={mockOnSave} 
      />
    );

    const likeButton = screen.getAllByRole('button')[0];
    fireEvent.click(likeButton);

    expect(mockOnLike).toHaveBeenCalledWith('1');
  });
});
```

## Deployment

### 1. Docker Configuration

```dockerfile
# Backend Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

EXPOSE 5000

CMD ["npm", "start"]
```

```dockerfile
# Frontend Dockerfile
FROM node:18-alpine as build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 2. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: dribbble_clone
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/dribbble_clone
      REDIS_URL: redis://redis:6379
      JWT_SECRET: your-secret-key
    depends_on:
      - postgres
      - redis
    ports:
      - "5000:5000"

  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend

volumes:
  postgres_data:
```

### 3. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
          cache-dependency-path: |
            backend/package-lock.json
            frontend/package-lock.json
      
      - name: Install backend dependencies
        run: cd backend && npm ci
      
      - name: Install frontend dependencies
        run: cd frontend && npm ci
      
      - name: Run backend tests
        run: cd backend && npm test
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
      
      - name: Run frontend tests
        run: cd frontend && npm test -- --coverage --watchAll=false
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Production
        run: |
          # Add your deployment commands here
          echo "Deploying to production..."
```

## Performance Optimization

### 1. Database Optimization

```sql
-- Add database indexes for better performance
-- Add to your migration file

-- Index for shots query optimization
CREATE INDEX idx_shots_created_at ON shots(created_at DESC);
CREATE INDEX idx_shots_user_id ON shots(user_id);
CREATE INDEX idx_shots_tags ON shots USING GIN(tags);

-- Index for likes
CREATE INDEX idx_likes_shot_id ON likes(shot_id);
CREATE INDEX idx_likes_user_id ON likes(user_id);

-- Index for comments
CREATE INDEX idx_comments_shot_id ON comments(shot_id);
CREATE INDEX idx_comments_created_at ON comments(created_at DESC);

-- Index for follows
CREATE INDEX idx_follows_follower_id ON follows(follower_id);
CREATE INDEX idx_follows_following_id ON follows(following_id);

-- Full text search index
CREATE INDEX idx_shots_search ON shots USING GIN(to_tsvector('english', title || ' ' || COALESCE(description, '')));
```

### 2. Caching Strategy

```typescript
// Backend: src/services/cacheService.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

export class CacheService {
  static async get<T>(key: string): Promise<T | null> {
    try {
      const cached = await redis.get(key);
      return cached ? JSON.parse(cached) : null;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  static async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    try {
      await redis.setex(key, ttl, JSON.stringify(value));
    } catch (error) {
      console.error('Cache set error:', error);
    }
  }

  static async del(key: string): Promise<void> {
    try {
      await redis.del(key);
    } catch (error) {
      console.error('Cache delete error:', error);
    }
  }

  static async invalidatePattern(pattern: string): Promise<void> {
    try {
      const keys = await redis.keys(pattern);
      if (keys.length > 0) {
        await redis.del(...keys);
      }
    } catch (error) {
      console.error('Cache invalidate error:', error);
    }
  }
}

// Usage in shot controller
export const getShots = async (req: Request, res: Response) => {
  try {
    const cacheKey = `shots:${JSON.stringify(req.query)}`;
    const cached = await CacheService.get(cacheKey);
    
    if (cached) {
      return res.json(cached);
    }

    // ... fetch from database logic ...
    
    // Cache the result
    await CacheService.set(cacheKey, result, 600); // 10 minutes
    
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch shots' });
  }
};
```

### 3. Frontend Optimization

```tsx
// Frontend: src/components/shots/VirtualizedShotGrid.tsx
import React, { useMemo } from 'react';
import { FixedSizeGrid as Grid } from 'react-window';
import { Shot } from '../../types';

interface VirtualizedShotGridProps {
  shots: Shot[];
  onLike: (shotId: string) => void;
  onSave: (shotId: string) => void;
}

const COLUMN_COUNT = 4;
const ITEM_WIDTH = 300;
const ITEM_HEIGHT = 400;

const VirtualizedShotGrid: React.FC<VirtualizedShotGridProps> = ({
  shots,
  onLike,
  onSave
}) => {
  const rowCount = Math.ceil(shots.length / COLUMN_COUNT);

  const Cell = useMemo(() => {
    return ({ columnIndex, rowIndex, style }: any) => {
      const shotIndex = rowIndex * COLUMN_COUNT + columnIndex;
      const shot = shots[shotIndex];

      if (!shot) return <div style={style} />;

      return (
        <div style={{ ...style, padding: '8px' }}>
          <ShotCard shot={shot} onLike={onLike} onSave={onSave} />
        </div>
      );
    };
  }, [shots, onLike, onSave]);

  return (
    <Grid
      columnCount={COLUMN_COUNT}
      columnWidth={ITEM_WIDTH}
      height={600}
      rowCount={rowCount}
      rowHeight={ITEM_HEIGHT}
      width="100%"
    >
      {Cell}
    </Grid>
  );
};

export default VirtualizedShotGrid;
```

### 4. Image Optimization

```tsx
// Frontend: src/components/common/OptimizedImage.tsx
import React, { useState, useRef, useEffect } from 'react';

interface OptimizedImageProps {
  src: string;
  alt: string;
  className?: string;
  placeholder?: string;
  sizes?: string;
}

const OptimizedImage: React.FC<OptimizedImageProps> = ({
  src,
  alt,
  className,
  placeholder = 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMzIwIiBoZWlnaHQ9IjI0MCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cmVjdCB3aWR0aD0iMTAwJSIgaGVpZ2h0PSIxMDAlIiBmaWxsPSIjZjNmNGY2Ii8+PC9zdmc+',
  sizes = '(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw'
}) => {
  const [isLoaded, setIsLoaded] = useState(false);
  const [isInView, setIsInView] = useState(false);
  const imgRef = useRef<HTMLImageElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsInView(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, []);

  // Generate responsive image URLs (assuming Cloudinary)
  const generateSrcSet = (originalUrl: string) => {
    const baseUrl = originalUrl.replace('/upload/', '/upload/');
    return [
      `${baseUrl.replace('/upload/', '/upload/w_400/')} 400w`,
      `${baseUrl.replace('/upload/', '/upload/w_800/')} 800w`,
      `${baseUrl.replace('/upload/', '/upload/w_1200/')} 1200w`
    ].join(', ');
  };

  return (
    <div className={`relative overflow-hidden ${className}`}>
      <img
        ref={imgRef}
        src={isInView ? src : placeholder}
        srcSet={isInView ? generateSrcSet(src) : undefined}
        sizes={sizes}
        alt={alt}
        className={`transition-opacity duration-300 ${isLoaded ? 'opacity-100' : 'opacity-0'}`}
        onLoad={() => setIsLoaded(true)}
        loading="lazy"
      />
      {!isLoaded && isInView && (
        <div className="absolute inset-0 bg-gray-200 animate-pulse" />
      )}
    </div>
  );
};

export default OptimizedImage;
```

## Additional Features

### 1. Pro Membership System

```typescript
// Backend: src/controllers/subscriptionController.ts
import Stripe from 'stripe';
import { Request, Response } from 'express';
import { prisma } from '../app';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16',
});

export const createSubscription = async (req: Request, res: Response) => {
  try {
    const { priceId } = req.body;
    const userId = req.user!.id;

    const user = await prisma.user.findUnique({
      where: { id: userId }
    });

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    // Create Stripe customer if doesn't exist
    let customerId = user.stripeCustomerId;
    if (!customerId) {
      const customer = await stripe.customers.create({
        email: user.email,
        metadata: { userId }
      });
      
      customerId = customer.id;
      await prisma.user.update({
        where: { id: userId },
        data: { stripeCustomerId: customerId }
      });
    }

    // Create subscription
    const subscription = await stripe.subscriptions.create({
      customer: customerId,
      items: [{ price: priceId }],
      payment_behavior: 'default_incomplete',
      payment_settings: { save_default_payment_method: 'on_subscription' },
      expand: ['latest_invoice.payment_intent'],
    });

    res.json({
      subscriptionId: subscription.id,
      clientSecret: (subscription.latest_invoice as any).payment_intent.client_secret,
    });
  } catch (error) {
    res.status(500).json({ error: 'Failed to create subscription' });
  }
};

export const handleWebhook = async (req: Request, res: Response) => {
  const sig = req.headers['stripe-signature'] as string;
  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    return res.status(400).send(`Webhook signature verification failed.`);
  }

  switch (event.type) {
    case 'invoice.payment_succeeded':
      const invoice = event.data.object as Stripe.Invoice;
      await handleSuccessfulPayment(invoice);
      break;
    
    case 'invoice.payment_failed':
      const failedInvoice = event.data.object as Stripe.Invoice;
      await handleFailedPayment(failedInvoice);
      break;
    
    case 'customer.subscription.deleted':
      const subscription = event.data.object as Stripe.Subscription;
      await handleCancelledSubscription(subscription);
      break;
  }

  res.json({ received: true });
};

const handleSuccessfulPayment = async (invoice: Stripe.Invoice) => {
  const customerId = invoice.customer as string;
  
  const user = await prisma.user.findFirst({
    where: { stripeCustomerId: customerId }
  });

  if (user) {
    await prisma.user.update({
      where: { id: user.id },
      data: { 
        isPro: true,
        proExpiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) // 30 days
      }
    });
  }
};
```

### 2. Job Board Feature

```typescript
// Backend: src/models/job.ts (Prisma schema addition)
model Job {
  id          String   @id @default(cuid())
  title       String
  company     String
  description String
  location    String?
  remote      Boolean  @default(false)
  salary      String?
  type        JobType  @default(FULL_TIME)
  tags        String[]
  applyUrl    String?
  applyEmail  String?
  isActive    Boolean  @default(true)
  isPremium   Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  expiresAt   DateTime

  userId String
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("jobs")
}

enum JobType {
  FULL_TIME
  PART_TIME
  CONTRACT
  FREELANCE
  INTERNSHIP
}
```

```tsx
// Frontend: src/components/jobs/JobCard.tsx
import React from 'react';
import { MapPin, Clock, DollarSign, ExternalLink } from 'lucide-react';
import { Job } from '../../types';

interface JobCardProps {
  job: Job;
}

const JobCard: React.FC<JobCardProps> = ({ job }) => {
  const formatSalary = (salary: string) => {
    return salary.replace(/\B(?=(\d{3})+(?!\d))/g, ',');
  };

  return (
    <div className="bg-white border border-gray-200 rounded-lg p-6 hover:shadow-lg transition-shadow">
      <div className="flex justify-between items-start mb-4">
        <div>
          <h3 className="text-lg font-semibold text-gray-900 mb-1">
            {job.title}
          </h3>
          <p className="text-gray-600 font-medium">{job.company}</p>
        </div>
        {job.isPremium && (
          <span className="bg-yellow-100 text-yellow-800 text-xs px-2 py-1 rounded-full">
            Featured
          </span>
        )}
      </div>

      <div className="flex flex-wrap items-center gap-4 text-sm text-gray-500 mb-4">
        {job.location && (
          <div className="flex items-center space-x-1">
            <MapPin className="w-4 h-4" />
            <span>{job.location}</span>
          </div>
        )}
        {job.remote && (
          <span className="bg-green-100 text-green-800 px-2 py-1 rounded-full">
            Remote
          </span>
        )}
        <div className="flex items-center space-x-1">
          <Clock className="w-4 h-4" />
          <span>{job.type.replace('_', ' ')}</span>
        </div>
        {job.salary && (
          <div className="flex items-center space-x-1">
            <DollarSign className="w-4 h-4" />
            <span>{formatSalary(job.salary)}</span>
          </div>
        )}
      </div>

      <p className="text-gray-700 mb-4 line-clamp-3">
        {job.description}
      </p>

      <div className="flex flex-wrap gap-2 mb-4">
        {job.tags.slice(0, 3).map((tag) => (
          <span
            key={tag}
            className="bg-gray-100 text-gray-700 text-xs px-2 py-1 rounded-full"
          >
            {tag}
          </span>
        ))}
        {job.tags.length > 3 && (
          <span className="text-gray-500 text-xs">
            +{job.tags.length - 3} more
          </span>
        )}
      </div>

      <div className="flex justify-between items-center">
        <span className="text-sm text-gray-500">
          Posted {new Date(job.createdAt).toLocaleDateString()}
        </span>
        <a
          href={job.applyUrl || `mailto:${job.applyEmail}`}
          target="_blank"
          rel="noopener noreferrer"
          className="inline-flex items-center space-x-1 bg-pink-600 text-white px-4 py-2 rounded-md hover:bg-pink-700 transition-colors"
        >
          <span>Apply</span>
          <ExternalLink className="w-4 h-4" />
        </a>
      </div>
    </div>
  );
};

export default JobCard;
```

### 3. Advanced Search with Filters

```tsx
// Frontend: src/components/search/AdvancedSearch.tsx
import React, { useState } from 'react';
import { Search, Filter, X } from 'lucide-react';

interface AdvancedSearchProps {
  onSearch: (filters: SearchFilters) => void;
  loading?: boolean;
}

interface SearchFilters {
  query: string;
  tags: string[];
  colors: string[];
  dateRange: string;
  sort: string;
  pro: boolean;
}

const POPULAR_TAGS = [
  'web design', 'mobile', 'ui/ux', 'logo', 'branding', 
  'illustration', 'typography', 'icon', 'app design', 'landing page'
];

const COLOR_PALETTE = [
  '#ff6b6b', '#4ecdc4', '#45b7d1', '#96ceb4', '#feca57',
  '#ff9ff3', '#54a0ff', '#5f27cd', '#00d2d3', '#ff9f43'
];

const AdvancedSearch: React.FC<AdvancedSearchProps> = ({ onSearch, loading }) => {
  const [showFilters, setShowFilters] = useState(false);
  const [filters, setFilters] = useState<SearchFilters>({
    query: '',
    tags: [],
    colors: [],
    dateRange: 'all',
    sort: 'recent',
    pro: false
  });

  const handleTagToggle = (tag: string) => {
    setFilters(prev => ({
      ...prev,
      tags: prev.tags.includes(tag)
        ? prev.tags.filter(t => t !== tag)
        : [...prev.tags, tag]
    }));
  };

  const handleColorToggle = (color: string) => {
    setFilters(prev => ({
      ...prev,
      colors: prev.colors.includes(color)
        ? prev.colors.filter(c => c !== color)
        : [...prev.colors, color]
    }));
  };

  const handleSearch = () => {
    onSearch(filters);
  };

  const clearFilters = () => {
    setFilters({
      query: '',
      tags: [],
      colors: [],
      dateRange: 'all',
      sort: 'recent',
      pro: false
    });
  };

  const activeFiltersCount = filters.tags.length + filters.colors.length + 
    (filters.dateRange !== 'all' ? 1 : 0) + (filters.pro ? 1 : 0);

  return (
    <div className="bg-white shadow-sm border-b">
      <div className="max-w-7xl mx-auto px-4 py-4">
        {/* Search Bar */}
        <div className="flex items-center space-x-4 mb-4">
          <div className="flex-1 relative">
            <Search className="absolute left-3 top-1/2 transform -translate-y-1/2 text-gray-400 w-5 h-5" />
            <input
              type="text"
              value={filters.query}
              onChange={(e) => setFilters(prev => ({ ...prev, query: e.target.value }))}
              placeholder="Search for shots..."
              className="w-full pl-10 pr-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-pink-500 focus:border-transparent"
              onKeyPress={(e) => e.key === 'Enter' && handleSearch()}
            />
          </div>
          <button
            onClick={() => setShowFilters(!showFilters)}
            className={`flex items-center space-x-2 px-4 py-3 border rounded-lg transition-colors ${
              showFilters || activeFiltersCount > 0 
                ? 'bg-pink-50 border-pink-200 text-pink-700' 
                : 'border-gray-300 hover:bg-gray-50'
            }`}
          >
            <Filter className="w-5 h-5" />
            <span>Filters</span>
            {activeFiltersCount > 0 && (
              <span className="bg-pink-600 text-white text-xs rounded-full w-5 h-5 flex items-center justify-center">
                {activeFiltersCount}
              </span>
            )}
          </button>
          <button
            onClick={handleSearch}
            disabled={loading}
            className="bg-pink-600 text-white px-6 py-3 rounded-lg hover:bg-pink-700 disabled:opacity-50 transition-colors"
          >
            {loading ? 'Searching...' : 'Search'}
          </button>
        </div>

        {/* Filters Panel */}
        {showFilters && (
          <div className="border-t pt-4 space-y-6">
            {/* Tags */}
            <div>
              <h3 className="text-sm font-medium text-gray-900 mb-3">Tags</h3>
              <div className="flex flex-wrap gap-2">
                {POPULAR_TAGS.map((tag) => (
                  <button
                    key={tag}
                    onClick={() => handleTagToggle(tag)}
                    className={`px-3 py-1 rounded-full text-sm transition-colors ${
                      filters.tags.includes(tag)
                        ? 'bg-pink-600 text-white'
                        : 'bg-gray-100 text-gray-700 hover:bg-gray-200'
                    }`}
                  >
                    {tag}
                  </button>
                ))}
              </div>
            </div>

            {/* Colors */}
            <div>
              <h3 className="text-sm font-medium text-gray-900 mb-3">Colors</h3>
              <div className="flex flex-wrap gap-2">
                {COLOR_PALETTE.map((color) => (
                  <button
                    key={color}
                    onClick={() => handleColorToggle(color)}
                    className={`w-8 h-8 rounded-full border-2 transition-all ${
                      filters.colors.includes(color)
                        ? 'border-gray-900 scale-110'
                        : 'border-gray-300 hover:border-gray-400'
                    }`}
                    style={{ backgroundColor: color }}
                  />
                ))}
              </div>
            </div>

            {/* Date Range & Sort */}
            <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
              <div>
                <label className="block text-sm font-medium text-gray-900 mb-2">
                  Date Range
                </label>
                <select
                  value={filters.dateRange}
                  onChange={(e) => setFilters(prev => ({ ...prev, dateRange: e.target.value }))}
                  className="w-full px-3 py-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-pink-500"
                >
                  <option value="all">All Time</option>
                  <option value="today">Today</option>
                  <option value="week">This Week</option>
                  <option value="month">This Month</option>
                  <option value="year">This Year</option>
                </select>
              </div>

              <div>
                <label className="block text-sm font-medium text-gray-900 mb-2">
                  Sort By
                </label>
                <select
                  value={filters.sort}
                  onChange={(e) => setFilters(prev => ({ ...prev, sort: e.target.value }))}
                  className="w-full px-3 py-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-pink-500"
                >
                  <option value="recent">Most Recent</option>
                  <option value="popular">Most Popular</option>
                  <option value="views">Most Viewed</option>
                  <option value="comments">Most Commented</option>
                </select>
              </div>

              <div>
                <label className="block text-sm font-medium text-gray-900 mb-2">
                  Filters
                </label>
                <label className="flex items-center space-x-2">
                  <input
                    type="checkbox"
                    checked={filters.pro}
                    onChange={(e) => setFilters(prev => ({ ...prev, pro: e.target.checked }))}
                    className="rounded border-gray-300 text-pink-600 focus:ring-pink-500"
                  />
                  <span className="text-sm text-gray-700">Pro Members Only</span>
                </label>
              </div>
            </div>

            {/* Clear Filters */}
            {activeFiltersCount > 0 && (
              <div className="flex justify-end">
                <button
                  onClick={clearFilters}
                  className="flex items-center space-x-1 text-gray-600 hover:text-gray-800"
                >
                  <X className="w-4 h-4" />
                  <span>Clear all filters</span>
                </button>
              </div>
            )}
          </div>
        )}
      </div>
    </div>
  );
};

export default AdvancedSearch;
```

## Security Considerations

### 1. Rate Limiting

```typescript
// Backend: src/middleware/rateLimiter.ts
import rateLimit from 'express-rate-limit';
import { Request, Response } from 'express';

export const globalRateLimit = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 1000, // limit each IP to 1000 requests per windowMs
  message: 'Too many requests from this IP, please try again later.',
  standardHeaders: true,
  legacyHeaders: false,
});

export const authRateLimit = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // limit each IP to 5 login requests per windowMs
  message: 'Too many login attempts, please try again later.',
  skipSuccessfulRequests: true,
});

export const uploadRateLimit = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 10, // limit each user to 10 uploads per hour
  keyGenerator: (req: Request) => req.user?.id || req.ip,
  message: 'Upload limit exceeded, please try again later.',
});
```

### 2. Input Validation & Sanitization

```typescript
// Backend: src/middleware/validation.ts
import { body, query, param } from 'express-validator';
import DOMPurify from 'isomorphic-dompurify';

export const validateShotCreation = [
  body('title')
    .trim()
    .isLength({ min: 1, max: 100 })
    .withMessage('Title must be between 1 and 100 characters')
    .customSanitizer(value => DOMPurify.sanitize(value)),
  
  body('description')
    .optional()
    .isLength({ max: 1000 })
    .withMessage('Description must not exceed 1000 characters')
    .customSanitizer(value => DOMPurify.sanitize(value)),
  
  body('tags')
    .optional()
    .isString()
    .custom((value) => {
      const tags = value.split(',').map((tag: string) => tag.trim());
      if (tags.length > 10) {
        throw new Error('Maximum 10 tags allowed');
      }
      return true;
    }),
];

export const validateSearch = [
  query('search')
    .optional()
    .isLength({ max: 100 })
    .withMessage('Search query too long')
    .customSanitizer(value => DOMPurify.sanitize(value)),
  
  query('page')
    .optional()
    .isInt({ min: 1, max: 1000 })
    .withMessage('Invalid page number'),
  
  query('limit')
    .optional()
    .isInt({ min: 1, max: 50 })
    .withMessage('Invalid limit'),
];
```

### 3. CORS & Security Headers

```typescript
// Backend: src/middleware/security.ts
import cors from 'cors';
import helmet from 'helmet';

export const corsOptions = {
  origin: process.env.NODE_ENV === 'production' 
    ? ['https://yourdomain.com', 'https://www.yourdomain.com']
    : ['http://localhost:3000', 'http://localhost:5173'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
};

export const helmetConfig = helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "https://res.cloudinary.com", "data:", "https:"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      connectSrc: ["'self'", process.env.API_URL],
    },
  },
  crossOriginEmbedderPolicy: false,
});
```

## Monitoring & Analytics

### 1. Application Monitoring

```typescript
// Backend: src/middleware/monitoring.ts
import { Request, Response, NextFunction } from 'express';

interface RequestMetrics {
  method: string;
  url: string;
  statusCode: number;
  responseTime: number;
  timestamp: Date;
  userAgent?: string;
  ip: string;
}

export const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  const startTime = Date.now();

  res.on('finish', () => {
    const metrics: RequestMetrics = {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      responseTime: Date.now() - startTime,
      timestamp: new Date(),
      userAgent: req.headers['user-agent'],
      ip: req.ip,
    };

    // Log to console (in production, use a proper logging service)
    console.log(JSON.stringify(metrics));

    // Send to analytics service (optional)
    if (process.env.ANALYTICS_ENDPOINT) {
      sendToAnalytics(metrics);
    }
  });

  next();
};

const sendToAnalytics = async (metrics: RequestMetrics) => {
  try {
    // Implementation depends on your analytics service
    // Example: Google Analytics, Mixpanel, custom analytics
  } catch (error) {
    console.error('Failed to send analytics:', error);
  }
};
```

## Conclusion

This comprehensive build guide covers all aspects of creating a full-featured Dribbble replica, including:

- **Core Features**: User authentication, shot management, social interactions
- **Advanced Features**: Real-time notifications, collections, pro memberships, job board
- **Performance**: Caching, image optimization, virtualization, database indexing
- **Security**: Rate limiting, input validation, CORS, security headers
- **Testing**: Unit tests, integration tests, E2E testing strategies
- **Deployment**: Docker, CI/CD, production configurations
- **Monitoring**: Request logging, analytics, error tracking

### Next Steps

1. **Start with the MVP**: Implement core features first (auth, shots, basic interactions)
2. **Iterate**: Add advanced features based on user feedback
3. **Scale**: Implement performance optimizations as your user base grows
4. **Monitor**: Set up proper monitoring and analytics from day one
5. **Security**: Regular security audits and updates

### Additional Resources

- [Prisma Documentation](https://www.prisma.io/docs)
- [React Query Documentation](https://tanstack.com/query/latest)
- [Stripe Integration Guide](https://stripe.com/docs)
- [Socket.io Documentation](https://socket.io/docs)
- [Cloudinary Image Optimization](https://cloudinary.com/documentation)

This guide provides a solid foundation for building a professional-grade design portfolio platform. Remember to adapt the implementation based on your specific requirements and constraints.
