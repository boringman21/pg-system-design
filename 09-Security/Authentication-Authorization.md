# Authentication & Authorization

**Tags**: #security #auth #jwt #oauth
**Date**: 2024-01-01

## 📝 Overview

Authentication (Who are you?) và Authorization (What can you do?) là hai security concepts quan trọng nhất trong system design. Hiểu rõ mechanisms và best practices là crucial cho secure systems.

## 🔑 Authentication vs Authorization

### **Authentication (AuthN)**
- **Purpose**: Verify user identity
- **Question**: "Who are you?"
- **Methods**: Password, biometrics, tokens
- **Result**: User identity confirmed

### **Authorization (AuthZ)**  
- **Purpose**: Control access to resources
- **Question**: "What can you do?"
- **Methods**: Roles, permissions, policies
- **Result**: Access granted/denied

## 🛡️ Authentication Methods

### **Username/Password**
```python
import bcrypt

class PasswordAuth:
    def hash_password(self, password):
        salt = bcrypt.gensalt()
        return bcrypt.hashpw(password.encode('utf-8'), salt)
    
    def verify_password(self, password, hashed):
        return bcrypt.checkpw(password.encode('utf-8'), hashed)
```

### **JWT Tokens**
```python
import jwt
from datetime import datetime, timedelta

class JWTAuth:
    def __init__(self, secret_key):
        self.secret_key = secret_key
    
    def generate_token(self, user_id):
        payload = {
            'user_id': user_id,
            'exp': datetime.utcnow() + timedelta(hours=24)
        }
        return jwt.encode(payload, self.secret_key, algorithm='HS256')
    
    def verify_token(self, token):
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=['HS256'])
            return payload['user_id']
        except jwt.ExpiredSignatureError:
            raise AuthenticationError("Token expired")
```

### **OAuth 2.0**
```python
# OAuth flow example
class OAuthProvider:
    def authorize(self, client_id, redirect_uri, scope):
        # Redirect user to consent page
        auth_url = f"/oauth/authorize?client_id={client_id}&redirect_uri={redirect_uri}&scope={scope}"
        return auth_url
    
    def exchange_code_for_token(self, code, client_id, client_secret):
        # Exchange authorization code for access token
        if self.verify_client(client_id, client_secret):
            access_token = self.generate_access_token()
            return {'access_token': access_token, 'token_type': 'Bearer'}
```

## 🎭 Authorization Models

### **Role-Based Access Control (RBAC)**
```python
class RBACSystem:
    def __init__(self):
        self.roles = {
            'admin': ['read', 'write', 'delete'],
            'editor': ['read', 'write'],
            'viewer': ['read']
        }
        self.user_roles = {}
    
    def assign_role(self, user_id, role):
        self.user_roles[user_id] = role
    
    def check_permission(self, user_id, action):
        user_role = self.user_roles.get(user_id)
        if user_role and action in self.roles.get(user_role, []):
            return True
        return False
```

### **Attribute-Based Access Control (ABAC)**
```python
class ABACSystem:
    def check_access(self, subject, action, resource, environment):
        # Subject: user attributes (role, department)
        # Action: read, write, delete
        # Resource: file, database, API
        # Environment: time, location, device
        
        if action == 'read' and subject['role'] == 'employee':
            return True
        
        if action == 'write' and subject['department'] == resource['owner_dept']:
            return True
        
        if subject['role'] == 'admin':
            return True
        
        return False
```

## 🔐 Security Best Practices

### **Password Security**
```python
# Password requirements
def validate_password(password):
    checks = [
        len(password) >= 8,           # Minimum length
        re.search(r'[A-Z]', password), # Uppercase letter
        re.search(r'[a-z]', password), # Lowercase letter  
        re.search(r'\d', password),    # Digit
        re.search(r'[!@#$%^&*]', password)  # Special character
    ]
    return all(checks)

# Rate limiting for auth
class AuthRateLimit:
    def __init__(self):
        self.attempts = {}
    
    def check_rate_limit(self, identifier):
        current_time = time.time()
        if identifier in self.attempts:
            attempts = [t for t in self.attempts[identifier] if current_time - t < 3600]
            if len(attempts) >= 5:  # Max 5 attempts per hour
                return False
            self.attempts[identifier] = attempts + [current_time]
        else:
            self.attempts[identifier] = [current_time]
        return True
```

