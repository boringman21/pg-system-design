# OAuth 2.0 & JWT Implementation Guide

**Tags**: #security #oauth #jwt #authentication #authorization
**Date**: 2024-01-01

## 📝 Overview

Complete guide về OAuth 2.0 và JWT implementation cho modern applications, bao gồm security best practices, code examples, và common pitfalls.

## 🎯 OAuth 2.0 Fundamentals

### **Core Concepts**
- **Resource Owner**: User sở hữu protected resources
- **Client**: Application muốn access protected resources
- **Authorization Server**: Server xác thực và cấp access tokens
- **Resource Server**: Server chứa protected resources

### **OAuth 2.0 Flow Types**
```python
# Authorization Code Flow (Most secure)
# 1. Client redirects user to authorization server
# 2. User authenticates and authorizes
# 3. Authorization server redirects back with authorization code
# 4. Client exchanges code for access token
# 5. Client uses access token to access resources

# Implicit Flow (Legacy, not recommended)
# 1. Client redirects user to authorization server
# 2. User authenticates and authorizes
# 3. Authorization server redirects back with access token directly

# Client Credentials Flow (Server-to-server)
# 1. Client authenticates directly with authorization server
# 2. Authorization server returns access token
# 3. Client uses access token to access resources

# Resource Owner Password Credentials Flow (Legacy)
# 1. Client collects username/password from user
# 2. Client exchanges credentials for access token
# 3. Client uses access token to access resources
```

## 🔐 JWT (JSON Web Token) Implementation

### **JWT Structure**
```python
import jwt
import json
from datetime import datetime, timedelta
from typing import Dict, Optional

class JWTService:
    def __init__(self, secret_key: str, algorithm: str = 'HS256'):
        self.secret_key = secret_key
        self.algorithm = algorithm
    
    def generate_access_token(self, user_id: str, permissions: list = None) -> str:
        """Generate JWT access token with user claims"""
        payload = {
            'user_id': user_id,
            'permissions': permissions or [],
            'token_type': 'access',
            'iat': datetime.utcnow(),
            'exp': datetime.utcnow() + timedelta(hours=1),
            'iss': 'your-app-name',
            'aud': 'your-api'
        }
        
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def generate_refresh_token(self, user_id: str) -> str:
        """Generate JWT refresh token for token renewal"""
        payload = {
            'user_id': user_id,
            'token_type': 'refresh',
            'iat': datetime.utcnow(),
            'exp': datetime.utcnow() + timedelta(days=30),
            'iss': 'your-app-name',
            'aud': 'your-api'
        }
        
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def verify_token(self, token: str) -> Dict:
        """Verify and decode JWT token"""
        try:
            payload = jwt.decode(
                token, 
                self.secret_key, 
                algorithms=[self.algorithm],
                audience='your-api',
                issuer='your-app-name'
            )
            return payload
        except jwt.ExpiredSignatureError:
            raise AuthenticationError("Token has expired")
        except jwt.InvalidTokenError:
            raise AuthenticationError("Invalid token")
    
    def refresh_access_token(self, refresh_token: str) -> str:
        """Generate new access token using refresh token"""
        payload = self.verify_token(refresh_token)
        
        if payload.get('token_type') != 'refresh':
            raise AuthenticationError("Invalid refresh token")
        
        # Generate new access token
        return self.generate_access_token(
            user_id=payload['user_id'],
            permissions=payload.get('permissions', [])
        )
```

