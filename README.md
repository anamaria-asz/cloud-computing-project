# Cloud-Based Fall Detection System for Elderly and Hospitalized Patients

**Author:** Ana Maria Alvarez – DVOR2C  
**Deployment:** Google Cloud Platform (GCE) · Docker  
**Stack:** Python · FastAPI · Apache Kafka · MongoDB · InfluxDB · Grafana · Twilio

---

## Overview

Falls are the second leading cause of unintentional injury deaths worldwide, with approximately 684,000 fatalities annually. In hospital settings, they are the most frequent adverse event — occurring at 3 to 5 per 1,000 bed-days, with over one third resulting in injury.

This project implements a real-time, cloud-deployed fall detection system using simulated wearable sensor data, a streaming data pipeline, and automated multi-channel alerting. The full system runs as containerized microservices orchestrated via Docker Compose and deployed on Google Cloud.

**GitHub:** [anamaria-st/cloud-computing-project](https://github.com/anamaria-st/cloud-computing-project/tree/main)

---

## Architecture

```
Sensor Simulator
      │
      │  JSON events (1/sec)
      ▼
 Kafka Broker ──── Zookeeper
      │
      │  delivers events
      ▼
Fall Detector (FastAPI)
      ├── stores structured data ──► MongoDB
      ├── stores time-series ───────► InfluxDB ──► Grafana Dashboard
      ├── triggers SMS ────────────► Twilio
      └── triggers email ──────────► SMTP (Gmail)
```

Two architecture modes are supported:

**Real-world** — Wearable accelerometers and room/bed sensors send data through an IoT Gateway to the Kafka broker.

**Simulated (implemented)** — A Python sensor simulator generates realistic JSON acceleration events directly to Kafka, enabling full end-to-end testing without physical hardware.

---

## Services

| Service | Image / Framework | Role |
|---------|-------------------|------|
| Sensor Simulator | `python:3.11-slim` | Generates synthetic acceleration events |
| Kafka Broker | `bitnami/kafka:3.5.1` | Real-time event streaming |
| Zookeeper | `bitnami/zookeeper` | Kafka coordination |
| Fall Detector | `FastAPI` + `uvicorn` | Consumes events, detects falls, triggers alerts |
| MongoDB | `mongo` | Structured event storage (audit/history) |
| InfluxDB | `influxdb:1.8` | Time-series storage for Grafana |
| Grafana | `grafana/grafana` | Real-time dashboard with threshold alerts |
| Alert System | Twilio + SMTP | SMS and email notifications |

---

## Fall Detection Logic

The sensor simulator generates acceleration values (in g) according to literature-backed distributions:

- **Normal activity:** 0.2 – 1.9 g (99% of events)
- **Fall event:** 2.0 – 3.0 g (1% probability)

The detector applies a threshold rule: if `acceleration > 2.0g`, a potential fall is flagged and both SMS and email alerts are dispatched.

### Grafana Thresholds

| Color | Threshold | Meaning |
|-------|-----------|---------|
| 🟢 Green | Base | Normal acceleration |
| 🟡 Yellow | ≥ 1.7 g | High but non-critical |
| 🔴 Red | ≥ 2.0 g | Possible fall — alert triggered |

---

## Project Structure

```
/
├── simulator/
│   ├── sensor_simulator.py     # Kafka producer — generates acceleration data
│   └── Dockerfile
├── detector/
│   ├── main.py                 # FastAPI app entry point
│   ├── kafka_consumer.py       # Kafka consumer + fall detection logic
│   ├── database.py             # MongoDB + InfluxDB integration
│   ├── alert.py                # Twilio SMS + SMTP email alerts
│   ├── entrypoint.sh           # Waits for Kafka before starting uvicorn
│   └── Dockerfile
├── docker-compose.yml          # Full service orchestration
├── .env                        # Secrets: Twilio, email credentials (not committed)
└── README.md
```

---

## Quick Start

### Prerequisites

- Docker + Docker Compose
- A Twilio account (trial is sufficient)
- A Gmail account with an app password enabled

### 1. Configure environment variables

Create a `.env` file in the project root:

```env
TWILIO_SID=your_twilio_sid
TWILIO_TOKEN=your_twilio_token
TWILIO_PHONE=+1234567890
NUMERO_DESTINO=+0987654321

EMAIL_USER=your@gmail.com
EMAIL_PASS=your_app_password
EMAIL_DESTINO=recipient@email.com
```

### 2. Start the system

```bash
docker compose up --build
```

### 3. Access Grafana

Open [http://localhost:3000](http://localhost:3000) and configure the InfluxDB data source:

- **URL:** `http://influxdb:8086`
- **Database:** `falls`
- **Measurement:** `acceleration_data`

Create a **Stat** panel per patient using this InfluxQL query:

```sql
SELECT last("acceleration") FROM "acceleration_data"
WHERE ("paciente" = 'patient_1') AND $timeFilter
GROUP BY time($_interval) fill(null)
```

Repeat for `patient_2` and `patient_3`.

---

## Cloud Deployment (Google Cloud)

The system was deployed on a **Google Compute Engine** e2-medium VM:

| Parameter | Value |
|-----------|-------|
| Machine type | e2-medium (2 vCPU, 4 GB RAM) |
| OS | Ubuntu 22.04 LTS |
| Boot disk | 10 GB balanced persistent disk |
| Architecture | x86-64 |
| HTTP/HTTPS | Enabled |
| Network | Default VPC, ephemeral external IP |

All services run inside Docker containers, mirroring the local compose setup exactly.

---

## Results

The system was validated end-to-end across three simulated patients:

- Real-time acceleration data streamed continuously via Kafka
- Grafana dashboards refreshed every 5 seconds with color-coded risk indicators
- Automatic SMS alerts delivered via Twilio when acceleration exceeded 2g
- Email alerts delivered via SMTP with patient ID and acceleration value
- All events stored durably in both MongoDB and InfluxDB

---

## Future Work

- **Predictive detection** — Train an ML model on historical data to identify pre-fall risk patterns before a fall occurs
- **Real sensor integration** — Replace the simulator with actual wearable data from smartwatches or smartphones
- **Patient-specific thresholds** — Calibrate detection thresholds per patient based on age, mobility profile, or medical history

---

## References

1. WHO – *Falls Fact Sheet* (2021)
2. WHO – *Patient Safety Fact Sheet* (2023)
3. Biancone et al. – *Multi-Sensor Fall Detection for Smartphones*, Biomedical J. of Scientific and Technical Research (2021)
4. Harari et al. – *A smartphone-based online system for fall detection with alert notifications*, J. of Neuroengineering and Rehabilitation (2021)
