# Authentication & Authorization Service Specification

## Purpose

The Authentication & Authorization Service provides secure, scalable identity management and access control for the Digital Lending Platform. The service implements OAuth2/OpenID Connect standards with JWT-based authentication, role-based authorization, and comprehensive audit capabilities.

## Technology Stack

### Runtime Environment
- **Framework**: Spring Boot 3.x with Spring Security 6.x
- **Language**: Java 17+ with Maven build system
- **Authentication**: OAuth2/OpenID Connect with JWT tokens
- **Database**: PostgreSQL 14+ with JPA/Hibernate ORM
- **Caching**: Redis 7+ for session and token management
- **Messaging**: Apache Kafka for event streaming
- **Deployment**: Kubernetes with Docker containers

### External Integrations
- **Identity Provider**: Keycloak or AWS Cognito for OAuth2/OIDC
- **Communication**: SMS/Email providers for OTP delivery
- **Storage**: S3/Blob storage for document uploads
- **Monitoring**: Prometheus metrics and distributed tracing

## Service Components

### Presentation Layer
- **AuthController**: REST API endpoints for authentication operations
  - Handles HTTP requests/responses with proper error handling
  - Implements request validation and security headers
  - Manages session context and device tracking

### Business Layer
- **AuthService**: Core authentication and authorization logic
  - Implements OAuth2 flows and JWT token management
  - Handles user registration, login, and profile updates
  - Manages role-based access control and permissions
- **SecurityValidator**: Token validation and security enforcement
  - Validates JWT signatures and expiration
  - Implements RBAC policy evaluation
  - Handles multi-factor authentication workflows

### Data Access Layer
- **AuthRepository**: Database operations for user data
  - JPA/Hibernate implementation with PostgreSQL
  - Optimized queries for user lookup and session management
  - Transaction management for data consistency

### Integration Layer
- **OAuthClient**: External identity provider integration
  - OAuth2 client configuration and token exchange
  - Provider-specific authentication flows
  - Fallback mechanisms for provider unavailability
- **EventPublisher**: Kafka event publishing for audit trails
  - Publishes authentication events asynchronously
  - Ensures event delivery with retry mechanisms
  - Maintains event schema compatibility

## API Specifications

### Authentication Endpoints

#### POST /auth/register
**Purpose**: Register new user account with email/phone verification
**Input**: UserRegistrationRequest (email, phone, password)
**Output**: 201 UserResponse with userId and roles, or 400 ValidationErrorResponse
**Flow**: Validate input → Check uniqueness → Hash password → Store user → Return response

#### POST /auth/login  
**Purpose**: Authenticate user credentials and issue JWT tokens
**Input**: LoginRequest (username/email, password)
**Output**: 200 TokenResponse with access/refresh tokens, or 401 ErrorResponse
**Flow**: Validate credentials → Generate JWT → Store session → Publish login event → Return tokens

#### POST /auth/refresh
**Purpose**: Refresh expired access token using valid refresh token
**Input**: TokenRefreshRequest (refreshToken)
**Output**: 200 TokenResponse with new tokens, or 400 ErrorResponse
**Flow**: Validate refresh token → Generate new access token → Update session → Return tokens

### Profile Management Endpoints

#### GET /profiles
**Purpose**: List user and co-applicant profiles with pagination and filtering
**Security**: Bearer token with 'profile.read' scope
**Parameters**: page, size, filter (search, sortBy, order)
**Output**: 200 ProfileListResponse with items and pagination metadata
**Flow**: Validate token → Apply filters → Query database → Return paginated results

#### GET /profiles/{profileId}
**Purpose**: Retrieve specific user profile by ID
**Security**: Bearer token with 'profile.read' scope  
**Output**: 200 UserProfile or 404 ErrorResponse
**Flow**: Validate token → Check permissions → Query profile → Return data

#### PUT /profiles/{profileId}
**Purpose**: Update user profile information with validation
**Security**: Bearer token with 'profile.write' scope
**Input**: UserProfileUpdateRequest (firstName, lastName, email, phone, address)
**Output**: 200 UserProfile or 400 ValidationErrorResponse
**Flow**: Validate token → Check permissions → Validate changes → Update profile → Publish event

#### POST /profiles/{profileId}/actions/requestChange
**Purpose**: Submit change request for sensitive profile fields requiring approval
**Security**: Bearer token with 'profile.write' scope
**Input**: ChangeRequest (fieldName, newValue, reason)
**Output**: 202 ChangeRequestResponse with requestId and status
**Flow**: Validate request → Create change request → Send OTP → Return request ID