### **Advanced JWT Features**
```python
import json
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa

class AdvancedJWTService:
    def __init__(self, private_key_path: str, public_key_path: str):
        # Load RSA keys for RS256 algorithm
        with open(private_key_path, 'rb') as f:
            self.private_key = serialization.load_pem_private_key(
                f.read(), password=None
            )
        
        with open(public_key_path, 'rb') as f:
            self.public_key = serialization.load_pem_public_key(f.read())
    
    def generate_signed_token(self, user_id: str, scope: str, custom_claims: Dict = None) -> str:
        """Generate RS256 signed JWT with custom claims"""
        payload = {
            'sub': user_id,  # Subject
            'scope': scope,
            'iat': datetime.utcnow(),
            'exp': datetime.utcnow() + timedelta(hours=1),
            'iss': 'https://your-domain.com',
            'aud': 'your-api-audience',
            'jti': str(uuid.uuid4()),  # JWT ID for revocation
        }
        
        if custom_claims:
            payload.update(custom_claims)
        
        return jwt.encode(payload, self.private_key, algorithm='RS256')
    
    def verify_signed_token(self, token: str) -> Dict:
        """Verify RS256 signed JWT"""
        try:
            payload = jwt.decode(
                token,
                self.public_key,
                algorithms=['RS256'],
                audience='your-api-audience',
                issuer='https://your-domain.com'
            )
            
            # Check if token is revoked
            if self.is_token_revoked(payload.get('jti')):
                raise AuthenticationError("Token has been revoked")
            
            return payload
        except jwt.ExpiredSignatureError:
            raise AuthenticationError("Token has expired")
        except jwt.InvalidTokenError:
            raise AuthenticationError("Invalid token")
    
    def is_token_revoked(self, jti: str) -> bool:
        """Check if token is in revocation list"""
        # In production, check against Redis or database
        return jti in self.revoked_tokens
```

## 🔄 Complete OAuth 2.0 Implementation

### **Authorization Server**
```python
from flask import Flask, request, redirect, jsonify
import secrets
import hashlib
from urllib.parse import urlencode

class AuthorizationServer:
    def __init__(self, jwt_service, user_service):
        self.jwt_service = jwt_service
        self.user_service = user_service
        self.authorization_codes = {}  # In production: use Redis
        self.clients = {
            'client_id_123': {
                'client_secret': 'client_secret_456',
                'redirect_uris': ['https://client-app.com/callback'],
                'scopes': ['read', 'write']
            }
        }
    
    def authorize_endpoint(self, client_id: str, redirect_uri: str, 
                          response_type: str, scope: str, state: str = None):
        """OAuth 2.0 authorization endpoint"""
        # Validate client
        client = self.clients.get(client_id)
        if not client:
            return self.error_response('invalid_client')
        
        # Validate redirect URI
        if redirect_uri not in client['redirect_uris']:
            return self.error_response('invalid_redirect_uri')
        
        # Validate response type
        if response_type != 'code':
            return self.error_response('unsupported_response_type')
        
        # Validate scope
        requested_scopes = scope.split(' ')
        if not all(s in client['scopes'] for s in requested_scopes):
            return self.error_response('invalid_scope')
        
        # In real implementation, show consent page
        # For demo, auto-approve
        user_id = self.get_authenticated_user()  # From session
        if not user_id:
            return redirect(f'/login?{urlencode(request.args)}')
        
        # Generate authorization code
        auth_code = secrets.token_urlsafe(32)
        self.authorization_codes[auth_code] = {
            'client_id': client_id,
            'user_id': user_id,
            'scope': scope,
            'redirect_uri': redirect_uri,
            'expires_at': datetime.utcnow() + timedelta(minutes=10)
        }
        
        # Redirect back to client
        callback_params = {
            'code': auth_code,
            'state': state
        }
        return redirect(f"{redirect_uri}?{urlencode(callback_params)}")
    
    def token_endpoint(self, grant_type: str, code: str = None, 
                      client_id: str = None, client_secret: str = None,
                      redirect_uri: str = None, refresh_token: str = None):
        """OAuth 2.0 token endpoint"""
        # Authenticate client
        client = self.authenticate_client(client_id, client_secret)
        if not client:
            return self.error_response('invalid_client')
        
        if grant_type == 'authorization_code':
            return self.handle_authorization_code_grant(
                code, client_id, redirect_uri
            )
        elif grant_type == 'refresh_token':
            return self.handle_refresh_token_grant(refresh_token, client_id)
        elif grant_type == 'client_credentials':
            return self.handle_client_credentials_grant(client_id)
        else:
            return self.error_response('unsupported_grant_type')
    
    def handle_authorization_code_grant(self, code: str, client_id: str, 
                                      redirect_uri: str):
        """Handle authorization code grant"""
        # Validate authorization code
        auth_data = self.authorization_codes.get(code)
        if not auth_data:
            return self.error_response('invalid_grant')
        
        # Check expiration
        if datetime.utcnow() > auth_data['expires_at']:
            del self.authorization_codes[code]
            return self.error_response('invalid_grant')
        
        # Validate client and redirect URI
        if (auth_data['client_id'] != client_id or 
            auth_data['redirect_uri'] != redirect_uri):
            return self.error_response('invalid_grant')
        
        # Generate tokens
        access_token = self.jwt_service.generate_access_token(
            user_id=auth_data['user_id'],
            permissions=auth_data['scope'].split(' ')
        )
        refresh_token = self.jwt_service.generate_refresh_token(
            user_id=auth_data['user_id']
        )
        
        # Clean up authorization code
        del self.authorization_codes[code]
        
        return {
            'access_token': access_token,
            'refresh_token': refresh_token,
            'token_type': 'Bearer',
            'expires_in': 3600,
            'scope': auth_data['scope']
        }
    
    def handle_refresh_token_grant(self, refresh_token: str, client_id: str):
        """Handle refresh token grant"""
        try:
            # Verify refresh token
            payload = self.jwt_service.verify_token(refresh_token)
            
            if payload.get('token_type') != 'refresh':
                return self.error_response('invalid_grant')
            
            # Generate new access token
            new_access_token = self.jwt_service.generate_access_token(
                user_id=payload['user_id'],
                permissions=payload.get('permissions', [])
            )
            
            return {
                'access_token': new_access_token,
                'token_type': 'Bearer',
                'expires_in': 3600
            }
        except AuthenticationError:
            return self.error_response('invalid_grant')
    
    def handle_client_credentials_grant(self, client_id: str):
        """Handle client credentials grant"""
        client = self.clients.get(client_id)
        
        # Generate access token for client
        access_token = self.jwt_service.generate_access_token(
            user_id=client_id,  # Use client_id as user_id
            permissions=client['scopes']
        )
        
        return {
            'access_token': access_token,
            'token_type': 'Bearer',
            'expires_in': 3600,
            'scope': ' '.join(client['scopes'])
        }
    
    def authenticate_client(self, client_id: str, client_secret: str):
        """Authenticate OAuth client"""
        client = self.clients.get(client_id)
        if not client:
            return None
        
        # Verify client secret
        if client['client_secret'] != client_secret:
            return None
        
        return client
    
    def error_response(self, error_code: str):
        """Return OAuth error response"""
        return {
            'error': error_code,
            'error_description': self.get_error_description(error_code)
        }
```

