FROM python:3.6.0-alpine
RUN apk update && apk add gcc musl-dev \
  && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY . /app
RUN pip install -r /app/requirements.txt
ENV POST_DATABASE_HOST post_db
ENV POST_DATABASE posts
CMD ["python3", "post_app.py"]
