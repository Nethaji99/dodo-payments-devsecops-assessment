FROM python:3.12-slim

# Create non-root user/group
RUN groupadd -r appgroup && useradd -r -g appgroup -u 10001 appuser
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
# Ensure app files are owned by non-root user
RUN chown -R appuser:appgroup /app
# Switch to non-root user
USER 10001
EXPOSE 8080
CMD ["python", "app.py"]