### **Resource Server**
```python
from functools import wraps
from flask import request, jsonify

class ResourceServer:
    def __init__(self, jwt_service):
        self.jwt_service = jwt_service
    
    def require_oauth(self, scope: str = None):
        """Decorator to require OAuth token"""
        def decorator(f):
            @wraps(f)
            def decorated_function(*args, **kwargs):
                # Extract token from Authorization header
                auth_header = request.headers.get('Authorization')
                if not auth_header:
                    return jsonify({'error': 'missing_token'}), 401
                
                try:
                    token_type, token = auth_header.split(' ')
                    if token_type.lower() != 'bearer':
                        return jsonify({'error': 'invalid_token_type'}), 401
                except ValueError:
                    return jsonify({'error': 'invalid_authorization_header'}), 401
                
                # Verify token
                try:
                    payload = self.jwt_service.verify_token(token)
                except AuthenticationError as e:
                    return jsonify({'error': 'invalid_token', 'message': str(e)}), 401
                
                # Check scope if required
                if scope:
                    token_scopes = payload.get('permissions', [])
                    if scope not in token_scopes:
                        return jsonify({'error': 'insufficient_scope'}), 403
                
                # Add user info to request context
                request.oauth_user_id = payload['user_id']
                request.oauth_scopes = payload.get('permissions', [])
                
                return f(*args, **kwargs)
            return decorated_function
        return decorator
    
    def get_current_user(self):
        """Get current authenticated user"""
        return getattr(request, 'oauth_user_id', None)
    
    def has_scope(self, scope: str):
        """Check if current token has required scope"""
        token_scopes = getattr(request, 'oauth_scopes', [])
        return scope in token_scopes
```

## 🔧 PKCE (Proof Key for Code Exchange)

