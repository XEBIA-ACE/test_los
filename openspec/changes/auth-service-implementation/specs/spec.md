# Authentication & Authorization Service Specification

## Purpose

The Authentication & Authorization Service provides centralized identity management, secure authentication, and fine-grained authorization for the Digital Lending Platform. This service SHALL implement OAuth2/OpenID Connect standards, manage user profiles and sessions, enforce role-based access control, and maintain comprehensive audit trails for regulatory compliance.

## Technologies and Runtime Stack

### Core Technologies
- **Framework**: Spring Boot 3.x with Spring Security 6.x
- **Authentication**: OAuth2, OpenID Connect, JWT tokens
- **Database**: PostgreSQL with JPA/Hibernate ORM
- **Caching**: Redis for session storage and token caching
- **Messaging**: Apache Kafka for event streaming
- **Identity Provider**: Keycloak or AWS Cognito integration
- **Container Runtime**: Docker on Kubernetes (EKS/AKS/GKE)

### Development Stack
- **Language**: Java 17+ with Spring Boot
- **Build Tool**: Maven or Gradle
- **Testing**: JUnit 5, Testcontainers, WireMock
- **Documentation**: OpenAPI 3.0 specifications
- **Monitoring**: Micrometer with Prometheus metrics

## Service Components

### Presentation Layer
- **AuthController**: Exposes REST API endpoints for authentication operations
  - Handles login, registration, token refresh, and logout requests
  - Validates input parameters and delegates to service layer
  - Returns standardized API responses with proper HTTP status codes

### Business Layer
- **AuthService**: Core authentication and authorization logic
  - Implements user registration and credential validation
  - Generates and validates JWT tokens with appropriate claims
  - Manages user sessions and token lifecycle
  - Coordinates with external OAuth providers

- **SecurityValidator**: Token validation and RBAC enforcement
  - Validates JWT signatures and expiration
  - Implements role-based and attribute-based access control
  - Provides authorization decisions for API endpoints

### Data Access Layer
- **UserRepository**: JPA repository for user entity management
  - Provides CRUD operations for user profiles
  - Implements custom queries for authentication lookups
  - Handles user-role relationship mapping

- **AuthRepository**: Authentication-specific data operations
  - Manages session data and token blacklisting
  - Stores authentication attempts and security events
  - Handles password reset tokens and verification codes

### Integration Layer
- **OAuthClient**: External OAuth provider integration
  - Handles OAuth2 authorization code flow
  - Manages provider-specific token exchange
  - Implements provider discovery and configuration

- **EventPublisher**: Kafka event publishing
  - Publishes authentication events for audit trails
  - Sends user profile change notifications
  - Implements reliable event delivery patterns

### Cross-Cutting Components
- **JwtAuthenticationFilter**: Request interception and token validation
  - Extracts and validates JWT tokens from requests
  - Sets security context for downstream processing
  - Handles token refresh and expiration scenarios

## API Endpoints

### Authentication Endpoints

#### POST /auth/register
**Purpose**: Register new user with email/phone validation
**Input**: UserRegistrationRequest (email, phone, password)
**Output**: UserResponse (userId, email, roles) - 201 Created
**Main Flow**:
1. Validate email format and password strength
2. Check for existing user with same email/phone
3. Hash password using bcrypt with salt
4. Create user entity with default role
5. Send verification email/SMS
6. Return user details without sensitive data

#### POST /auth/login
**Purpose**: Authenticate user and issue JWT tokens
**Input**: LoginRequest (username, password)
**Output**: TokenResponse (accessToken, refreshToken, expiresIn) - 200 OK
**Main Flow**:
1. Validate credentials against stored hash
2. Check account status and verification
3. Generate JWT access token with user claims
4. Generate refresh token and store in database
5. Log successful authentication event
6. Return token pair with expiration details

