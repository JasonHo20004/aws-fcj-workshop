+++
title = "Overview & Introduction"
weight = 1
+++

## Workshop Overview

**Workshop Name**: Deploy Real-Time Chat System on AWS Serverless  
**Objective**: Deploy and operate a complete real-time chat system using serverless microservices architecture on AWS  
**Estimated Duration**: 3-4 hours (can be divided into multiple sessions)  
**Level**: Intermediate (AWS knowledge required)

## What You'll Build

In this workshop, you'll deploy a **Real-Time Communication Platform** - a fully functional chat application similar to WhatsApp or Telegram, built entirely on AWS serverless services. The platform includes:

### **Core Features**
- **User Authentication & Registration** - Complete user management with AWS Cognito
- **Real-time Messaging** - Instant message delivery using WebSocket APIs
- **User Search & Discovery** - Find and connect with other users
- **Direct Conversations** - Create private chats between users
- **Message Persistence** - All messages stored in DynamoDB
- **Modern React Frontend** - Beautiful UI built with React, TypeScript, and Tailwind CSS

### **Architecture Overview**

The system follows a **microservices architecture** with these core services:

![Architecture Overview](/images/serverless-chatapp-structure.drawio.png)

*The architecture diagram shows the microservices structure with React frontend, WebSocket API Gateway, SNS topics, Lambda services, and DynamoDB tables interconnected for real-time chat functionality.*

### **AWS Services Used**

| Service | Purpose | What You'll Deploy |
|---------|---------|-------------------|
| **AWS Lambda** | Serverless compute | 4 microservices with 15+ functions |
| **API Gateway** | HTTP & WebSocket APIs | REST endpoints + real-time connections |
| **DynamoDB** | NoSQL database | 5 tables with optimized indexes |
| **AWS Cognito** | User authentication | User pools and identity management |
| **SNS** | Message fan-out | Real-time notifications between services |
| **ElastiCache** | Redis caching | Performance optimization for real-time features |
| **IAM** | Security & permissions | Role-based access control |
| **VPC** | Network isolation | Private subnets for Redis cluster |

### **Database Schema**

The system automatically creates these DynamoDB tables:

- **`Users`** - User profiles and authentication data
- **`Messages`** - Chat messages with conversation indexing
- **`conversations`** - Chat room metadata and settings
- **`conversation-members-table`** - User membership in conversations
- **`websocket-connections`** - Active WebSocket connections

### **Frontend Technology Stack**

- **React 18** with TypeScript for type safety
- **Vite** for fast development and building
- **Tailwind CSS** for modern, responsive design
- **Radix UI** components for accessible UI elements
- **React Router** for client-side navigation
- **React Query** for server state management
- **WebSocket API** for real-time communication

## Workshop Learning Outcomes

By the end of this workshop, you will:

 **Understand serverless architecture** - Learn how to build scalable applications without managing servers  
 **Master AWS Lambda deployment** - Deploy multiple microservices using Serverless Framework  
 **Implement real-time features** - Build WebSocket APIs for instant messaging  
 **Integrate AWS services** - Connect Cognito, SNS, ElastiCache, and more  
 **Deploy full-stack applications** - From backend services to React frontend  
 **Monitor and operate** - Set up CloudWatch monitoring and troubleshooting  

## Prerequisites

- **AWS Account** (free tier is sufficient)
- **Basic AWS knowledge** (Lambda, DynamoDB, API Gateway)
- **Node.js 18+** and npm installed
- **AWS CLI** configured with appropriate permissions
- **Serverless Framework** installed globally

## What's Next?

In the following sections, you'll:
1. **Set up your development environment** and AWS credentials
2. **Deploy the infrastructure** including VPC, Redis cluster, and IAM roles
3. **Deploy all microservices** using Serverless Framework
4. **Configure the frontend** and test the complete system
5. **Monitor and optimize** your deployment
6. **Clean up resources** to avoid unnecessary costs

Ready to build your own real-time chat platform? Let's get started! 🚀