### **Enhanced Security for Public Clients**
```python
import base64
import hashlib
import secrets

class PKCEService:
    def generate_code_verifier(self) -> str:
        """Generate code verifier for PKCE"""
        # Generate 128 random bytes and base64url encode
        return base64.urlsafe_b64encode(secrets.token_bytes(96)).decode('utf-8').rstrip('=')
    
    def generate_code_challenge(self, code_verifier: str) -> str:
        """Generate code challenge from verifier"""
        # SHA256 hash of code verifier
        digest = hashlib.sha256(code_verifier.encode('utf-8')).digest()
        # Base64url encode without padding
        return base64.urlsafe_b64encode(digest).decode('utf-8').rstrip('=')
    
    def verify_code_challenge(self, code_verifier: str, code_challenge: str) -> bool:
        """Verify code challenge against verifier"""
        expected_challenge = self.generate_code_challenge(code_verifier)
        return expected_challenge == code_challenge

class PKCEAuthorizationServer(AuthorizationServer):
    def __init__(self, jwt_service, user_service):
        super().__init__(jwt_service, user_service)
        self.pkce_service = PKCEService()
    
    def authorize_endpoint_with_pkce(self, client_id: str, redirect_uri: str,
                                   response_type: str, scope: str, state: str = None,
                                   code_challenge: str = None, code_challenge_method: str = None):
        """Authorization endpoint with PKCE support"""
        # Validate PKCE parameters for public clients
        client = self.clients.get(client_id)
        if client and client.get('client_type') == 'public':
            if not code_challenge or code_challenge_method != 'S256':
                return self.error_response('invalid_request')
        
        # Store code challenge with authorization code
        user_id = self.get_authenticated_user()
        if not user_id:
            return redirect(f'/login?{urlencode(request.args)}')
        
        auth_code = secrets.token_urlsafe(32)
        self.authorization_codes[auth_code] = {
            'client_id': client_id,
            'user_id': user_id,
            'scope': scope,
            'redirect_uri': redirect_uri,
            'code_challenge': code_challenge,
            'expires_at': datetime.utcnow() + timedelta(minutes=10)
        }
        
        callback_params = {
            'code': auth_code,
            'state': state
        }
        return redirect(f"{redirect_uri}?{urlencode(callback_params)}")
    
    def token_endpoint_with_pkce(self, grant_type: str, code: str = None,
                                client_id: str = None, client_secret: str = None,
                                redirect_uri: str = None, code_verifier: str = None):
        """Token endpoint with PKCE verification"""
        if grant_type == 'authorization_code':
            # Validate authorization code
            auth_data = self.authorization_codes.get(code)
            if not auth_data:
                return self.error_response('invalid_grant')
            
            # Verify PKCE for public clients
            client = self.clients.get(client_id)
            if client and client.get('client_type') == 'public':
                if not code_verifier:
                    return self.error_response('invalid_request')
                
                code_challenge = auth_data.get('code_challenge')
                if not code_challenge:
                    return self.error_response('invalid_request')
                
                if not self.pkce_service.verify_code_challenge(code_verifier, code_challenge):
                    return self.error_response('invalid_grant')
            
            # Continue with normal authorization code flow
            return self.handle_authorization_code_grant(code, client_id, redirect_uri)
        
        return super().token_endpoint(grant_type, code, client_id, client_secret, redirect_uri)
```

## 🛡️ Security Best Practices