#### POST /auth/refresh
**Purpose**: Refresh expired access token using refresh token
**Input**: TokenRefreshRequest (refreshToken)
**Output**: TokenResponse (new accessToken, refreshToken, expiresIn) - 200 OK
**Main Flow**:
1. Validate refresh token signature and expiration
2. Verify token exists in database and not revoked
3. Generate new access token with updated claims
4. Optionally rotate refresh token for security
5. Update token usage tracking
6. Return new token pair

### Profile Management Endpoints

#### GET /profiles
**Purpose**: List user and co-applicant profiles with pagination
**Security**: Bearer token with 'profile.read' scope
**Output**: ProfileListResponse with pagination metadata - 200 OK
**Main Flow**:
1. Extract user context from JWT token
2. Apply pagination and filtering parameters
3. Query user profiles with co-applicant relationships
4. Filter results based on user permissions
5. Return paginated profile list

#### GET /profiles/{profileId}
**Purpose**: Retrieve specific user profile details
**Security**: Bearer token with 'profile.read' scope
**Output**: UserProfile with co-applicant details - 200 OK
**Main Flow**:
1. Validate profile access permissions
2. Query profile data including co-applicants
3. Apply field-level security filtering
4. Return complete profile information

#### PUT /profiles/{profileId}
**Purpose**: Update user profile information
**Security**: Bearer token with 'profile.write' scope
**Input**: UserProfileUpdateRequest
**Output**: Updated UserProfile - 200 OK
**Main Flow**:
1. Validate update permissions and field access
2. Identify sensitive fields requiring approval
3. Apply immediate updates for non-sensitive fields
4. Create change requests for sensitive fields
5. Publish profile update events
6. Return updated profile with pending changes

#### POST /profiles/{profileId}/actions/requestChange
**Purpose**: Submit change request for sensitive profile fields
**Security**: Bearer token with 'profile.write' scope
**Input**: ChangeRequest (fieldName, newValue, reason)
**Output**: ChangeRequestResponse (requestId, status) - 202 Accepted
**Main Flow**:
1. Validate field sensitivity and change permissions
2. Create change request with approval workflow
3. Generate OTP for verification if required
4. Send notification to user and approvers
5. Return request tracking information

#### POST /profiles/{profileId}/otp
**Purpose**: Verify OTP for sensitive profile changes
**Security**: Bearer token
**Input**: OtpVerificationRequest (otpCode)
**Output**: OtpVerificationResponse (verified) - 200 OK
**Main Flow**:
1. Validate OTP code against stored value
2. Check OTP expiration and attempt limits
3. Apply pending profile changes if verified
4. Publish verification success/failure events
5. Return verification result

## Data Models

### Core Entities

#### User Entity
- **id**: UUID - Primary key
- **email**: String - Unique email address (required)
- **phone**: String - Phone number with country code
- **passwordHash**: String - Bcrypt hashed password
- **roles**: List<Role> - Associated user roles
- **status**: Enum - ACTIVE, INACTIVE, SUSPENDED, PENDING_VERIFICATION
- **createdAt**: Timestamp - Account creation time
- **lastLoginAt**: Timestamp - Last successful login
- **verificationStatus**: Enum - EMAIL_VERIFIED, PHONE_VERIFIED, KYC_VERIFIED

#### Role Entity
- **name**: String - Role name (CUSTOMER, ADMIN, SUPPORT_AGENT)
- **permissions**: List<String> - Associated permissions/scopes
- **description**: String - Role description

#### UserProfile Entity
- **profileId**: UUID - Primary key
- **userId**: UUID - Foreign key to User
- **firstName**: String - User's first name (required)
- **lastName**: String - User's last name (required)
- **address**: String - Complete address
- **coApplicants**: List<UserProfile> - Linked co-applicant profiles
- **kycStatus**: Enum - KYC verification status
- **documentsUploaded**: List<String> - Document reference IDs

### Request/Response DTOs

#### LoginRequest
- **username**: String - Email or phone number (required)
- **password**: String - User password (required)

#### TokenResponse
- **accessToken**: String - JWT access token (required)
- **refreshToken**: String - Refresh token (required)
- **expiresIn**: Integer - Token expiration in seconds (required)
- **tokenType**: String - Always "Bearer"

