
Project Structure

```
unify-api/
├── backend/
├── frontend/
├── scripts/
├── docker-compose.yml
└── README.md
```

1. Backend Setup Scripts

1.1 Backend Main App (backend/app.py)

```python
"""
Unify API Backend - FastAPI Application
"""
from fastapi import FastAPI, HTTPException, Depends, Security
from fastapi.security import APIKeyHeader
from fastapi.middleware.cors import CORSMiddleware
from typing import Dict, Any, Optional
import asyncpg
import redis.asyncio as redis
from datetime import datetime, timedelta
import json
import logging
from pydantic import BaseModel
import httpx

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Unify API", version="1.0.0")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Database and Redis connections
pool = None
redis_client = None

# Models
class APIRequest(BaseModel):
    connector_id: str
    endpoint: str
    method: str = "GET"
    params: Optional[Dict[str, Any]] = None
    body: Optional[Dict[str, Any]] = None
    headers: Optional[Dict[str, str]] = None

class ConnectorConfig(BaseModel):
    name: str
    base_url: str
    auth_type: str  # api_key, oauth, basic
    auth_config: Dict[str, Any]
    endpoints: Dict[str, Dict[str, Any]]

# API Key security
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def get_api_key(api_key: str = Security(api_key_header)):
    if not api_key:
        raise HTTPException(status_code=401, detail="API key missing")
    
    async with pool.acquire() as conn:
        key_info = await conn.fetchrow(
            "SELECT * FROM api_keys WHERE key = $1 AND active = true",
            api_key
        )
    
    if not key_info:
        raise HTTPException(status_code=401, detail="Invalid API key")
    
    # Check rate limiting
    redis_key = f"rate_limit:{api_key}"
    current = await redis_client.get(redis_key)
    
    if current and int(current) >= key_info["rate_limit"]:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    
    # Increment counter
    if current:
        await redis_client.incr(redis_key)
    else:
        await redis_client.setex(redis_key, 3600, 1)  # 1 hour TTL
    
    return key_info

@app.on_event("startup")
async def startup():
    global pool, redis_client
    
    # Initialize database connection
    pool = await asyncpg.create_pool(
        host="localhost",
        database="unify_api",
        user="postgres",
        password="postgres",
        min_size=5,
        max_size=20
    )
    
    # Initialize Redis
    redis_client = redis.Redis(host="localhost", port=6379, decode_responses=True)
    
    logger.info("Backend services initialized")

@app.on_event("shutdown")
async def shutdown():
    if pool:
        await pool.close()
    if redis_client:
        await redis_client.close()

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}

@app.post("/api/v1/execute")
async def execute_request(
    request: APIRequest,
    api_key_info: Dict = Depends(get_api_key)
):
    """
    Execute API request through a connector
    """
    try:
        # Get connector configuration
        async with pool.acquire() as conn:
            connector = await conn.fetchrow(
                "SELECT * FROM connectors WHERE id = $1",
                request.connector_id
            )
            
            if not connector:
                raise HTTPException(status_code=404, detail="Connector not found")
            
            # Check if user has access to this connector
            access = await conn.fetchrow(
                "SELECT 1 FROM user_connectors WHERE user_id = $1 AND connector_id = $2",
                api_key_info["user_id"], request.connector_id
            )
            
            if not access:
                raise HTTPException(status_code=403, detail="No access to connector")
        
        # Get connector config
        config = json.loads(connector["config"])
        
        # Build URL
        url = f"{config['base_url']}/{request.endpoint}"
        
        # Prepare headers
        headers = request.headers or {}
        
        # Add authentication based on connector type
        if config["auth_type"] == "api_key":
            headers[config["auth_config"]["header_name"]] = config["auth_config"]["api_key"]
        elif config["auth_type"] == "bearer":
            headers["Authorization"] = f"Bearer {config['auth_config']['token']}"
        
        # Make the request
        async with httpx.AsyncClient() as client:
            response = await client.request(
                method=request.method,
                url=url,
                params=request.params,
                json=request.body,
                headers=headers,
                timeout=30.0
            )
        
        # Log the request
        async with pool.acquire() as conn:
            await conn.execute("""
                INSERT INTO request_logs 
                (user_id, connector_id, endpoint, method, status_code, request_time)
                VALUES ($1, $2, $3, $4, $5, $6)
            """, api_key_info["user_id"], request.connector_id, 
               request.endpoint, request.method, response.status_code, datetime.utcnow())
        
        # Return response
        return {
            "success": response.status_code < 400,
            "status_code": response.status_code,
            "data": response.json() if response.headers.get("content-type") == "application/json" else response.text,
            "headers": dict(response.headers)
        }
        
    except httpx.RequestError as e:
        logger.error(f"Request error: {str(e)}")
        raise HTTPException(status_code=502, detail=f"External service error: {str(e)}")
    except Exception as e:
        logger.error(f"Unexpected error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/v1/connectors")
async def list_connectors(api_key_info: Dict = Depends(get_api_key)):
    """List available connectors for the user"""
    async with pool.acquire() as conn:
        connectors = await conn.fetch("""
            SELECT c.* FROM connectors c
            JOIN user_connectors uc ON c.id = uc.connector_id
            WHERE uc.user_id = $1
        """, api_key_info["user_id"])
    
    return [dict(connector) for connector in connectors]

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

1.2 Database Setup Script (backend/database/setup.sql)

```sql
-- Unify API Database Schema
CREATE DATABASE unify_api;

