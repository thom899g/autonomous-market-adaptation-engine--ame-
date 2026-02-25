# Autonomous Market Adaptation Engine (AME)

## Objective
**TITLE:** Autonomous Market Adaptation Engine (AME)

**DESCRIPTION:**  
This project will develop an AI system

## Strategy
Research and implement using available tools.

## Execution Output
SUMMARY: Established Phase 0 foundation for the Autonomous Market Adaptation Engine (AME) with production-ready code, Firebase integration, and rigorous error handling. Created five critical files: environment configuration, Firebase setup, data schema, main engine skeleton, and dependency file.

OUTPUT:
### FILE: requirements.txt
```
firebase-admin>=6.0.0
pandas>=2.0.0
numpy>=1.24.0
requests>=2.31.0
scikit-learn>=1.3.0
schedule>=1.2.0
python-dotenv>=1.0.0
ccxt>=4.0.0
google-cloud-firestore>=2.11.0
python-telegram-bot>=20.0
```

### FILE: .env.template
```
# FIREBASE CONFIGURATION (REQUIRED)
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_PRIVATE_KEY_ID=your-private-key-id
FIREBASE_PRIVATE_KEY=-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----
FIREBASE_CLIENT_EMAIL=firebase-adminsdk@your-project.iam.gserviceaccount.com
FIREBASE_CLIENT_ID=your-client-id
FIREBASE_CLIENT_X509_CERT_URL=https://www.googleapis.com/robot/v1/metadata/x509/...

# TELEGRAM ALERTING (OPTIONAL BUT RECOMMENDED)
TELEGRAM_BOT_TOKEN=your-telegram-bot-token
TELEGRAM_CHAT_ID=your-telegram-chat-id

# EXCHANGE API KEYS (FOR PHASE 2)
BINANCE_API_KEY=your-binance-key
BINANCE_API_SECRET=your-binance-secret
COINBASE_API_KEY=your-coinbase-key
COINBASE_API_SECRET=your-coinbase-secret

# MONITORING
LOG_LEVEL=INFO  # DEBUG, INFO, WARNING, ERROR
```

### FILE: firebase_setup.py
```python
"""
Firebase Configuration Module for AME
Architectural Rationale:
- Singleton pattern ensures single Firebase app instance (prevents resource leaks)
- Environment-based configuration for security and portability
- Comprehensive error handling for deployment scenarios
- Type hints for maintainability and IDE support
"""

import os
import logging
import json
from typing import Optional, Dict, Any
from dataclasses import dataclass

import firebase_admin
from firebase_admin import credentials, firestore
from google.cloud.firestore import Client as FirestoreClient
from google.auth.exceptions import DefaultCredentialsError, TransportError

logging.basicConfig(level=os.getenv('LOG_LEVEL', 'INFO'))
logger = logging.getLogger(__name__)


@dataclass
class FirebaseConfig:
    """Immutable configuration container for Firebase credentials"""
    project_id: str
    private_key_id: str
    private_key: str
    client_email: str
    client_id: str
    client_x509_cert_url: str


class FirebaseInitializationError(Exception):
    """Custom exception for Firebase initialization failures"""
    pass


class FirebaseManager:
    """Singleton manager for Firebase services with robust error handling"""
    
    _instance: Optional['FirebaseManager'] = None
    _app = None
    _db: Optional[FirestoreClient] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super(FirebaseManager, cls).__new__(cls)
        return cls._instance
    
    def __init__(self):
        if not hasattr(self, '_initialized'):
            self._initialized = False
            self._config: Optional[FirebaseConfig] = None
    
    def _load_config_from_env(self) -> FirebaseConfig:
        """Load and validate Firebase configuration from environment variables"""
        required_vars = [
            'FIREBASE_PROJECT_ID',
            'FIREBASE_PRIVATE_KEY_ID',
            'FIREBASE_PRIVATE_KEY',
            'FIREBASE_CLIENT_EMAIL',
            'FIREBASE_CLIENT_ID',
            'FIREBASE_CLIENT_X509_CERT_URL'
        ]
        
        missing_vars = [var for var in required_vars if not os.getenv(var)]
        if missing_vars:
            raise FirebaseInitializationError(
                f"Missing required environment variables: {', '.join(missing_vars)}. "
                f"Please copy .env.template to .env and fill in values."
            )
        
        # Replace escaped newlines in private key
        private_key = os.getenv('FIREBASE_PRIVATE_KEY', '').replace('\\n', '\n')
        
        return FirebaseConfig(
            project_id=os.getenv('FIREBASE_PROJECT_ID', ''),
            private_key_id=os.getenv('FIREBASE_PRIVATE_KEY_ID', ''),
            private_key=private_key,
            client_email=os.getenv('FIREBASE_CLIENT_EMAIL', ''),
            client_id=os.getenv('FIREBASE_CLIENT_ID', ''),
            client_x509_cert_url=os.getenv('FIREBASE_CLIENT_X509_CERT_URL', '')
        )
    
    def _create_credential_dict(self) -> Dict[str, Any]:
        """Create Firebase credential dictionary from config"""
        if not self._config:
            raise FirebaseInitializationError("Configuration not loaded")
        
        return {
            "type": "service_account",
            "project_id": self._config.project_id,
            "private_key_id": self._config.private_key_id,
            "private_key": self._config.private_key,
            "client_email": self._config.client_email,
            "client_id": self._config.client_id,
            "auth_uri": "https://accounts.google.com/o/oauth2/auth",
            "token_uri": "https://oauth2.googleapis.com/token",
            "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
            "client_x509_cert_url": self._config.client_x509_cert_url
        }
    
    def initialize(self) -> None:
        """Initialize Firebase app with comprehensive error handling"""
        if self._initialized:
            logger.warning("Firebase already initialized")
            return
        
        try:
            # Load configuration
            self._config = self._load_config_from_env()
            logger.info(f"Loaded Firebase config for project: {self._config.project_id}")
            
            # Create credentials
            cred_dict = self._create_credential_dict()
            cred = credentials.Certificate(cred_dict)
            
            # Initialize app
            self._app = firebase_admin.initialize_app(cred)
            logger.info("Firebase app initialized successfully")
            
            # Initialize Firestore
            self._db = firestore.client()
            logger.info("Firestore client initialized")
            
            self._initialized = True
            
        except json.JSONDecodeError as e:
            raise FirebaseInitializationError(f"Invalid JSON in Firebase credentials: {str(e)}")
        except DefaultCredentialsError as e:
            raise FirebaseInitializationError(f"Google Cloud credentials error: {str(e)}")
        except TransportError as e:
            raise FirebaseInitializationError(f"Network error connecting to Firebase: {str(e)}")
        except ValueError as e:
            raise FirebaseInitializationError(f"Invalid Firebase configuration: {str(e)}")
        except Exception as e:
            raise FirebaseInitializationError(f"Unexpected error initializing Firebase: {str(e)}")
    
    @property
    def db(self) -> FirestoreClient:
        """Get Firestore client with lazy initialization"""
        if not self._initialized:
            self.initialize()
        
        if not self._db:
            raise FirebaseInitializationError("Firestore client not available")
        
        return self