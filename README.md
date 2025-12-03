# [UA] Electricity Consumption Prediction Service

Microservices-based architecture designed to predict electricity consumption for specific building meters. This system orchestrates data acquisition from external providers (Kunna) and processes it through a Multi-Layer Perceptron (MLP) model to generate somewhat accurate usage forecasts.

## Architecture Overview

The system is composed of three microservices, managed as Git submodules within this repository:

1.  **Orchestrator:** The central controller that manages the workflow between data acquisition and prediction.
2.  **Acquisitor:** Interfaces with the external Kunna API to retrieve raw meter data and format it into the features used for prediction.
3.  **Predictor:** Loads a pre-trained MLP model to generate predictions based on provided features.

All services are containerized via Docker and utilize MongoDB for caching and auditing.

## Getting Started

### Prerequisites

* Git
* Docker & Docker Compose

### Installation

1.  **Clone the repository and its submodules:**

Simply do:

```bash
git clone <repo_url>
cd <repo_name>
git submodule update --init --recursive
```

2.  **Configure Environment Variables:**

Ensure valid `.env`, if running manually, or `.env.docker`, if running with docker-compose, files exist in the service directories (see [Configuration](#configuration) below).

You can, for example, do:

```bash
for svc in acquisitor predictor orquestrator; do
    cp "./${svc}/.env.docker.example" "./${svc}/.env.docker";
done

sed -i "s/KUNNA_TOKEN=/KUNNA_TOKEN='<token>'" ./acquisitor/.env.docker
```

3.  **Start the System:**

Using the provided script:

```bash
./scripts/start.sh
```

## Microservices API References

### 1. Orchestrator API Endpoints

**Role:** Coordinates the prediction flow. It requests data from the Acquisitor and forwards it to the Predictor.

| Path | Method | Description | Payload / Response | Default |
| :--- | :--- | :--- | :--- | :--- |
| `/health` | `GET` | Check the current status of the service. | **Response:** `{ "service": "orquestrator", "status": "ok" }` | N/A |
| `/run` | `POST` | Triggers the prediction pipeline for a list of dates. | **Payload:** `{ "dates": ["YYYY-MM-DD", ...] }` | Current date (Today) |

### 2. Acquisitor API Endpoints

**Role:** Connects to the Kunna API to fetch meter data for a specific meter and date and transforms it into the input features required by the model.

| Path | Method | Description | Payload / Response | Default |
| :--- | :--- | :--- | :--- | :--- |
| `/health` | `GET` | Check the current status of the service. | **Response:** `{ "service": "acquisitor", "status": "ok" }` | N/A |
| `/data` | `POST` | Retrieves features for a specific date. | **Payload:** `{ "date": "YYYY-MM-DD" }` | Current date (Today) |

### 3. Predictor API Endpoints

**Role:** Hosts the MLP model and performs inference.

| Path | Method | Description | Payload / Response | Default |
| :--- | :--- | :--- | :--- | :--- |
| `/health` | `GET` | Check the current status of the service. | **Response:** `{ "service": "predictor", "status": "ok" }` | N/A |
| `/ready` | `GET` | Checks if the MLP model is loaded and ready for inference. | **Response:** `{ "ready": true/false, "modelVersion": "vX.X", [message] }` | N/A |
| `/predict` | `POST` | Generates a prediction based on input features (current model uses 7 features). | **Payload:** `{ "features": [Number, ...], "meta": { "featureCount": Number } }` | N/A |

## Data Management & Utility Interfaces

### Web Interface (`/app`)
Each service exposes an optional lightweight web interface for manual testing.
* **Functionality:** Displays health status and provides a way to manually trigger `/run`, `/data`, or `/predict` endpoints.
* **Configuration:** Can be enabled/disabled via the `EXPOSE_APP` environment variable.

### Caching & Auditing (`/db`)
All services persist data to MongoDB to optimize performance and ensure traceability.

* **Cache (`POST /db/cache`):**
    * **TTL:** 1 Hour.
    * **Behavior:** If a request is repeated within the TTL, the service returns cached data instead of reprocessing. The Orchestrator caches results individually per date.
* **Audit (`POST /db/audit`):**
    * **TTL:** 1 Month.
    * **Purpose:** Debugging and historical tracking.
    * **Indexing:**
        * **Acquisitor:** Indexed by `dataId`.
        * **Predictor:** Indexed by `predictionId`.
        * **Orchestrator:** Indexed by `correlationId` (links `dataId` to `predictionId`).

## Configuration

Each service is configured via environment variables.

### Acquisitor Variables

| Variable | Description | Default |
| :--- | :--- | :--- |
| `SERVICE` | Internal service identifier | `acquisitor` |
| `LOG_LEVEL` | Logging verbosity (info or error) | `info` |
| `PORT` | Service internal port | `3001` |
| `ADDR` | Host address | `localhost` |
| `MONGO_URI` | MongoDB Connection String | `mongodb://localhost:27017` |
| `EXPOSE_DB` | Expose DB endpoints | `false` |
| `KUNNA_URL` | Kunna API Endpoint | `https://openapi.kunna.es` |
| `KUNNA_TOKEN` | API Authorization Token | `[SECRET]` |
| `KUNNA_METER` | Target Building Meter ID | `MLU00360002` |
| `EXPOSE_APP` | Enable Web UI | `false` |

### Predictor Variables

| Variable | Description | Default/Example |
| :--- | :--- | :--- |
| `SERVICE` | Internal service identifier | `predictor` |
| `LOG_LEVEL` | Logging verbosity (info or error) | `info` |
| `PORT` | Service internal port | `3002` |
| `ADDR` | Host address | `localhost` |
| `MONGO_URI` | MongoDB Connection String | `mongodb://localhost:27017` |
| `EXPOSE_DB` | Expose DB endpoints | `false` |
| `MODEL_VERSION`| Current ML Model Tag | `v1.0` |
| `EXPOSE_APP` | Enable Web UI | `false` |

### Orchestrator Variables

| Variable | Description | Default/Example |
| :--- | :--- | :--- |
| `SERVICE` | Internal service identifier | `orquestrator` |
| `LOG_LEVEL` | Logging verbosity (info or error) | `info` |
| `PORT` | Service internal port | `8080` |
| `ADDR` | Host address | `localhost` |
| `ACQUISITOR_URI`| Address of Acquisitor Service | `http://localhost:3001` |
| `PREDICTOR_URI` | Address of Predictor Service | `http://localhost:3002` |
| `MONGO_URI` | MongoDB Connection String | `mongodb://localhost:27017` |
| `EXPOSE_DB` | Expose DB endpoints | `false` |
| `EXPOSE_APP` | Enable Web UI | `false` |