\c unify_api;

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    tier VARCHAR(50) DEFAULT 'hobbyist',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    active BOOLEAN DEFAULT true
);

-- API Keys table
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    key VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    rate_limit INTEGER DEFAULT 1000,
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP
);

-- Connectors table
CREATE TABLE connectors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    service_type VARCHAR(100) NOT NULL, -- payment, shipping, crm, etc.
    icon_url VARCHAR(500),
    config JSONB NOT NULL, -- Base URL, auth config, endpoints
    version VARCHAR(50) DEFAULT '1.0.0',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    active BOOLEAN DEFAULT true
);

-- User-Connector mapping (which connectors users have access to)
CREATE TABLE user_connectors (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    connector_id UUID REFERENCES connectors(id) ON DELETE CASCADE,
    config JSONB, -- User-specific config overrides
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, connector_id)
);

-- Request logs
CREATE TABLE request_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    connector_id UUID REFERENCES connectors(id),
    endpoint VARCHAR(500) NOT NULL,
    method VARCHAR(10) NOT NULL,
    status_code INTEGER,
    request_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    response_time_ms INTEGER,
    error_message TEXT
);

-- Webhooks table
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    url VARCHAR(500) NOT NULL,
    events JSONB NOT NULL, -- Events to listen for
    secret VARCHAR(255), -- For signature verification
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_triggered_at TIMESTAMP
);

-- Usage analytics
CREATE TABLE usage_analytics (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    date DATE NOT NULL,
    connector_id UUID REFERENCES connectors(id),
    request_count INTEGER DEFAULT 0,
    success_count INTEGER DEFAULT 0,
    error_count INTEGER DEFAULT 0,
    total_response_time_ms BIGINT DEFAULT 0,
    UNIQUE(user_id, date, connector_id)
);

-- Create indexes
CREATE INDEX idx_request_logs_user_time ON request_logs(user_id, request_time DESC);
CREATE INDEX idx_request_logs_connector ON request_logs(connector_id);
CREATE INDEX idx_api_keys_key ON api_keys(key);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_usage_analytics_date ON usage_analytics(date);

-- Insert sample data
INSERT INTO users (email, name, tier) VALUES 
('admin@unifyapi.com', 'Admin User', 'enterprise'),
('user@example.com', 'Test User', 'hobbyist');

-- Insert sample API key (in production, generate properly)
INSERT INTO api_keys (user_id, key, name, rate_limit) 
SELECT id, 'test_api_key_123', 'Default Key', 10000 
FROM users WHERE email = 'user@example.com';

-- Insert sample connectors
INSERT INTO connectors (name, service_type, config) VALUES 
('Stripe', 'payment', '{
    "base_url": "https://api.stripe.com/v1",
    "auth_type": "bearer",
    "auth_config": {
        "header_name": "Authorization",
        "prefix": "Bearer"
    },
    "endpoints": {
        "create_charge": {"path": "/charges", "method": "POST"},
        "get_balance": {"path": "/balance", "method": "GET"}
    }
}'),
('SendGrid', 'email', '{
    "base_url": "https://api.sendgrid.com/v3",
    "auth_type": "bearer",
    "auth_config": {
        "header_name": "Authorization",
        "prefix": "Bearer"
    },
    "endpoints": {
        "send_email": {"path": "/mail/send", "method": "POST"}
    }
}');

