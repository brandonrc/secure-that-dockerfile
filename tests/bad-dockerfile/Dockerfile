# Deliberately bad Dockerfile for testing secure-dockerfile:audit
# This file should trigger findings for most Standard and Hardened rules.

FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y gcc make curl wget python3 python3-pip nodejs npm

ENV DB_PASSWORD=supersecret123
ENV API_KEY=sk-abc123def456

COPY . .

RUN pip3 install -r requirements.txt
RUN npm install

RUN python3 setup.py build

EXPOSE 8080
EXPOSE 9090
EXPOSE 3000
EXPOSE 5432

CMD python3 app.py
