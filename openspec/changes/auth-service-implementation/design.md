# Authentication & Authorization Service - Technical Design

## Architecture Overview

The Authentication & Authorization Service implements a layered hexagonal architecture with clear separation of concerns between presentation, business logic, data access, and integration layers. The service follows Domain-Driven Design principles with bounded contexts for authentication, authorization, and user management.

### Design Principles
- **Hexagonal Architecture**: Clean separation between core business logic and external adapters
- **Dependency Inversion**: Business logic depends on abstractions, not concrete implementations  
- **Single Responsibility**: Each component has a focused, well-defined purpose
- **Stateless Design**: Service instances are stateless to enable horizontal scaling
- **Event-Driven**: Asynchronous communication through domain events for loose coupling
- **Security by Design**: Security controls embedded at every architectural layer

## Technical Approach

### Authentication Strategy
The service implements OAuth2/OpenID Connect with JWT tokens for stateless authentication. External identity providers (Keycloak/AWS Cognito) handle primary authentication, while the service manages session state and authorization policies. JWT tokens contain user identity and role claims, enabling distributed authorization without centralized lookups.

### Authorization Model
Role-Based Access Control (RBAC) with hierarchical roles and fine-grained permissions. Roles are assigned to users and contain permission sets that map to API operations. Authorization decisions are made at the API Gateway level using JWT claims and enforced at the service level through Spring Security annotations.

### Data Management
Polyglot persistence approach with PostgreSQL for transactional user data, Redis for session caching and token blacklists, and Kafka for event streaming. The service implements CQRS patterns for read/write separation and maintains eventual consistency through event sourcing.

### Security Implementation
Multi-layered security with OAuth2 authentication, JWT token validation, RBAC authorization, input validation, SQL injection prevention, and comprehensive audit logging. Sensitive operations require multi-factor authentication via OTP verification.

## Data Flow Architecture

### Authentication Flow
1. **Client Request**: Mobile app initiates login with credentials
2. **API Gateway**: Validates request format and rate limits
3. **Auth Controller**: Receives login request and delegates to AuthService
4. **Auth Service**: Validates credentials against database or external OAuth provider
5. **Token Generation**: Creates JWT access/refresh tokens with user claims
6. **Session Storage**: Stores session metadata in Redis for quick validation
7. **Event Publishing**: Publishes authentication event to Kafka for audit
8. **Response**: Returns tokens to client via secure HTTPS

### Authorization Flow  
1. **Protected Request**: Client sends API request with JWT token
2. **JWT Filter**: Extracts and validates token signature and expiration
3. **User Context**: Loads user roles and permissions from token claims
4. **Permission Check**: Evaluates required permissions against user roles
5. **Access Decision**: Allows or denies request based on authorization policy
6. **Audit Logging**: Records access attempts for security monitoring

### Profile Management Flow
1. **Profile Update**: User submits profile change request
2. **Validation**: Service validates input and checks for sensitive fields
3. **OTP Generation**: For sensitive changes, generates and sends OTP
4. **Change Request**: Stores pending change with expiration timestamp
5. **OTP Verification**: User submits OTP for validation
6. **Change Application**: Applies verified changes to user profile
7. **Event Publishing**: Publishes profile update event for downstream services
8. **Notification**: Triggers confirmation notification to user

## API Design

### RESTful Principles
The service exposes RESTful APIs following OpenAPI 3.0 specifications with consistent resource naming, HTTP status codes, and error response formats. All endpoints support JSON content type with proper CORS headers for web client integration.

### Security Headers
All API responses include security headers (HSTS, CSP, X-Frame-Options) and implement proper authentication/authorization mechanisms. Sensitive endpoints require additional security measures like OTP verification.

### Error Handling
Standardized error responses with consistent structure including error codes, messages, and detailed field-level validation errors. HTTP status codes follow REST conventions (200, 201, 400, 401, 403, 404, 500).

### Pagination and Filtering
List endpoints support pagination with page/size parameters and filtering with search, sortBy, and order parameters. Responses include pagination metadata for client navigation.

## Component Design

### AuthController (Presentation Layer)
- **Responsibilities**: HTTP request/response handling, input validation, security headers
- **Dependencies**: AuthService for business logic delegation
- **Implementation**: Spring Boot REST controller with OpenAPI annotations
- **Error Handling**: Global exception handler for consistent error responses

### AuthService (Business Layer)  
- **Responsibilities**: Core authentication logic, token management, user operations
- **Dependencies**: UserRepository for data access, OAuthClient for external auth
- **Implementation**: Spring service with transactional boundaries
- **Security**: Method-level security annotations and input sanitization

### UserRepository (Data Access Layer)
- **Responsibilities**: Database operations for user entities
- **Implementation**: JPA repository with custom queries and caching
- **Performance**: Connection pooling and query optimization
- **Consistency**: Transaction management with optimistic locking

### OAuthClient (Integration Layer)
- **Responsibilities**: External OAuth provider integration
- **Implementation**: Spring Security OAuth2 client with circuit breaker
- **Resilience**: Retry logic and fallback mechanisms
- **Configuration**: Provider-specific settings and endpoint mappings

### EventPublisher (Integration Layer)
- **Responsibilities**: Kafka event publishing for audit and notifications
- **Implementation**: Spring Kafka producer with schema registry
- **Reliability**: Guaranteed delivery with retry and dead letter queues
- **Schema**: AVRO schemas for event structure and evolution

## File and Module Structure

### Core Domain Modules
- `com.lending.auth.domain.model`: User, Role, Permission entities
- `com.lending.auth.domain.repository`: Repository interfaces
- `com.lending.auth.domain.service`: Domain services and business logic
- `com.lending.auth.domain.event`: Domain events and event handlers

### Application Layer
- `com.lending.auth.application.service`: Application services and use cases
- `com.lending.auth.application.dto`: Request/response DTOs
- `com.lending.auth.application.mapper`: Entity-DTO mapping utilities

### Infrastructure Layer
- `com.lending.auth.infrastructure.persistence`: JPA repositories and entities
- `com.lending.auth.infrastructure.oauth`: OAuth client implementations
- `com.lending.auth.infrastructure.messaging`: Kafka producers and consumers
- `com.lending.auth.infrastructure.cache`: Redis cache implementations

### Presentation Layer
- `com.lending.auth.presentation.controller`: REST controllers
- `com.lending.auth.presentation.filter`: Security filters and interceptors
- `com.lending.auth.presentation.config`: Spring configuration classes

### Configuration Files
- `application.yml`: Service configuration and profiles
- `bootstrap.yml`: Bootstrap configuration for service discovery
- `logback-spring.xml`: Logging configuration with structured output
- `Dockerfile`: Container image definition with security hardening
- `k8s/`: Kubernetes deployment manifests and service definitions

## Security Architecture

### Authentication Security
- OAuth2/OIDC integration with external identity providers
- JWT token validation with signature verification
- Secure password hashing using bcrypt with salt
- Session management with Redis-based token storage
- Multi-factor authentication via OTP verification

### Authorization Security  
- Role-based access control with hierarchical permissions
- Method-level security annotations in Spring Security
- API Gateway integration for centralized policy enforcement
- Dynamic permission evaluation based on user context

### Data Security
- Encryption at rest for sensitive user data
- TLS 1.3 for all external communications
- Input validation and SQL injection prevention
- Audit logging for all security-relevant operations
- Secure document upload with virus scanning

### Operational Security
- Container security with minimal base images
- Secrets management via Kubernetes secrets or HashiCorp Vault
- Network security with VPC isolation and security groups
- Monitoring and alerting for security events and anomalies