-- Grant user access to connectors
INSERT INTO user_connectors (user_id, connector_id)
SELECT u.id, c.id FROM users u, connectors c
WHERE u.email = 'user@example.com';

-- Create function to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Add triggers for updated_at
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_connectors_updated_at BEFORE UPDATE ON connectors
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

1.3 Requirements File (backend/requirements.txt)

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
asyncpg==0.29.0
redis==5.0.1
httpx==0.25.1
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
pydantic==2.5.0
pydantic-settings==2.1.0
python-dotenv==1.0.0
sqlalchemy==2.0.23
alembic==1.13.0
celery==5.3.4
pytest==7.4.3
pytest-asyncio==0.21.1
```

1.4 Backend Configuration (backend/config.py)

```python
"""
Configuration for Unify API Backend
"""
import os
from pydantic_settings import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    # Application
    app_name: str = "Unify API"
    app_version: str = "1.0.0"
    debug: bool = os.getenv("DEBUG", "False").lower() == "true"
    
    # Database
    database_url: str = os.getenv("DATABASE_URL", "postgresql://postgres:postgres@localhost:5432/unify_api")
    
    # Redis
    redis_url: str = os.getenv("REDIS_URL", "redis://localhost:6379")
    
    # Security
    secret_key: str = os.getenv("SECRET_KEY", "your-secret-key-here-change-in-production")
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    
    # Rate limiting
    default_rate_limit: int = 1000
    redis_rate_limit_ttl: int = 3600  # 1 hour
    
    # CORS
    cors_origins: list = [
        "http://localhost:3000",
        "http://localhost:8000",
        "https://app.unifyapi.com"
    ]
    
    # External APIs
    stripe_api_key: Optional[str] = os.getenv("STRIPE_API_KEY")
    sendgrid_api_key: Optional[str] = os.getenv("SENDGRID_API_KEY")
    
    class Config:
        env_file = ".env"
        case_sensitive = False

settings = Settings()
```

1.5 Dockerfile for Backend (backend/Dockerfile)

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

# Run the application
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

2. Frontend Setup Scripts

2.1 Next.js Frontend (frontend/package.json)

```json
{
  "name": "unify-api-dashboard",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "14.0.4",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "axios": "^1.6.2",
    "react-query": "^3.39.3",
    "react-hook-form": "^7.48.2",
    "zod": "^3.22.4",
    "@hookform/resolvers": "^3.3.2",
    "framer-motion": "^10.16.16",
    "recharts": "^2.10.3",
    "date-fns": "^3.0.6",
    "clsx": "^2.0.0",
    "tailwind-merge": "^2.0.0",
    "next-themes": "^0.2.1",
    "lucide-react": "^0.309.0",
    "jsonwebtoken": "^9.0.2",
    "crypto-js": "^4.2.0",
    "react-hot-toast": "^2.4.1"
  },
  "devDependencies": {
    "@types/node": "20.10.0",
    "@types/react": "18.2.45",
    "@types/react-dom": "18.2.18",
    "autoprefixer": "10.4.16",
    "eslint": "8.55.0",
    "eslint-config-next": "14.0.4",
    "postcss": "8.4.32",
    "tailwindcss": "3.4.0",
    "typescript": "5.3.3",
    "@types/jsonwebtoken": "^9.0.5",
    "@types/crypto-js": "^4.2.1"
  }
}
```

2.2 Frontend Main Layout (frontend/app/layout.tsx)

```tsx
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'
import { Providers } from './providers'
import { Toaster } from 'react-hot-toast'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: 'Unify API Dashboard',
  description: 'Unified API Integration Platform',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={inter.className}>
        <Providers>
          <div className="min-h-screen bg-gray-50 dark:bg-gray-900">
            {children}
          </div>
          <Toaster position="top-right" />
        </Providers>
      </body>
    </html>
  )
}
```

2.3 Providers Component (frontend/app/providers.tsx)

```tsx
'use client'

