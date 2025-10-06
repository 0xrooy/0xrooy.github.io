In this document, I will record on how to create a project from scratch using Next.js and Fast API. I often found that it was so difficult for me to set up an environment on new project. So, this documentation will guide to create a project along Docker containerization and shell script.

Before we start, it is important to set up a directory for our project. In Linux, we can do so by:

# 1. Create project directory
mkdir fullstack-app
cd fullstack-app

# 2. Create backend and frontend folders
mkdir backend frontend

# 3. Create shell scripts
touch setup.sh start.sh stop.sh

# 4. Create docker-compose.yml
touch docker-compose.yml


BACKEND
After creating main sub files, we deep dive into each one. Start with backend first:
cd backend
mkdir app
touch requirements.txt Dockerfile .env

requirements is all the tools that need to be installed for backend such as Fastapi, uvicorn and so on. In order to download it one by one, we need to put it on one file and download it once and for all. Sample:
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary
python-dotenv

Dockerfile is the file that control docker activity in backend. It would interact with Dockerfile base in the main directory, and same goes to Docker file in Frontend. This is the sample of Dockerfile in backend:
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY ./app ./app
EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

At this time, Dockerfile is the one that execute requirements.txt.

.env file is the file that protecting backend sensitive information especially the database information such as link,  name and password.
sample:
DATABASE_URL=postgresql://postgres:postgres@db:5432/mydatabase

Once backend settle, we can start to write the main.py code and its database. sample:

from fastapi import FastAPI
from .api import routes

app = FastAPI(title="FastAPI Backend")

app.include_router(routes.router)

@app.get("/")
def root():
    return {"message": "FastAPI backend running successfully!"}


Apart from that, it is important to connect the web with database. here is app/db/database.py:
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os
from dotenv import load_dotenv

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()



FRONTEND
This is how we can create complete frontend directory:
cd ../frontend
npm init -y
npm install next react react-dom
mkdir -p src/pages
touch src/pages/index.js Dockerfile next.config.js

Similar to Backend, Frontend also need Dockerfile.
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app ./
EXPOSE 3000
CMD ["npm", "start"]


NOW, back to project root, as each backend and frontend had Dockerfile, the root directory need to have docker-compose.yaml:
version: "3.9"

services:
  db:
    image: postgres:16
    container_name: postgres_db
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: mydatabase
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backend:
    build: ./backend
    container_name: fastapi_backend
    restart: always
    env_file:
      - ./backend/.env
    depends_on:
      - db
    ports:
      - "8000:8000"

  frontend:
    build: ./frontend
    container_name: nextjs_frontend
    restart: always
    depends_on:
      - backend
    ports:
      - "3000:3000"

volumes:
  postgres_data:


This is important for the docker to interact with the one in backend and frontend each.

FINALLY, THE BEST PART IS SHELL SCRIPTING


setup.sh:
#!/bin/bash
echo "🔧 Building and starting containers..."
docker-compose up --build -d
echo "✅ Setup complete!"
echo "Frontend → http://localhost:3000"
echo "Backend  → http://localhost:8000"


start.sh:
#!/bin/bash
echo "▶️ Starting containers..."
docker-compose up -d

stop.sh:
#!/bin/bash
echo "🛑 Stopping containers..."
docker-compose down


AND MAKE SURE THEY ARE ACTIVE FIRST:
chmod +x setup.sh start.sh stop.sh


USING GIT VERSION CONTROL
cd ......
git init
then, create .gitignore:, automatically:npx gitignore node,python

# Python
__pycache__/
*.pyc
.env
venv/
.env.local
.env.*.local

# Node (Next.js)
node_modules/
.next/
dist/
out/

# Docker
*.pid
*.log

# OS
.DS_Store


After that, just:
git add .
git commit -m "Whatever u wanna say"
git push -u origin main