#### UserRegistrationRequest
- **email**: String - Valid email format (required)
- **phone**: String - E.164 format phone number
- **password**: String - Minimum 8 characters with complexity rules (required)

#### ChangeRequest
- **fieldName**: String - Profile field to change (required)
- **newValue**: String - New field value (required)
- **reason**: String - Justification for change

### Validation Rules
- Email addresses MUST follow RFC 5322 format
- Phone numbers MUST follow E.164 international format
- Passwords MUST be minimum 8 characters with uppercase, lowercase, number, and special character
- OTP codes MUST be 4-6 digits and expire within 5 minutes
- Profile changes for PII fields MUST require OTP verification

## External System Interactions

### OAuth Provider Integration
**Protocol**: OAuth2/OpenID Connect over HTTPS
**Operations**:
- Authorization code exchange for tokens
- Token introspection and validation
- User info retrieval from provider
- Provider discovery and configuration

**Error Handling**: Circuit breaker pattern with fallback to local authentication
**Retry Logic**: Exponential backoff for transient failures

### KYC Service Integration
**Protocol**: REST API over HTTPS with mutual TLS
**Operations**:
- Identity verification initiation
- Document upload and validation
- KYC status polling and webhooks
- eSign workflow integration

**Error Handling**: Graceful degradation with manual verification fallback
**Retry Logic**: Immediate retry for network errors, delayed retry for rate limits

### Kafka Event Publishing
**Protocol**: Kafka binary protocol with SASL/SSL
**Topics**:
- `auth.events.login` - User authentication events
- `auth.events.profile-change` - Profile modification events
- `auth.events.security` - Security-related events

**Message Format**: Avro schema with event metadata
**Delivery Guarantee**: At-least-once with idempotent consumers

## Key Flows

### User Registration Flow
1. **Input Validation**: Validate email format, password strength, and phone number
2. **Duplicate Check**: Query database for existing users with same email/phone
3. **Password Hashing**: Generate bcrypt hash with random salt
4. **User Creation**: Create user entity with PENDING_VERIFICATION status
5. **Verification Trigger**: Send email/SMS verification code
6. **Event Publishing**: Publish user registration event to Kafka
7. **Response**: Return user details without sensitive information

### Authentication Flow
1. **Credential Validation**: Verify username/password against stored hash
2. **Account Verification**: Check user status and verification requirements
3. **Token Generation**: Create JWT with user claims and appropriate scopes
4. **Session Management**: Store refresh token and update last login timestamp
5. **Security Logging**: Log authentication attempt with IP and device info
6. **Event Publishing**: Publish successful login event
7. **Response**: Return access token, refresh token, and expiration details

### Profile Change Approval Flow
1. **Change Request**: User submits profile change for sensitive field
2. **Sensitivity Check**: Determine if field requires approval workflow
3. **OTP Generation**: Create and send verification code to user
4. **Approval Workflow**: Route request to appropriate approvers if needed
5. **OTP Verification**: Validate user-provided OTP code
6. **Change Application**: Apply approved changes to user profile
7. **Notification**: Send confirmation to user and audit trail
8. **Event Publishing**: Publish profile change completion event

### Token Refresh Flow
1. **Token Validation**: Verify refresh token signature and expiration
2. **Database Lookup**: Check token exists and not revoked
3. **User Status Check**: Verify user account still active
4. **New Token Generation**: Create fresh access token with updated claims
5. **Token Rotation**: Optionally generate new refresh token
6. **Usage Tracking**: Update token usage statistics
7. **Response**: Return new token pair with expiration

### Security Event Flow
1. **Event Detection**: Identify suspicious activity or security violation
2. **Event Classification**: Categorize threat level and required response
3. **Immediate Response**: Apply security measures (account lock, token revocation)
4. **Notification**: Alert user and security team of incident
5. **Audit Logging**: Record detailed security event information
6. **Event Publishing**: Publish security event to monitoring systems
7. **Follow-up**: Initiate investigation or remediation procedures