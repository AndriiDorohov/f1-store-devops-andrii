# backend_redis/Dockerfile

# 1. Base image
FROM python:3.10-slim

# 2. Python settings
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# 3. System dependencies (just in case, as for backend_rds)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# 4. Working directory
WORKDIR /app

# 5. First copy requirements.txt
COPY requirements.txt .

# 6. Install dependencies
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# 7. Copy all source code
COPY . .

# 8. Expose port inside container
EXPOSE 8000

# 9. Run Django server
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