import { ThemeProvider } from 'next-themes'
import { QueryClient, QueryClientProvider } from 'react-query'
import { ReactQueryDevtools } from 'react-query/devtools'
import { useState } from 'react'

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 minute
        retry: 1,
      },
    },
  }))

  return (
    <ThemeProvider attribute="class" defaultTheme="light" enableSystem>
      <QueryClientProvider client={queryClient}>
        {children}
        <ReactQueryDevtools initialIsOpen={false} />
      </QueryClientProvider>
    </ThemeProvider>
  )
}
```

2.4 Dashboard Page (frontend/app/dashboard/page.tsx)

```tsx
'use client'

import { useState, useEffect } from 'react'
import { useQuery } from 'react-query'
import axios from 'axios'
import {
  BarChart3,
  Globe,
  Key,
  Shield,
  Zap,
  Activity,
  CreditCard,
  Mail,
  Truck,
  Users
} from 'lucide-react'
import DashboardCard from '@/components/DashboardCard'
import ConnectorCard from '@/components/ConnectorCard'
import UsageChart from '@/components/UsageChart'

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000'

export default function DashboardPage() {
  const [apiKey, setApiKey] = useState('')
  const [stats, setStats] = useState({
    totalRequests: 0,
    successRate: 0,
    activeConnectors: 0,
    monthlyUsage: 0
  })

  // Fetch connectors
  const { data: connectors = [], isLoading } = useQuery(
    'connectors',
    async () => {
      const response = await axios.get(`${API_BASE_URL}/api/v1/connectors`, {
        headers: { 'X-API-Key': apiKey }
      })
      return response.data
    },
    {
      enabled: !!apiKey,
      retry: false
    }
  )

  // Fetch usage stats
  const { data: usageData = [] } = useQuery(
    'usage',
    async () => {
      const response = await axios.get(`${API_BASE_URL}/api/v1/analytics/usage`, {
        headers: { 'X-API-Key': apiKey }
      })
      return response.data
    },
    {
      enabled: !!apiKey,
      refetchInterval: 30000 // Refetch every 30 seconds
    }
  )

  // Sample connectors data
  const sampleConnectors = [
    {
      id: 'stripe',
      name: 'Stripe',
      description: 'Payment processing',
      icon: CreditCard,
      status: 'connected',
      requests: 1245
    },
    {
      id: 'sendgrid',
      name: 'SendGrid',
      description: 'Email service',
      icon: Mail,
      status: 'connected',
      requests: 876
    },
    {
      id: 'shopify',
      name: 'Shopify',
      description: 'E-commerce platform',
      icon: Globe,
      status: 'disconnected',
      requests: 0
    }
  ]

  return (
    <div className="p-6 space-y-6">
      {/* Header */}
      <div className="flex justify-between items-center">
        <div>
          <h1 className="text-3xl font-bold text-gray-900 dark:text-white">
            Dashboard
          </h1>
          <p className="text-gray-600 dark:text-gray-400">
            Manage your API integrations and monitor usage
          </p>
        </div>
        <div className="flex items-center space-x-4">
          <div className="relative">
            <input
              type="password"
              placeholder="Enter API Key"
              value={apiKey}
              onChange={(e) => setApiKey(e.target.value)}
              className="px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            />
            <Key className="absolute right-3 top-2.5 h-5 w-5 text-gray-400" />
          </div>
        </div>
      </div>

      {/* Stats Grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        <DashboardCard
          title="Total Requests"
          value={stats.totalRequests.toLocaleString()}
          icon={Activity}
          trend="+12.5%"
          color="blue"
        />
        <DashboardCard
          title="Success Rate"
          value={`${stats.successRate}%`}
          icon={BarChart3}
          trend="+2.3%"
          color="green"
        />
        <DashboardCard
          title="Active Connectors"
          value={stats.activeConnectors}
          icon={Zap}
          trend="+3"
          color="purple"
        />
        <DashboardCard
          title="Monthly Usage"
          value={`${stats.monthlyUsage}K`}
          icon={Shield}
          trend="+8.7%"
          color="orange"
        />
      </div>

      {/* Connectors Grid */}
      <div>
        <h2 className="text-2xl font-semibold text-gray-900 dark:text-white mb-4">
          Connectors
        </h2>
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          {sampleConnectors.map((connector) => (
            <ConnectorCard
              key={connector.id}
              {...connector}
              onConnect={() => console.log(`Connect ${connector.name}`)}
              onDisconnect={() => console.log(`Disconnect ${connector.name}`)}
            />
          ))}
        </div>
      </div>

      {/* Usage Chart */}
      <div className="bg-white dark:bg-gray-800 rounded-xl p-6 shadow">
        <h2 className="text-2xl font-semibold text-gray-900 dark:text-white mb-4">
          Usage Analytics
        </h2>
        <UsageChart data={usageData} />
      </div>

      {/* Quick Actions */}
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        <div className="bg-gradient-to-r from-blue-500 to-blue-600 rounded-xl p-6 text-white">
          <h3 className="text-xl font-semibold mb-2">Create New API Key</h3>
          <p className="mb-4">Generate secure API keys for different environments</p>
          <button className="bg-white text-blue-600 px-4 py-2 rounded-lg font-semibold hover:bg-blue-50">
            Generate Key
          </button>
        </div>
        <div className="bg-gradient-to-r from-purple-500 to-purple-600 rounded-xl p-6 text-white">
          <h3 className="text-xl font-semibold mb-2">Add New Connector</h3>
          <p className="mb-4">Connect to additional services and APIs</p>
          <button className="bg-white text-purple-600 px-4 py-2 rounded-lg font-semibold hover:bg-purple-50">
            Browse Connectors
          </button>
        </div>
        <div className="bg-gradient-to-r from-green-500 to-green-600 rounded-xl p-6 text-white">
          <h3 className="text-xl font-semibold mb-2">View Documentation</h3>
          <p className="mb-4">Explore API documentation and examples</p>
          <button className="bg-white text-green-600 px-4 py-2 rounded-lg font-semibold hover:bg-green-50">
            Open Docs
          </button>
        </div>
      </div>
    </div>
  )
}
```

2.5 Tailwind Config (frontend/tailwind.config.js)

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: ['class'],
  content: [
    './pages/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
    './src/**/*.{ts,tsx}',
  ],
  theme: {
    container: {
      center: true,
      padding: "2rem",
      screens: {
        "2xl": "1400px",
      },
    },
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
        "pulse-slow": "pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
}
```

