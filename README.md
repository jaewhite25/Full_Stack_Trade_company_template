# Full_Stack_Trade_company_template
ready-to-run full-stack Trade Company template that uses Python (FastAPI), C# (.NET 8 Web API), and JavaScript (React + Vite). I packaged it as a monorepo with Docker support.

How to run locally (no Docker)L

1. Catalog (Python/FastAPI)

cd services/catalog-python
python -m venv .venv
# Windows: .venv\Scripts\activate   macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload

2. Orders (C#/.NET 8)

cd services/orders-csharp
dotnet restore
dotnet run --urls=http://0.0.0.0:8002

3. Frontend (React + Vite)
cd frontend
npm install
npm run dev -- --host

Open the printed URL (usually http://localhost:5173)

}Run with Docker|

docker compose up --build

API routes

GET http://localhost:8001/api/catalog → sample products

GET http://localhost:8001/api/partners → sample trade partners

GET http://localhost:8002/api/orders → list in-memory orders

POST http://localhost:8002/api/orders
json
{ "productId": "sku-001", "quantity": 2, "customer": "Acme Ltd." }

SETTING UP FURTHER Advancements including:
1.Authentication with JWT

2. PostgreSQL for data persistence (replacing the in-memory DB)

3. CI/CD pipeline for Azure deployment

Step 1: Set Up JWT Authentication for Frontend & Backend

Backend (Python - FastAPI)

Install pyjwt and sqlalchemy to handle JWT auth and PostgreSQL integration:

   Step 1: Set Up JWT Authentication for Frontend & Backend
pip install pyjwt sqlalchemy psycopg2

2. Update the FastAPI app to support authentication:Add JWT-based login.

Protect some routes with token verification.

Here’s a simple JWT setup for FastAPI:

services/catalog-python/app/main.py

from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
import jwt
from datetime import datetime, timedelta
from pydantic import BaseModel
from typing import List

app = FastAPI()

# JWT Config
SECRET_KEY = "mysecretkey"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# OAuth2 scheme for password login
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class Product(BaseModel):
    id: str
    name: str
    description: str

# Simulate a database
fake_users_db = {
    "user1": {
        "username": "user1",
        "password": "password1",
    },
}

def create_access_token(data: dict, expires_delta: timedelta = timedelta(minutes=15)):
    to_encode = data.copy()
    expire = datetime.utcnow() + expires_delta
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def verify_token(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token has expired")
    except jwt.JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")

@app.post("/token")
async def login_for_access_token(form_data: OAuth2PasswordRequestForm = Depends()):
    user = fake_users_db.get(form_data.username)
    if user and user["password"] == form_data.password:
        token = create_access_token(data={"sub": form_data.username})
        return {"access_token": token, "token_type": "bearer"}
    raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid credentials")

# Protecting routes
@app.get("/api/catalog", response_model=List[Product])
async def get_catalog(token: str = Depends(oauth2_scheme)):
    verify_token(token)
    return [
        Product(id="sku-001", name="Copper Wire", description="High-conductivity copper wire, 1mm"),
        Product(id="sku-002", name="Steel Rods", description="A36 hot rolled steel rods"),
        Product(id="sku-003", name="Polymer Pellets", description="Injection-grade polymer pellets"),
    ]

This adds an authentication endpoint /token to get a JWT token. The /api/catalog route is now protected and requires a valid token.

Backend (C# - .NET 8 Web API)

Install the JWT middleware package for ASP.NET:

dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

Configure JWT authentication in Program.cs:

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = false,
        ValidateAudience = false,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("mysecretkey"))
    };
});

Protect routes with [Authorize]:
[Authorize]
[HttpGet]
public IActionResult Get() => Ok(_svc.GetAll());
Step 2: Add PostgreSQL for Persistent Data

Install PostgreSQL and configure connection in FastAPI:

Update requirements.txt to include psycopg2 and sqlalchemy for PostgreSQL.

Set up models and migrations using SQLAlchemy.

Update FastAPI to connect to PostgreSQL:

from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/dbname"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

class Product(Base):
    __tablename__ = "products"
    id = Column(String, primary_key=True, index=True)
    name = Column(String, index=True)
    description = Column(String)

Base.metadata.create_all(bind=engine)

Update C# service to store orders in PostgreSQL by using Entity Framework (EF) Core.

Update Docker Compose to include a PostgreSQL container:
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: tradecompany
    ports:
      - "5432:5432"
  catalog-python:
    build: ./services/catalog-python
    ports:
      - "8001:8001"
    depends_on:
      - postgres

Step 3: Add CI/CD Pipeline for Azure Deployment

Set up a GitHub Actions workflow (.github/workflows/deploy.yml) to deploy to Azure.

Configure Azure CLI to push to a Web App:
name: Deploy to Azure

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Build Docker images
        run: |
          docker-compose -f docker-compose.yml build

      - name: Log in to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ secrets.AZURE_APP_NAME }}
          images: ${{ secrets.DOCKER_IMAGE }}

This will trigger the deployment to Azure whenever a change is pushed to main.

Step 4: Deploy to AWS (Optional)

For AWS deployment, you can set up the AWS CodePipeline or use AWS Elastic Beanstalk to deploy your application. You would need to create an Elastic Beanstalk environment and configure your project for Docker.

Once you integrate the JWT auth, PostgreSQL, and CI/CD pipeline, you’ll have a fully functional, persistent, and deployable web app with modern authentication and persistent storage.