### **Token Security**
```python
class SecureTokenService:
    def __init__(self, secret_key: str):
        self.secret_key = secret_key
        self.token_blacklist = set()  # In production: use Redis
    
    def generate_secure_token(self, user_id: str, additional_claims: dict = None):
        """Generate secure JWT with additional security measures"""
        # Add security claims
        payload = {
            'user_id': user_id,
            'jti': str(uuid.uuid4()),  # JWT ID for revocation
            'iat': datetime.utcnow(),
            'exp': datetime.utcnow() + timedelta(hours=1),
            'iss': 'your-secure-app',
            'aud': 'your-api',
            'nbf': datetime.utcnow(),  # Not before
        }
        
        if additional_claims:
            payload.update(additional_claims)
        
        return jwt.encode(payload, self.secret_key, algorithm='HS256')
    
    def revoke_token(self, jti: str):
        """Revoke a token by adding to blacklist"""
        self.token_blacklist.add(jti)
    
    def is_token_revoked(self, jti: str) -> bool:
        """Check if token is revoked"""
        return jti in self.token_blacklist
    
    def verify_secure_token(self, token: str) -> dict:
        """Verify token with additional security checks"""
        try:
            payload = jwt.decode(
                token,
                self.secret_key,
                algorithms=['HS256'],
                audience='your-api',
                issuer='your-secure-app'
            )
            
            # Check if token is revoked
            if self.is_token_revoked(payload.get('jti')):
                raise AuthenticationError("Token has been revoked")
            
            # Additional security checks
            if 'nbf' in payload:
                nbf = datetime.fromtimestamp(payload['nbf'])
                if datetime.utcnow() < nbf:
                    raise AuthenticationError("Token not yet valid")
            
            return payload
        except jwt.ExpiredSignatureError:
            raise AuthenticationError("Token has expired")
        except jwt.InvalidTokenError:
            raise AuthenticationError("Invalid token")
```

### **Rate Limiting for Auth Endpoints**
```python
import time
from collections import defaultdict

class AuthRateLimiter:
    def __init__(self, redis_client=None):
        self.redis = redis_client
        self.local_cache = defaultdict(list)  # Fallback for testing
    
    def check_rate_limit(self, identifier: str, limit: int, window: int) -> bool:
        """Check if request is within rate limit"""
        current_time = time.time()
        
        if self.redis:
            # Use Redis for distributed rate limiting
            return self._redis_rate_limit(identifier, limit, window, current_time)
        else:
            # Use local cache for single instance
            return self._local_rate_limit(identifier, limit, window, current_time)
    
    def _redis_rate_limit(self, identifier: str, limit: int, window: int, current_time: float) -> bool:
        """Redis-based rate limiting"""
        key = f"rate_limit:{identifier}"
        
        # Remove old entries
        self.redis.zremrangebyscore(key, 0, current_time - window)
        
        # Count current requests
        current_count = self.redis.zcard(key)
        
        if current_count >= limit:
            return False
        
        # Add current request
        self.redis.zadd(key, {str(current_time): current_time})
        self.redis.expire(key, window)
        
        return True
    
    def _local_rate_limit(self, identifier: str, limit: int, window: int, current_time: float) -> bool:
        """Local memory-based rate limiting"""
        requests = self.local_cache[identifier]
        
        # Remove old requests
        self.local_cache[identifier] = [
            req_time for req_time in requests 
            if current_time - req_time < window
        ]
        
        if len(self.local_cache[identifier]) >= limit:
            return False
        
        self.local_cache[identifier].append(current_time)
        return True

class SecureAuthEndpoints:
    def __init__(self, auth_server, rate_limiter):
        self.auth_server = auth_server
        self.rate_limiter = rate_limiter
    
    def login_endpoint(self, username: str, password: str, client_ip: str):
        """Secure login endpoint with rate limiting"""
        # Rate limiting by IP
        if not self.rate_limiter.check_rate_limit(f"login:{client_ip}", 5, 300):
            return {'error': 'rate_limit_exceeded'}, 429
        
        # Rate limiting by username
        if not self.rate_limiter.check_rate_limit(f"login:{username}", 3, 600):
            return {'error': 'account_locked'}, 423
        
        # Authenticate user
        user = self.auth_server.authenticate_user(username, password)
        if not user:
            return {'error': 'invalid_credentials'}, 401
        
        # Generate tokens
        access_token = self.auth_server.generate_access_token(user.id)
        refresh_token = self.auth_server.generate_refresh_token(user.id)
        
        return {
            'access_token': access_token,
            'refresh_token': refresh_token,
            'token_type': 'Bearer',
            'expires_in': 3600
        }
```

## 🔒 Advanced Security Features