3. Deployment & Infrastructure Scripts

3.1 Docker Compose (docker-compose.yml)

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: unify-postgres
    environment:
      POSTGRES_DB: unify_api
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backend/database/setup.sql:/docker-entrypoint-initdb.d/setup.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - unify-network

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: unify-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - unify-network

  # Backend API
  backend:
    build: ./backend
    container_name: unify-backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/unify_api
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY=${SECRET_KEY:-dev-secret-key-change-in-prod}
      - DEBUG=true
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./backend:/app
    networks:
      - unify-network
    restart: unless-stopped

  # Frontend (Next.js)
  frontend:
    build: ./frontend
    container_name: unify-frontend
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000
      - NODE_ENV=development
    depends_on:
      - backend
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - /app/.next
    networks:
      - unify-network
    restart: unless-stopped

  # Nginx Reverse Proxy (Optional)
  nginx:
    image: nginx:alpine
    container_name: unify-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - backend
      - frontend
    networks:
      - unify-network

  # Monitoring (Optional)
  grafana:
    image: grafana/grafana:latest
    container_name: unify-grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - unify-network

networks:
  unify-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  grafana_data:
```

3.2 Nginx Configuration (nginx/nginx.conf)

```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server backend:8000;
    }

    upstream frontend {
        server frontend:3000;
    }

    server {
        listen 80;
        server_name localhost;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name localhost;

        ssl_certificate /etc/nginx/ssl/localhost.crt;
        ssl_certificate_key /etc/nginx/ssl/localhost.key;

        # Frontend
        location / {
            proxy_pass http://frontend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Backend API
        location /api/ {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Health checks
        location /health {
            proxy_pass http://backend/health;
        }

        # WebSocket support
        location /ws/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
        }
    }
}
```

3.3 Deployment Script (scripts/deploy.sh)

```bash
#!/bin/bash

# Unify API Deployment Script
set -e

echo "🚀 Starting Unify API deployment..."

# Load environment variables
if [ -f .env ]; then
    export $(cat .env | grep -v '^#' | xargs)
fi

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Function to print colored output
print_message() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

print_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

print_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

# Check prerequisites
check_prerequisites() {
    print_message "Checking prerequisites..."
    
    # Check Docker
    if ! command -v docker &> /dev/null; then
        print_error "Docker is not installed. Please install Docker first."
        exit 1
    fi
    
    # Check Docker Compose
    if ! command -v docker-compose &> /dev/null; then
        print_error "Docker Compose is not installed. Please install Docker Compose first."
        exit 1
    fi
    
    print_message "Prerequisites check passed ✓"
}

