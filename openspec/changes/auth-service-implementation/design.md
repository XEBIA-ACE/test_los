# Authentication & Authorization Service - Technical Design

## Architecture Overview

The Authentication & Authorization Service follows a hexagonal architecture pattern with clear separation of concerns across presentation, business, data access, and integration layers. The service implements OAuth2/OpenID Connect standards with JWT token management, role-based access control, and event-driven audit logging.

### Design Principles
- **Domain-Driven Design**: Clear bounded context for authentication and authorization
- **Hexagonal Architecture**: Ports and adapters pattern for external integrations
- **Event-Driven Architecture**: Asynchronous event publishing for audit and notifications
- **Security by Design**: Defense in depth with multiple security layers
- **Stateless Design**: JWT tokens enable horizontal scaling without session affinity

## Component Architecture

### Core Components

#### Authentication Controller (Presentation Layer)
- **Responsibility**: HTTP request handling and response formatting
- **Technology**: Spring Boot REST controllers with OpenAPI documentation
- **Key Classes**: `AuthController`, `ProfileController`
- **Security**: Input validation, rate limiting, and CORS configuration

#### Authentication Service (Business Layer)
- **Responsibility**: Core authentication logic and business rules
- **Technology**: Spring Service components with transaction management
- **Key Classes**: `AuthService`, `ProfileService`, `TokenService`
- **Patterns**: Command pattern for operations, Strategy pattern for authentication methods

#### Security Validator (Cross-Cutting)
- **Responsibility**: JWT validation and authorization decisions
- **Technology**: Spring Security filters and method-level security
- **Key Classes**: `JwtAuthenticationFilter`, `RoleBasedAccessControl`
- **Integration**: Custom authentication providers and access decision voters

#### Data Access Layer
- **Responsibility**: Database operations and data persistence
- **Technology**: Spring Data JPA with PostgreSQL
- **Key Classes**: `UserRepository`, `RoleRepository`, `SessionRepository`
- **Patterns**: Repository pattern with custom query methods

#### Integration Layer
- **Responsibility**: External system communication
- **Technology**: Spring WebClient for HTTP, Kafka producers for events
- **Key Classes**: `OAuthClient`, `KycServiceClient`, `EventPublisher`
- **Patterns**: Circuit breaker, retry, and bulkhead patterns

## Data Flow Architecture

### Authentication Data Flow
1. **Request Reception**: API Gateway forwards authenticated requests to service
2. **Token Extraction**: JWT filter extracts and validates bearer tokens
3. **Security Context**: Populate Spring Security context with user details
4. **Business Processing**: Service layer executes authentication logic
5. **Data Persistence**: Repository layer manages database transactions
6. **Event Publishing**: Kafka publisher sends audit events asynchronously
7. **Response Formation**: Controller formats and returns API responses

### Profile Management Data Flow
1. **Profile Request**: User requests profile information or updates
2. **Authorization Check**: Verify user permissions for requested profile
3. **Change Detection**: Identify sensitive fields requiring approval workflow
4. **OTP Generation**: Create verification codes for sensitive changes
5. **Workflow Orchestration**: Route changes through approval processes
6. **Data Validation**: Apply business rules and data integrity checks
7. **Persistence**: Commit approved changes to database
8. **Event Notification**: Publish profile change events to interested services

## API Design

### RESTful API Principles
- **Resource-Based URLs**: `/auth/login`, `/profiles/{id}`, `/profiles/{id}/otp`
- **HTTP Methods**: Proper use of GET, POST, PUT, DELETE for operations
- **Status Codes**: Meaningful HTTP status codes (200, 201, 400, 401, 403, 404, 500)
- **Content Negotiation**: JSON request/response with proper content types
- **Versioning**: API versioning through URL path (`/api/v1/auth`)

### Security Headers
- **Authorization**: Bearer token in Authorization header
- **Content-Type**: application/json for request/response bodies
- **X-Request-ID**: Correlation ID for request tracing
- **X-Rate-Limit**: Rate limiting information in response headers

### Error Handling
- **Standardized Errors**: Consistent error response format with code and message
- **Validation Errors**: Field-level validation errors with specific messages
- **Security Errors**: Generic error messages to prevent information disclosure
- **Logging**: Comprehensive error logging with correlation IDs

## Database Design

### User Management Schema
```sql
-- Users table for authentication
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    password_hash VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING_VERIFICATION',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP,
    verification_status VARCHAR(100)
);

-- Roles and permissions
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    permissions JSONB
);

-- User-role relationships
CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id),
    role_id UUID REFERENCES roles(id),
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, role_id)
);

-- User profiles
CREATE TABLE user_profiles (
    profile_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    address TEXT,
    kyc_status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Session and Token Management
```sql
-- Refresh tokens
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    token_hash VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP
);

-- Authentication events for audit
CREATE TABLE auth_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    event_type VARCHAR(100) NOT NULL,
    ip_address INET,
    user_agent TEXT,
    success BOOLEAN NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Security Implementation