#### POST /profiles/{profileId}/otp
**Purpose**: Verify OTP code for sensitive profile changes
**Security**: Bearer token
**Input**: OtpVerificationRequest (otpCode)
**Output**: 200 OtpVerificationResponse (verified: boolean)
**Flow**: Validate OTP → Update verification status → Apply changes if verified

#### POST /auth/users/{userId}/documents
**Purpose**: Upload supporting documents for profile changes
**Security**: Bearer token with 'profile.write' scope
**Input**: DocumentUploadRequest (file, documentType) as multipart/form-data
**Output**: 201 DocumentResponse with documentId and timestamp
**Flow**: Validate file → Store in blob storage → Update change request → Return document ID

### System Endpoints

#### GET /health
**Purpose**: Service health check for load balancer and monitoring
**Output**: 200 HealthResponse with service status
**Flow**: Check database connectivity → Validate external dependencies → Return health status

#### GET /metrics
**Purpose**: Prometheus metrics endpoint for monitoring and alerting
**Output**: 200 text/plain with metrics data
**Flow**: Collect service metrics → Format for Prometheus → Return metrics

## Data Models

### Core Entities
- **User**: Domain entity with id (UUID), email, passwordHash, roles (List<Role>)
- **Role**: Entity with name and associated permissions
- **AuthToken**: Value object containing JWT string and expiry timestamp
- **UserProfile**: Complete profile with personal information and co-applicants

### Request/Response DTOs
- **LoginRequest**: username, password
- **TokenResponse**: accessToken, refreshToken, expiresIn
- **UserRegistrationRequest**: email, phone, password with validation constraints
- **UserProfileUpdateRequest**: Optional fields for profile updates
- **ChangeRequest**: fieldName, newValue, reason for sensitive changes
- **OtpVerificationRequest**: otpCode with pattern validation (4-6 digits)

### Error Models
- **ErrorResponse**: code, message, details array for general errors
- **ValidationErrorResponse**: code, message, errors array with field-specific validation failures

## External System Interactions

### OAuth2 Provider Integration
**Protocol**: OAuth2/OpenID Connect over HTTPS
**Operations**: 
- GET /oauth2/authorize - Authorization code flow initiation
- POST /oauth2/token - Token exchange and validation  
- GET /userinfo - User profile information retrieval
**Error Handling**: Circuit breaker pattern with fallback to local authentication
**Retry Logic**: Exponential backoff for transient failures

### Database Operations
**Protocol**: JDBC with connection pooling
**Operations**:
- User CRUD operations with optimistic locking
- Session management with TTL-based cleanup
- Role and permission queries with caching
**Consistency**: ACID transactions for critical operations
**Performance**: Read replicas for query optimization

### Event Bus Integration  
**Protocol**: Apache Kafka with AVRO schema registry
**Events Published**:
- UserRegistered: userId, email, timestamp, source
- UserLoggedIn: userId, sessionId, deviceInfo, timestamp
- ProfileUpdated: userId, changedFields, timestamp, requestId
- SecurityEvent: eventType, userId, details, severity, timestamp
**Delivery Guarantees**: At-least-once delivery with idempotent consumers

## Key Flows

### User Registration Flow
1. Validate registration request (email format, password strength, phone number)
2. Check email/phone uniqueness in database
3. Hash password using bcrypt with salt
4. Create user record with default role assignment
5. Generate email verification token
6. Send verification email via notification service
7. Publish UserRegistered event to Kafka
8. Return success response with user ID

### Secure Login Flow
1. Validate login credentials against database
2. Check account status (active, locked, suspended)
3. Verify password hash using bcrypt
4. Generate JWT access token (15-minute expiry) and refresh token (7-day expiry)
5. Store session information in Redis cache
6. Publish UserLoggedIn event with device information
7. Return token response with expiration details

### Profile Change Workflow
1. Validate profile update request and user permissions
2. Identify sensitive fields requiring additional verification
3. For sensitive changes: Generate OTP and send via SMS/email
4. Store pending change request with expiration
5. User submits OTP for verification
6. Validate OTP and apply approved changes
7. Publish ProfileUpdated event for audit trail
8. Return updated profile information

### Token Validation Flow
1. Extract JWT from Authorization header
2. Validate token signature using public key
3. Check token expiration and issuer claims
4. Verify user session exists in Redis cache
5. Load user roles and permissions from database
6. Evaluate access permissions for requested resource
7. Allow or deny request based on authorization policy

### Multi-Factor Authentication Flow
1. User initiates sensitive operation (profile change, password reset)
2. System generates 6-digit OTP with 5-minute expiration
3. Send OTP via user's preferred channel (SMS/email)
4. Store OTP hash in Redis with attempt counter
5. User submits OTP within time window
6. Validate OTP and check attempt limits (max 3 attempts)
7. Mark operation as verified and proceed with request
8. Publish SecurityEvent for audit compliance