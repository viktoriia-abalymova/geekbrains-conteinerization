from python:alpine

workdir /app

copy .  /app

run pip3 install -r requirements.txt

expose 8080

cmd ["python3", "app.py"]