# Build and start services
deploy_services() {
    print_message "Building and starting services..."
    
    # Build Docker images
    docker-compose build
    
    # Start services
    docker-compose up -d
    
    # Wait for services to be healthy
    print_message "Waiting for services to be ready..."
    sleep 30
    
    # Check service health
    if docker-compose ps | grep -q "unhealthy"; then
        print_error "Some services are unhealthy. Check logs with: docker-compose logs"
        exit 1
    fi
    
    print_message "Services deployed successfully ✓"
}

# Run database migrations
run_migrations() {
    print_message "Running database migrations..."
    
    # Create database if not exists
    docker-compose exec postgres psql -U postgres -c "CREATE DATABASE unify_api;" 2>/dev/null || true
    
    # Run SQL setup script
    docker-compose exec postgres psql -U postgres -d unify_api -f /docker-entrypoint-initdb.d/setup.sql
    
    print_message "Database setup completed ✓"
}

# Generate SSL certificates
generate_ssl() {
    print_message "Generating SSL certificates..."
    
    mkdir -p nginx/ssl
    
    # Generate self-signed certificate (for development)
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout nginx/ssl/localhost.key \
        -out nginx/ssl/localhost.crt \
        -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost" 2>/dev/null
    
    print_message "SSL certificates generated ✓"
}

# Setup monitoring
setup_monitoring() {
    print_message "Setting up monitoring..."
    
    # Create Grafana datasource for Prometheus
    cat > scripts/grafana-datasource.yaml << EOF
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
EOF
    
    print_message "Monitoring setup completed ✓"
}

# Show deployment info
show_deployment_info() {
    print_message "=== Deployment Complete ==="
    echo ""
    echo "Services:"
    echo "  Frontend:     http://localhost:3000"
    echo "  Backend API:  http://localhost:8000"
    echo "  PostgreSQL:   localhost:5432"
    echo "  Redis:        localhost:6379"
    echo "  Grafana:      http://localhost:3001"
    echo ""
    echo "Default credentials:"
    echo "  API Key:      test_api_key_123"
    echo "  Grafana:      admin/admin"
    echo ""
    print_message "To view logs: docker-compose logs -f"
    print_message "To stop services: docker-compose down"
}

# Main deployment flow
main() {
    print_message "Starting Unify API Platform deployment..."
    
    check_prerequisites
    generate_ssl
    deploy_services
    run_migrations
    setup_monitoring
    show_deployment_info
    
    print_message "🎉 Deployment completed successfully!"
}

# Run main function
main "$@"
```

3.4 Backup Script (scripts/backup.sh)

```bash
#!/bin/bash

# Unify API Backup Script
set -e

BACKUP_DIR="./backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$TIMESTAMP.tar.gz"

mkdir -p "$BACKUP_DIR"

echo "Starting backup..."

# Backup PostgreSQL
docker-compose exec -T postgres pg_dump -U postgres unify_api > "$BACKUP_DIR/database_$TIMESTAMP.sql"

# Backup Redis (if persistent)
docker-compose exec redis redis-cli SAVE
docker cp unify-redis:/data/dump.rdb "$BACKUP_DIR/redis_$TIMESTAMP.rdb"

# Backup configuration files
tar -czf "$BACKUP_FILE" \
    backend/config.py \
    backend/.env \
    frontend/.env.local \
    docker-compose.yml \
    "$BACKUP_DIR/database_$TIMESTAMP.sql" \
    "$BACKUP_DIR/redis_$TIMESTAMP.rdb"

# Cleanup
rm -f "$BACKUP_DIR/database_$TIMESTAMP.sql"
rm -f "$BACKUP_DIR/redis_$TIMESTAMP.rdb"

echo "Backup completed: $BACKUP_FILE"

# Keep only last 7 backups
ls -t "$BACKUP_DIR"/backup_*.tar.gz | tail -n +8 | xargs -r rm -f
```

4. Testing Scripts

4.1 API Test Script (scripts/test_api.py)

```python
#!/usr/bin/env python3
"""
Unify API Test Script
"""
import requests
import json
import time
import sys

