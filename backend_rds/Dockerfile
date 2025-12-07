# backend_rds/Dockerfile

# 1. Base image
# Django 3.2 officially supports Python up to 3.10  
FROM python:3.10-slim

# 2. Base environment variables for Python
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# 3. Install system dependencies
# psycopg2 requires libpq-dev + build-essential
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# 4. Working directory inside the container
WORKDIR /app

# 5. Copy only requirements first (for Docker cache efficiency)
COPY requirements.txt .

# 6. Install Python dependencies
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# 7. Copy all backend code (manage.py, backend_rds/, etc.)
COPY . .

# 8. Expose the port on which Django will run
EXPOSE 8000

# 9. Default command — run Django development server
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
