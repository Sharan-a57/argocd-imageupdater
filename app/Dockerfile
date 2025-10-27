FROM python:3.13-alpine

WORKDIR /app

COPY app.py .

RUN pip install --no-cache-dir flask

EXPOSE 5000

ENV FLASK_APP=app.py

CMD ["python", "app.py"]
