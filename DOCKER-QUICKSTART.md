# F5 AI Agents Flowise - Docker Quick Start Guide

This guide will help you get the F5-branded Flowise running with PostgreSQL (with pgvector) and Ollama using Docker.

## Prerequisites

- Docker Engine 20.10+ or Docker Desktop
- Docker Compose V2+
- Git
- At least 4GB of available RAM
- 10GB of free disk space

## Quick Start (Recommended)

### 1. Clone the F5 Flowise Repository

```bash
git clone https://github.com/maximuslee1226/Flowise.git
cd Flowise
```

### 2. Pull Required Docker Images

Pull the PostgreSQL and Ollama images:

```bash
# Pull PostgreSQL with pgvector extension
docker pull ankane/pgvector:latest

# Pull Ollama for local LLM inference
docker pull ollama/ollama:latest
```

> **Note**: The F5-branded Flowise image will be built from source automatically in the next step, ensuring you get the latest F5 branding and customizations.

### 3. Build and Start All Services

Build the F5-branded Flowise image and start all services:

```bash
docker compose -f docker-compose-complete.yml up -d --build
```

This will:
- Build the F5-branded Flowise image from source
- Start PostgreSQL with pgvector
- Start Ollama for local LLM inference
- Start the F5 Flowise application

### 4. Access Flowise

Open your browser and navigate to:
```
http://localhost:3000
```

### 5. Ollama Models

The Docker setup automatically pulls the embedding model on startup:

**Automatically Installed:**
- `nomic-embed-text` - Embedding model (pulled automatically on first start)

**To pull additional models (optional):**

```bash
# Pull a model (e.g., llama3.2)
docker exec -it f5-flowise-ollama ollama pull llama3.2

# Or pull a smaller model for testing
docker exec -it f5-flowise-ollama ollama pull llama3.2:1b

# List available models
docker exec -it f5-flowise-ollama ollama list
```

Popular models to try:
- `llama3.2` - Meta's latest Llama model
- `llama3.2:1b` - Smaller 1B parameter version
- `mistral` - Mistral AI's model
- `codellama` - Code-specialized model
- `phi3` - Microsoft's compact model

**Note:** The first startup may take a few minutes while the embedding model downloads (~274MB).

### 6. Configure Ollama in Flowise

Once inside Flowise:
1. Navigate to the chatflow builder
2. Add an "Ollama" node
3. Set the Base URL to: `http://ollama:11434`
4. Select your pulled model from the dropdown

## What You Get

- **F5-branded Flowise UI** with custom branding and messaging
- **PostgreSQL** with pgvector extension for vector embeddings
- **Ollama** for local LLM inference (no API keys required)
- All services networked together and ready to work
- Persistent data storage

## Service Details

### Container Names
- **Flowise**: `f5-flowise-app` (F5-branded build)
- **PostgreSQL**: `f5-flowise-postgres`
- **Ollama**: `f5-flowise-ollama`

### Docker Image
- **F5 Flowise**: `f5-flowise:latest` (built from source)

### Ports
- **Flowise UI**: http://localhost:3000
- **PostgreSQL**: localhost:5432
- **Ollama API**: http://localhost:11434

### Network Architecture

All three containers are connected to a shared Docker bridge network:

- **Network Name**: `f5_flowise_network`
- **Subnet**: `172.28.0.0/16`
- **Gateway**: `172.28.0.1`
- **Driver**: bridge

#### How Containers Communicate:

The shared network enables seamless communication between services:

1. **Flowise → PostgreSQL**:
   - Uses hostname: `postgres:5432`
   - Connection string: `postgresql://flowise:flowisepassword@postgres:5432/flowise`

2. **Flowise → Ollama**:
   - Uses hostname: `ollama:11434`
   - Base URL in Flowise: `http://ollama:11434`

3. **Service Discovery**:
   - Each container can reach others using their service name (postgres, ollama, flowise)
   - No need for IP addresses - Docker DNS handles name resolution
   - All traffic stays within the private Docker network

#### Network Inspection:

```bash
# View network details
docker network inspect f5_flowise_network

# List all containers on the network
docker network inspect f5_flowise_network --format='{{range .Containers}}{{.Name}} {{end}}'

# Test connectivity from Flowise to PostgreSQL
docker exec -it f5-flowise-app ping postgres

# Test connectivity from Flowise to Ollama
docker exec -it f5-flowise-app wget -O- http://ollama:11434/api/tags
```

