# Technical Interview Roadmap

## Overview

| # | Topic | Priority | Key Focus Areas |
| --- | --- | --- | --- |
| 1 | [Core Java and JVM](#1-core-java-and-jvm) | **Critical** | Java 8–21+, concurrency, collections, JVM internals, garbage collection, virtual threads |
| 2 | [Microservices and Distributed Systems](#2-microservices-and-distributed-systems) | **Critical** | Service decomposition, communication, resilience, consistency, transactions, scalability, observability |
| 3 | [Spring Boot and Hibernate](#3-spring-boot-and-hibernate) | **Critical** | Dependency injection, auto-configuration, transactions, JPA, Hibernate internals, REST APIs |
| 4 | [Quarkus and Reactive Programming](#4-quarkus-and-reactive-programming) | **Critical** | Quarkus architecture, CDI, Arc, build-time augmentation, Vert.x, Mutiny, GraalVM native image |
| 5 | [Multithreading and Concurrency](#5-multithreading-and-concurrency) | **Critical** | Race conditions, synchronization, locks, thread pools, virtual threads, deadlocks, memory model |
| 6 | [Kafka and RabbitMQ](#6-kafka-and-rabbitmq) | **High** | Partitions, consumer groups, offsets, delivery semantics, retries, DLQ, ordering, idempotency |
| 7 | [Database and Redis](#7-database-and-redis) | **High** | PostgreSQL, MySQL, indexing, query optimization, transactions, isolation, caching |
| 8 | [Kubernetes, Docker and AWS](#8-kubernetes-docker-and-aws) | **High** | Containerization, EKS, deployments, probes, autoscaling, networking, IAM, S3 |
| 9 | [Testing and CI/CD](#9-testing-and-cicd) | **High** | JUnit 5, Mockito, Testcontainers, REST Assured, GitHub Actions, deployment and rollback |
| 10 | [Security and Keycloak](#10-security-and-keycloak) | **High** | OAuth 2.0, OIDC, JWT, RBAC, password and secret security, session management |
| 11 | [GraphQL and gRPC](#11-graphql-and-grpc) | **Medium** | Protocol differences, streaming, schema design, HTTP/2, interceptors, error handling |
| 12 | [React and Node.js](#12-react-and-nodejs) | **Medium** | React fundamentals, hooks, API integration, asynchronous Node.js, event loop |
| 13 | [AWS](#13-aws) | **High** | IAM, VPC and networking, EC2, EKS and ECS, S3, RDS and DynamoDB, SQS/SNS, Lambda, API Gateway, ALB/Route 53, CloudWatch, Secrets Manager and KMS, IaC |

---

## 1. Core Java and JVM

> **Priority:** Critical

- Java 8–21+
- Concurrency
- Collections
- JVM internals
- Garbage collection
- Virtual threads

## 2. Microservices and Distributed Systems

> **Priority:** Critical

- Service decomposition
- Communication
- Resilience
- Consistency
- Transactions
- Scalability
- Observability

> **Detailed roadmap:** See [microservices-roadmap.md](./microservices-roadmap.md)

## 3. Spring Boot and Hibernate

> **Priority:** Critical

- Dependency injection
- Auto-configuration
- Transactions
- JPA
- Hibernate internals
- REST APIs

## 4. Quarkus and Reactive Programming

> **Priority:** Critical

- Quarkus architecture
- CDI
- Arc
- Build-time augmentation
- Vert.x
- Mutiny
- GraalVM native image

## 5. Multithreading and Concurrency

> **Priority:** Critical

- Race conditions
- Synchronization
- Locks
- Thread pools
- Virtual threads
- Deadlocks
- Memory model

## 6. Kafka and RabbitMQ

> **Priority:** High

- Partitions
- Consumer groups
- Offsets
- Delivery semantics
- Retries
- DLQ
- Ordering
- Idempotency

## 7. Database and Redis

> **Priority:** High

- PostgreSQL
- MySQL
- Indexing
- Query optimization
- Transactions
- Isolation
- Caching

## 8. Kubernetes, Docker and AWS

> **Priority:** High

- Containerization
- EKS
- Deployments
- Probes
- Autoscaling
- Networking
- IAM
- S3

## 9. Testing and CI/CD

> **Priority:** High

- JUnit 5
- Mockito
- Testcontainers
- REST Assured
- GitHub Actions
- Deployment and rollback

## 10. Security and Keycloak

> **Priority:** High

- OAuth 2.0
- OIDC
- JWT
- RBAC
- Password and secret security
- Session management

## 11. GraphQL and gRPC

> **Priority:** Medium

- Protocol differences
- Streaming
- Schema design
- HTTP/2
- Interceptors
- Error handling

## 12. React and Node.js

> **Priority:** Medium

- React fundamentals
- Hooks
- API integration
- Asynchronous Node.js
- Event loop

## 13. AWS

> **Priority:** High

- IAM (roles, policies, least privilege, IRSA)
- VPC and networking (subnets, route tables, NAT/IGW, security groups, NACLs)
- Compute (EC2, Auto Scaling, ECS/Fargate, EKS)
- Storage (S3, lifecycle policies, encryption, EBS/EFS)
- Databases (RDS, Aurora, ElastiCache, DynamoDB)
- Messaging and event-driven services (SQS, SNS, EventBridge, MSK)
- Serverless (Lambda, API Gateway)
- Load balancing and DNS (ALB, NLB, Route 53, CloudFront)
- Observability (CloudWatch logs/metrics/alarms, X-Ray, CloudTrail)
- Security and secrets (Secrets Manager, Parameter Store, KMS, ACM)
- Infrastructure as Code (Terraform, CloudFormation)
