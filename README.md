pip install sqlalchemy psycopg2 pyjwt fastapi uvicorn
fastapi==0.111.0
uvicorn==0.30.0
sqlalchemy==2.0.10
psycopg2==2.9.3
pyjwt==2.6.0

dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# Postgres database connection
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

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: tradecompany
    ports:
      - "5432:5432"

dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Design

builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql("Host=postgres;Port=5432;Database=tradecompany;Username=user;Password=password"));

version: "3.9"
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

  orders-csharp:
    build: ./services/orders-csharp
    ports:
      - "8002:8002"
    depends_on:
      - postgres

  frontend:
    build: ./frontend
    ports:
      - "5173:5173"
    depends_on:
      - catalog-python
      - orders-csharp

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

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

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

@app.get("/api/catalog", response_model=List[Product])
async def get_catalog(token: str = Depends(oauth2_scheme)):
    verify_token(token)
    return [
        Product(id="sku-001", name="Copper Wire", description="High-conductivity copper wire, 1mm"),
        Product(id="sku-002", name="Steel Rods", description="A36 hot rolled steel rods"),
        Product(id="sku-003", name="Polymer Pellets", description="Injection-grade polymer pellets"),
    ]
persistent storage.