API_BASE_URL = "http://localhost:8000"
API_KEY = "test_api_key_123"

def test_health():
    """Test health endpoint"""
    print("Testing health endpoint...")
    response = requests.get(f"{API_BASE_URL}/health")
    assert response.status_code == 200
    print(f"✓ Health check passed: {response.json()}")

def test_list_connectors():
    """Test listing connectors"""
    print("\nTesting connector listing...")
    headers = {"X-API-Key": API_KEY}
    response = requests.get(f"{API_BASE_URL}/api/v1/connectors", headers=headers)
    assert response.status_code == 200
    connectors = response.json()
    print(f"✓ Found {len(connectors)} connectors")
    return connectors

def test_api_execution():
    """Test API execution through a connector"""
    print("\nTesting API execution...")
    
    # This is a mock test since we need actual connector credentials
    test_request = {
        "connector_id": "test-connector",
        "endpoint": "test",
        "method": "GET",
        "params": {"test": "true"}
    }
    
    headers = {
        "X-API-Key": API_KEY,
        "Content-Type": "application/json"
    }
    
    try:
        response = requests.post(
            f"{API_BASE_URL}/api/v1/execute",
            json=test_request,
            headers=headers
        )
        print(f"Response status: {response.status_code}")
        if response.status_code < 400:
            print("✓ API execution test passed")
        else:
            print(f"✗ API execution test failed: {response.text}")
    except Exception as e:
        print(f"✗ API execution test failed: {str(e)}")

def test_rate_limiting():
    """Test rate limiting"""
    print("\nTesting rate limiting...")
    
    headers = {"X-API-Key": API_KEY}
    
    # Make multiple rapid requests
    for i in range(15):
        response = requests.get(f"{API_BASE_URL}/api/v1/connectors", headers=headers)
        if response.status_code == 429:
            print(f"✓ Rate limiting triggered after {i+1} requests")
            return
        time.sleep(0.1)
    
    print("✗ Rate limiting not triggered")

def run_all_tests():
    """Run all tests"""
    print("=" * 50)
    print("Running Unify API Tests")
    print("=" * 50)
    
    tests = [
        test_health,
        test_list_connectors,
        test_api_execution,
        test_rate_limiting
    ]
    
    passed = 0
    failed = 0
    
    for test in tests:
        try:
            test()
            passed += 1
        except Exception as e:
            print(f"✗ Test failed: {str(e)}")
            failed += 1
    
    print("\n" + "=" * 50)
    print(f"Test Results: {passed} passed, {failed} failed")
    print("=" * 50)
    
    return failed == 0

if __name__ == "__main__":
    success = run_all_tests()
    sys.exit(0 if success else 1)
```

5. README Documentation

5.1 Main README (README.md)

```markdown
# Unify API Platform

A unified API aggregation and integration platform that connects to multiple services and provides one consistent interface.

## Features

- **API Connector Builder**: GUI or config-based connector creation
- **Request/Response Mapping**: Transform requests between different API formats
- **Rate Limiting & Retry Logic**: Intelligent handling of API limits and failures
- **Webhook Support**: Receive and forward webhooks from connected services
- **API Key Management**: Secure API key generation and management
- **Usage Analytics**: Monitor API usage and performance

## Architecture

```

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│Frontend      │    │   Backend API   │    │   Database      │
│(Next.js)     │◄──►│   (FastAPI)     │◄──►│   (PostgreSQL)  │
││    │                 │    │                 │
└─────────────────┘└─────────────────┘    └─────────────────┘
▲                       ▲                       ▲
│                       │                       │
┌─────────────────┐┌─────────────────┐    ┌─────────────────┐
│Nginx       │    │     Redis       │    │    Monitoring   │
│(Reverse      │    │   (Cache &      │    │   (Grafana)     │
│Proxy)      │    │     Queue)      │    │                 │
└─────────────────┘└─────────────────┘    └─────────────────┘

```

## Quick Start

### Prerequisites
- Docker & Docker Compose
- Node.js 18+ (for local development)
- Python 3.11+ (for local development)

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/unify-api.git
   cd unify-api