### Database Credentials (Default)
- **Database**: flowise
- **User**: flowise
- **Password**: flowisepassword
- **Host**: postgres (internal), localhost (external)
- **Port**: 5432

> **Security Note**: Change the default database password in production! Edit the `docker-compose-complete.yml` file to set a secure password.

## Common Operations

### View Logs

```bash
# All services
docker compose -f docker-compose-complete.yml logs -f

# Flowise only
docker logs -f f5-flowise-app

# PostgreSQL only
docker logs -f f5-flowise-postgres

# Ollama only
docker logs -f f5-flowise-ollama
```

### Stop Services

```bash
docker compose -f docker-compose-complete.yml stop
```

### Start Services Again

```bash
docker compose -f docker-compose-complete.yml start
```

### Stop and Remove Containers

```bash
docker compose -f docker-compose-complete.yml down
```

### Stop and Remove Everything (Including Data)

> **Warning**: This will delete all your data!

```bash
docker compose -f docker-compose-complete.yml down -v
```

### Restart a Single Service

```bash
# Restart Flowise
docker restart f5-flowise-app

# Restart PostgreSQL
docker restart f5-flowise-postgres

# Restart Ollama
docker restart f5-flowise-ollama
```

## Data Persistence

All data is stored in named Docker volumes:
- `f5_flowise_data` - Flowise application data
- `f5_postgres_data` - PostgreSQL database
- `f5_ollama_data` - Ollama models and data

To back up your data, you can use Docker volume commands or the standard PostgreSQL backup tools.

## Environment Configuration

To customize Flowise configuration, edit the `docker-compose-complete.yml` file:

### Enable Authentication

Uncomment these lines in the `flowise` service:
```yaml
- FLOWISE_USERNAME=admin
- FLOWISE_PASSWORD=changeme
```

### Configure CORS

Uncomment and modify:
```yaml
- CORS_ORIGINS=*
- IFRAME_ORIGINS=*
```

## Troubleshooting

### Port Already in Use

If port 3000, 5432, or 11434 is already in use, edit `docker-compose-complete.yml` and change the port mapping:

```yaml
ports:
  - '3001:3000'  # Change 3000 to 3001 (or any available port)
```

### Flowise Can't Connect to PostgreSQL

Check if PostgreSQL is healthy:
```bash
docker exec -it f5-flowise-postgres pg_isready -U flowise
```

View PostgreSQL logs:
```bash
docker logs f5-flowise-postgres
```

### Ollama Models Not Showing

1. Ensure you've pulled at least one model
2. Check Ollama is running: `docker ps | grep ollama`
3. Test Ollama API: `curl http://localhost:11434/api/tags`

### Reset Everything

To start fresh:
```bash
# Stop and remove everything
docker compose -f docker-compose-complete.yml down -v

# Remove images (optional)
docker rmi f5-flowise:latest ankane/pgvector:latest ollama/ollama:latest

# Rebuild and start again
docker compose -f docker-compose-complete.yml up -d --build
```

### Rebuild F5 Flowise Image

If you've pulled latest changes from the repository and want to rebuild:
```bash
# Rebuild just the Flowise service
docker compose -f docker-compose-complete.yml build flowise

# Or rebuild and restart all services
docker compose -f docker-compose-complete.yml up -d --build
```

## Advanced Configuration

### Use External PostgreSQL

If you have an existing PostgreSQL instance with pgvector:

1. Remove the `postgres` service from `docker-compose-complete.yml`
2. Update Flowise environment variables:
```yaml
- DATABASE_HOST=your-postgres-host
- DATABASE_PORT=5432
- DATABASE_NAME=your-database
- DATABASE_USER=your-user
- DATABASE_PASSWORD=your-password
```

### Use External Ollama

To use a remote Ollama instance:

1. Remove the `ollama` service from `docker-compose-complete.yml`
2. In Flowise, set the Ollama Base URL to your external Ollama instance

## Next Steps

1. **Explore Flowise**: Start building your first AI agent workflow
2. **Try Different Models**: Experiment with various Ollama models
3. **Read Documentation**: Visit [Flowise Docs](https://docs.flowiseai.com/)
4. **Join Community**: [Discord](https://discord.gg/jbaHfsRVBW)

## Additional Resources

- [Ollama Model Library](https://ollama.ai/library)
- [pgvector Documentation](https://github.com/pgvector/pgvector)
- [Flowise Documentation](https://docs.flowiseai.com/)
- [Docker Compose Reference](https://docs.docker.com/compose/)
