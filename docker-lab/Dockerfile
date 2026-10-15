FROM python:3.11-slim
WORKDIR /app

# Keep dependencies separate so application edits reuse this build step.
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
EXPOSE 5000
ENV FLASK_APP=app.py
CMD ["python", "app.py"]
