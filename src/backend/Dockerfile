# Backend Order Processing Service
FROM python:3.11-slim AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.11-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app
COPY --from=builder /install /usr/local
RUN useradd --create-home --uid 10001 --shell /usr/sbin/nologin appuser
COPY --chown=appuser:appuser main.py .

USER appuser
EXPOSE 8080

ENTRYPOINT ["python", "-m", "uvicorn"]
CMD ["main:app", "--host", "0.0.0.0", "--port", "8080"]