### JWT Token Structure
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "auth-service-key-1"
  },
  "payload": {
    "sub": "user-uuid",
    "iss": "auth-service",
    "aud": ["lending-platform"],
    "exp": 1640995200,
    "iat": 1640991600,
    "roles": ["CUSTOMER"],
    "scopes": ["profile.read", "profile.write"],
    "email": "user@example.com"
  }
}
```

### Password Security
- **Hashing**: bcrypt with cost factor 12 and random salt
- **Validation**: Minimum 8 characters with complexity requirements
- **Reset**: Secure token-based password reset with expiration
- **History**: Prevent reuse of last 5 passwords

### OAuth2 Integration
- **Authorization Code Flow**: Standard OAuth2 flow with PKCE
- **Token Exchange**: Secure token exchange with external providers
- **Scope Management**: Fine-grained scope definitions for API access
- **Provider Discovery**: Dynamic configuration from OpenID Connect discovery

## Event-Driven Architecture

### Event Schema Design
```json
{
  "eventId": "uuid",
  "eventType": "USER_AUTHENTICATED",
  "timestamp": "2024-01-15T10:30:00Z",
  "source": "auth-service",
  "version": "1.0",
  "data": {
    "userId": "user-uuid",
    "email": "user@example.com",
    "ipAddress": "192.168.1.1",
    "userAgent": "Mozilla/5.0...",
    "success": true
  },
  "metadata": {
    "correlationId": "request-uuid",
    "causationId": "parent-event-uuid"
  }
}
```

### Event Types
- **USER_REGISTERED**: New user account creation
- **USER_AUTHENTICATED**: Successful login attempts
- **AUTHENTICATION_FAILED**: Failed login attempts
- **PROFILE_UPDATED**: User profile modifications
- **PASSWORD_CHANGED**: Password update events
- **ACCOUNT_LOCKED**: Security-related account locks
- **TOKEN_REFRESHED**: Token refresh operations

## Integration Patterns

### Circuit Breaker Implementation
```java
@Component
public class OAuthClientCircuitBreaker {
    
    @CircuitBreaker(name = "oauth-provider", fallbackMethod = "fallbackAuthentication")
    @Retry(name = "oauth-provider")
    @TimeLimiter(name = "oauth-provider")
    public CompletableFuture<TokenResponse> exchangeToken(String authCode) {
        return oauthClient.exchangeAuthorizationCode(authCode);
    }
    
    public CompletableFuture<TokenResponse> fallbackAuthentication(String authCode, Exception ex) {
        // Fallback to local authentication or cached tokens
        return localAuthenticationService.authenticate(authCode);
    }
}
```

### Kafka Event Publishing
```java
@Service
public class AuthEventPublisher {
    
    @Autowired
    private KafkaTemplate<String, AuthEvent> kafkaTemplate;
    
    @Async
    public void publishAuthenticationEvent(AuthEvent event) {
        try {
            kafkaTemplate.send("auth.events.authentication", event.getUserId(), event)
                .addCallback(
                    result -> log.info("Event published successfully: {}", event.getEventId()),
                    failure -> log.error("Failed to publish event: {}", event.getEventId(), failure)
                );
        } catch (Exception e) {
            log.error("Error publishing authentication event", e);
            // Store in dead letter queue or retry mechanism
        }
    }
}
```

## Monitoring and Observability

### Metrics Collection
- **Authentication Metrics**: Login success/failure rates, token generation times
- **Performance Metrics**: API response times, database query performance
- **Security Metrics**: Failed authentication attempts, suspicious activity detection
- **Business Metrics**: User registration rates, profile completion statistics

### Health Check Implementation
```java
@Component
public class AuthServiceHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        return Health.up()
            .withDetail("database", checkDatabaseConnection())
            .withDetail("oauth-provider", checkOAuthProvider())
            .withDetail("kafka", checkKafkaConnection())
            .withDetail("redis", checkRedisConnection())
            .build();
    }
}
```

### Distributed Tracing
- **Trace Context**: Propagate trace IDs across service boundaries
- **Span Creation**: Create spans for major operations (authentication, token validation)
- **Correlation**: Link related operations across multiple services
- **Performance Analysis**: Identify bottlenecks and optimization opportunities

## Deployment Considerations

### Container Configuration
```dockerfile
FROM openjdk:17-jre-slim
COPY target/auth-service.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Kubernetes Deployment
- **Resource Limits**: CPU and memory limits for predictable performance
- **Health Probes**: Liveness and readiness probes for container orchestration
- **ConfigMaps**: External configuration for environment-specific settings
- **Secrets**: Secure storage of database credentials and JWT signing keys
- **Service Mesh**: Istio or Linkerd for traffic management and security

### Scaling Strategy
- **Horizontal Scaling**: Stateless design enables multiple pod replicas
- **Auto-scaling**: HPA based on CPU, memory, and custom metrics
- **Database Scaling**: Read replicas for query performance
- **Cache Scaling**: Redis cluster for session storage scaling