### **JWT Token Encryption (JWE)**
```python
from cryptography.fernet import Fernet

class EncryptedJWTService:
    def __init__(self, signing_key: str, encryption_key: str):
        self.signing_key = signing_key
        self.encryption_key = encryption_key
        self.fernet = Fernet(encryption_key.encode())
    
    def generate_encrypted_token(self, user_id: str, permissions: list = None) -> str:
        """Generate encrypted JWT token"""
        # Create JWT payload
        payload = {
            'user_id': user_id,
            'permissions': permissions or [],
            'iat': datetime.utcnow(),
            'exp': datetime.utcnow() + timedelta(hours=1)
        }
        
        # Sign the JWT
        jwt_token = jwt.encode(payload, self.signing_key, algorithm='HS256')
        
        # Encrypt the signed JWT
        encrypted_token = self.fernet.encrypt(jwt_token.encode())
        
        return encrypted_token.decode()
    
    def verify_encrypted_token(self, encrypted_token: str) -> dict:
        """Verify and decrypt JWT token"""
        try:
            # Decrypt the token
            jwt_token = self.fernet.decrypt(encrypted_token.encode()).decode()
            
            # Verify the JWT
            payload = jwt.decode(jwt_token, self.signing_key, algorithms=['HS256'])
            
            return payload
        except Exception:
            raise AuthenticationError("Invalid or expired token")
```

### **Multi-Factor Authentication (MFA)**
```python
import pyotp
import qrcode
from io import BytesIO
import base64

class MFAService:
    def __init__(self, issuer_name: str):
        self.issuer_name = issuer_name
    
    def generate_totp_secret(self, user_id: str) -> str:
        """Generate TOTP secret for user"""
        secret = pyotp.random_base32()
        
        # Store secret in database associated with user
        self.store_totp_secret(user_id, secret)
        
        return secret
    
    def generate_qr_code(self, user_email: str, secret: str) -> str:
        """Generate QR code for TOTP setup"""
        totp_uri = pyotp.totp.TOTP(secret).provisioning_uri(
            name=user_email,
            issuer_name=self.issuer_name
        )
        
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(totp_uri)
        qr.make(fit=True)
        
        img = qr.make_image(fill_color="black", back_color="white")
        buffer = BytesIO()
        img.save(buffer, format='PNG')
        buffer.seek(0)
        
        return base64.b64encode(buffer.getvalue()).decode()
    
    def verify_totp(self, user_id: str, totp_code: str) -> bool:
        """Verify TOTP code"""
        secret = self.get_totp_secret(user_id)
        if not secret:
            return False
        
        totp = pyotp.TOTP(secret)
        return totp.verify(totp_code, valid_window=1)
    
    def generate_backup_codes(self, user_id: str) -> list:
        """Generate backup codes for MFA"""
        backup_codes = []
        for _ in range(10):
            code = secrets.token_hex(4).upper()
            backup_codes.append(code)
        
        # Store hashed backup codes
        self.store_backup_codes(user_id, backup_codes)
        
        return backup_codes
    
    def verify_backup_code(self, user_id: str, backup_code: str) -> bool:
        """Verify and consume backup code"""
        return self.consume_backup_code(user_id, backup_code)
```

## 📊 Security Monitoring

### **Audit Logging**
```python
import logging
from datetime import datetime
from typing import Dict, Any

class SecurityAuditLogger:
    def __init__(self, logger_name: str = 'security_audit'):
        self.logger = logging.getLogger(logger_name)
        self.logger.setLevel(logging.INFO)
        
        # Create file handler
        handler = logging.FileHandler('security_audit.log')
        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
    
    def log_authentication_event(self, user_id: str, event_type: str, 
                                success: bool, metadata: Dict[str, Any] = None):
        """Log authentication events"""
        log_entry = {
            'event_type': 'authentication',
            'sub_event': event_type,
            'user_id': user_id,
            'success': success,
            'timestamp': datetime.utcnow().isoformat(),
            'metadata': metadata or {}
        }
        
        level = logging.INFO if success else logging.WARNING
        self.logger.log(level, json.dumps(log_entry))
    
    def log_authorization_event(self, user_id: str, resource: str, 
                               action: str, granted: bool, metadata: Dict[str, Any] = None):
        """Log authorization events"""
        log_entry = {
            'event_type': 'authorization',
            'user_id': user_id,
            'resource': resource,
            'action': action,
            'granted': granted,
            'timestamp': datetime.utcnow().isoformat(),
            'metadata': metadata or {}
        }
        
        level = logging.INFO if granted else logging.WARNING
        self.logger.log(level, json.dumps(log_entry))
    
    def log_security_event(self, event_type: str, severity: str, 
                          description: str, metadata: Dict[str, Any] = None):
        """Log general security events"""
        log_entry = {
            'event_type': 'security',
            'sub_event': event_type,
            'severity': severity,
            'description': description,
            'timestamp': datetime.utcnow().isoformat(),
            'metadata': metadata or {}
        }
        
        level_map = {
            'low': logging.INFO,
            'medium': logging.WARNING,
            'high': logging.ERROR,
            'critical': logging.CRITICAL
        }
        
        self.logger.log(level_map.get(severity, logging.INFO), json.dumps(log_entry))
```