### **Session Management**
```python
class SessionManager:
    def __init__(self):
        self.sessions = {}
        self.session_timeout = 1800  # 30 minutes
    
    def create_session(self, user_id):
        session_id = secrets.token_urlsafe(32)
        self.sessions[session_id] = {
            'user_id': user_id,
            'created_at': time.time(),
            'last_access': time.time()
        }
        return session_id
    
    def validate_session(self, session_id):
        session = self.sessions.get(session_id)
        if not session:
            return None
        
        # Check timeout
        if time.time() - session['last_access'] > self.session_timeout:
            del self.sessions[session_id]
            return None
        
        # Update last access
        session['last_access'] = time.time()
        return session['user_id']
```

## 🎯 Implementation Patterns

### **API Authentication Middleware**
```python
def auth_required(f):
    def wrapper(*args, **kwargs):
        token = request.headers.get('Authorization')
        if not token:
            return {'error': 'No token provided'}, 401
        
        try:
            user_id = jwt_auth.verify_token(token.replace('Bearer ', ''))
            request.user_id = user_id
            return f(*args, **kwargs)
        except AuthenticationError:
            return {'error': 'Invalid token'}, 401
    
    return wrapper

@auth_required
def get_user_profile():
    user_id = request.user_id
    return get_user_data(user_id)
```

### **Multi-Factor Authentication**
```python
class MFASystem:
    def __init__(self):
        self.pending_auth = {}
    
    def initiate_mfa(self, user_id, password):
        # First factor: password
        if not self.verify_password(user_id, password):
            return False
        
        # Generate OTP for second factor
        otp = self.generate_otp()
        self.send_sms(user_id, otp)
        
        # Store pending authentication
        temp_token = secrets.token_urlsafe(16)
        self.pending_auth[temp_token] = {
            'user_id': user_id,
            'otp': otp,
            'expires': time.time() + 300  # 5 minutes
        }
        
        return temp_token
    
    def complete_mfa(self, temp_token, provided_otp):
        pending = self.pending_auth.get(temp_token)
        if not pending or time.time() > pending['expires']:
            return None
        
        if provided_otp == pending['otp']:
            del self.pending_auth[temp_token]
            return self.generate_jwt_token(pending['user_id'])
        
        return None
```

## 🔗 Liên kết và Tham khảo

### **Related Topics**
- [[02-System-Components/API-Gateway/API-Gateway-Pattern|API Gateway]] - Centralized authentication
- [[07-Microservices/Microservices-Architecture|Microservices]] - Distributed auth
- [[12-Practice-Problems/Rate-Limiter-Design|Rate Limiting]] - Prevent abuse
- [[Encryption]] - Data protection

### **Security Standards**
- OAuth 2.0 và OpenID Connect
- SAML 2.0 for enterprise SSO
- JWT best practices (RFC 7519)
- NIST Cybersecurity Framework

### **Tools và Libraries**
- **Identity Providers**: Auth0, Okta, AWS Cognito, Azure AD
- **JWT Libraries**: jsonwebtoken (Node.js), PyJWT (Python), jose4j (Java)
- **OAuth Libraries**: passport.js, spring-security-oauth2
- **Security Headers**: helmet.js, secure headers middleware

---

*Security là foundation của mọi system. Implement defense in depth, validate all inputs, và regularly audit security practices.*

## 📚 Further Reading

- OAuth 2.0 RFC specification
- JWT best practices
- NIST authentication guidelines
- OWASP authentication cheat sheet

---

**Key Takeaway**: Strong authentication và granular authorization là foundation của secure systems. Implement defense in depth với multiple security layers. 