```

1. Set up environment variables
   ```bash
   cp .env.example .env
   # Edit .env with your configuration
   ```
2. Deploy using Docker (Recommended)
   ```bash
   chmod +x scripts/deploy.sh
   ./scripts/deploy.sh
   ```
3. Or run locally
   ```bash
   # Start database and cache
   docker-compose up -d postgres redis
   
   # Set up backend
   cd backend
   python -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   python app.py
   
   # Set up frontend
   cd frontend
   npm install
   npm run dev
   ```

Access the Application

· Frontend Dashboard: http://localhost:3000
· Backend API: http://localhost:8000
· API Documentation: http://localhost:8000/docs
· Database: localhost:5432 (user: postgres, pass: postgres)
· Redis: localhost:6379

Default API Key: test_api_key_123

Project Structure

```
unify-api/
├── backend/              # FastAPI backend
│   ├── app.py           # Main application
│   ├── config.py        # Configuration
│   ├── requirements.txt # Python dependencies
│   └── database/        # Database scripts
├── frontend/            # Next.js frontend
│   ├── app/             # App router pages
│   ├── components/      # React components
│   ├── public/          # Static files
│   └── package.json     # Node dependencies
├── scripts/             # Deployment & utility scripts
├── nginx/               # Reverse proxy configuration
├── docker-compose.yml   # Docker orchestration
└── README.md            # This file
```

API Usage Examples

List Available Connectors

```bash
curl -H "X-API-Key: test_api_key_123" \
  http://localhost:8000/api/v1/connectors
```

Execute API Request

```bash
curl -X POST \
  -H "X-API-Key: test_api_key_123" \
  -H "Content-Type: application/json" \
  -d '{
    "connector_id": "stripe",
    "endpoint": "customers",
    "method": "GET"
  }' \
  http://localhost:8000/api/v1/execute
```

Pricing Tiers

Tier Price Features
Hobbyist $499 one-time 5 connectors, 10K req/month, basic dashboard
Pro $1999 one-time 20 connectors, 100K req/month, webhooks, team access
Enterprise $4,999 one-time Unlimited connectors, white-label, SSO, SLA, source code
Custom $10K+ Custom connectors, on-prem deploy, dedicated support

Development

Adding a New Connector

1. Create a connector configuration:
   ```json
   {
     "name": "Service Name",
     "base_url": "https://api.service.com/v1",
     "auth_type": "bearer",
     "auth_config": {
       "header_name": "Authorization",
       "prefix": "Bearer"
     },
     "endpoints": {
       "resource": {
         "path": "/resource",
         "method": "GET"
       }
     }
   }
   ```
2. Add to database:
   ```sql
   INSERT INTO connectors (name, service_type, config)
   VALUES ('Service Name', 'category', '{"config": "json"}');
   ```

Testing

```bash
# Run all tests
python scripts/test_api.py

# Run specific test
python -m pytest backend/tests/ -v
```

Deployment

Production Deployment

1. Update .env with production values
2. Set up SSL certificates in nginx/ssl/
3. Run deployment script:
   ```bash
   ./scripts/deploy.sh --production
   ```

Backup and Restore

```bash
# Create backup
./scripts/backup.sh

# Restore from backup
./scripts/restore.sh backups/backup_20231201_120000.tar.gz
```

Monitoring

Access Grafana at http://localhost:3001

· Username: admin
· Password: admin

Monitor:

· API request rates and latency
· Error rates by connector
· Resource usage (CPU, memory)
· Database performance

Troubleshooting

Common Issues

1. Database connection failed
   ```bash
   docker-compose logs postgres
   ```
2. API returning 429 errors
   · Check rate limits in API key configuration
   · Monitor usage in dashboard
3. Frontend not loading
   ```bash
   docker-compose logs frontend
   ```

Logs

```bash
# View all logs
docker-compose logs -f

# View specific service
docker-compose logs -f backend

# View real-time logs
docker-compose logs --tail=100 -f
```

Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run tests
5. Submit a pull request

License

This project is proprietary software. All rights reserved.

Support

For support, email: support@unifyapi.com
Documentation:https://docs.unifyapi.com

```

## **Getting Started Instructions**

1. **Clone and setup:**
   ```bash
   git clone <repository-url>
   cd unify-api
   chmod +x scripts/*.sh
```

1. Deploy with Docker:
   ```bash
   ./scripts/deploy.sh
   ```
2. Access the application:
   · Dashboard: http://localhost:3000
   · API Docs: http://localhost:8000/docs
   · Use API Key: test_api_key_123
3. Test the API:
   ```bash
   python scripts/test_api.py
   ```