### **Anomaly Detection**
```python
from collections import defaultdict
import numpy as np

class SecurityAnomalyDetector:
    def __init__(self, threshold_multiplier: float = 3.0):
        self.threshold_multiplier = threshold_multiplier
        self.user_patterns = defaultdict(list)
        self.global_patterns = defaultdict(list)
    
    def record_login_pattern(self, user_id: str, timestamp: datetime, 
                           ip_address: str, user_agent: str):
        """Record login pattern for anomaly detection"""
        pattern = {
            'timestamp': timestamp,
            'ip_address': ip_address,
            'user_agent': user_agent,
            'hour': timestamp.hour,
            'day_of_week': timestamp.weekday()
        }
        
        self.user_patterns[user_id].append(pattern)
        self.global_patterns['all'].append(pattern)
        
        # Keep only last 100 patterns per user
        if len(self.user_patterns[user_id]) > 100:
            self.user_patterns[user_id] = self.user_patterns[user_id][-100:]
    
    def detect_anomalies(self, user_id: str, current_pattern: dict) -> list:
        """Detect anomalies in user behavior"""
        anomalies = []
        user_history = self.user_patterns.get(user_id, [])
        
        if len(user_history) < 5:  # Not enough history
            return anomalies
        
        # Check for unusual login time
        usual_hours = [p['hour'] for p in user_history]
        if self._is_outlier(current_pattern['hour'], usual_hours):
            anomalies.append({
                'type': 'unusual_login_time',
                'severity': 'medium',
                'description': f"Login at unusual hour: {current_pattern['hour']}"
            })
        
        # Check for new IP address
        known_ips = set(p['ip_address'] for p in user_history)
        if current_pattern['ip_address'] not in known_ips:
            anomalies.append({
                'type': 'new_ip_address',
                'severity': 'high',
                'description': f"Login from new IP: {current_pattern['ip_address']}"
            })
        
        # Check for new user agent
        known_user_agents = set(p['user_agent'] for p in user_history)
        if current_pattern['user_agent'] not in known_user_agents:
            anomalies.append({
                'type': 'new_user_agent',
                'severity': 'medium',
                'description': "Login from new device/browser"
            })
        
        return anomalies
    
    def _is_outlier(self, value: float, data: list) -> bool:
        """Check if value is statistical outlier"""
        if len(data) < 3:
            return False
        
        mean = np.mean(data)
        std = np.std(data)
        
        return abs(value - mean) > self.threshold_multiplier * std
```

## 🔗 Related Topics

- [[Authentication-Authorization|Authentication & Authorization]]
- [[12-Practice-Problems/Rate-Limiter-Design|Rate Limiter Design]]
- [[08-Performance/Monitoring/Performance-Monitoring|Performance Monitoring]]
- [[02-System-Components/API-Gateway/API-Gateway-Pattern|API Gateway Pattern]]

## 📚 Further Reading

- **RFC 6749**: OAuth 2.0 Authorization Framework
- **RFC 7519**: JSON Web Token (JWT)
- **RFC 7636**: PKCE for OAuth Public Clients
- **OWASP OAuth Security Cheat Sheet**
- **"OAuth 2.0 Simplified" by Aaron Parecki**

---

**Key Takeaways**: 
- Always use HTTPS for OAuth flows
- Implement proper token expiration and refresh mechanisms
- Use PKCE for public clients (mobile apps, SPAs)
- Implement rate limiting and anomaly detection
- Log all security events for monitoring and compliance
- Consider token encryption for sensitive applications 