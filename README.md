# SBOM Risk Analysis Backend

AI-powered SBOM component risk analysis platform backend built with Node.js, Express, PostgreSQL, Qdrant, and OpenAI.

## 🏗️ Architecture

This backend processes Software Bill of Materials (SBOM) files and provides AI-powered risk analysis for each component.

### Tech Stack
- **Node.js** with Express - REST API server
- **PostgreSQL** - Report caching database
- **Qdrant** - Vector database for security news/insights
- **OpenAI API** - LLM-based risk analysis
- **External APIs**: npm registry, npms.io

### Project Structure

```
backend/
├── server.js              # Express server entry point
├── routes/
│   └── sbom.js           # SBOM upload and analysis endpoints
├── services/
│   ├── sbomProcessor.js  # Main SBOM processing pipeline
│   └── npmRegistry.js    # npm & npms.io API clients
├── db/
│   ├── connection.js     # PostgreSQL connection pool
│   └── queries.js        # Database queries for caching
├── rag/
│   └── qdrantClient.js   # Qdrant vector DB client
├── llm/
│   └── riskAnalyzer.js   # OpenAI risk analysis
└── uploads/              # Temporary SBOM file storage
```

## 🚀 Getting Started

### Prerequisites

1. **Node.js** (v18 or higher)
2. **PostgreSQL** (v12 or higher)
3. **Qdrant** (optional, for RAG features)
4. **OpenAI API Key**

### Installation

1. Install dependencies:
```bash
npm install
```

2. Set up environment variables:
```bash
cp .env.example .env
```

Edit `.env` with your configuration:
```env
# Server
PORT=3000
NODE_ENV=development

# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_NAME=sbom_risk_analysis
DB_USER=postgres
DB_PASSWORD=your_password

# Qdrant (optional)
QDRANT_URL=http://localhost:6333
QDRANT_COLLECTION_NAME=security_news

# OpenAI
OPENAI_API_KEY=your_openai_api_key
OPENAI_MODEL=gpt-4-turbo-preview

# Cache
CACHE_EXPIRY_DAYS=7
```

3. Set up PostgreSQL database:
```bash
# Create database
createdb sbom_risk_analysis

# Tables will be auto-created on first run
```

4. (Optional) Set up Qdrant:
```bash
# Using Docker
docker run -p 6333:6333 qdrant/qdrant
```

### Running the Server

**Development mode:**
```bash
npm run dev
```

**Production mode:**
```bash
npm start
```

Server will start on `http://localhost:3000`

## 📡 API Endpoints

### POST /api/sbom/upload
Upload and analyze an SBOM file.

**Request:**
- Method: `POST`
- Content-Type: `multipart/form-data`
- Body: `sbom` (file, JSON format)

**Example with curl:**
```bash
curl -X POST http://localhost:3000/api/sbom/upload \
  -F "sbom=@path/to/sbom.json"
```

**Response:**
```json
{
  "success": true,
  "report": {
    "sbom_metadata": {
      "format": "CycloneDX",
      "total_components": 25
    },
    "aggregate_risk": {
      "high_risk_count": 2,
      "medium_risk_count": 5,
      "safe_count": 18,
      "average_risk_score": 32.5,
      "overall_classification": "Medium Risk"
    },
    "component_reports": [...]
  },
  "processing_time_seconds": 45.23
}
```

### POST /api/sbom/analyze
Analyze SBOM from JSON body (no file upload).

**Request:**
- Method: `POST`
- Content-Type: `application/json`
- Body: SBOM JSON object

**Example:**
```bash
curl -X POST http://localhost:3000/api/sbom/analyze \
  -H "Content-Type: application/json" \
  -d @sbom.json
```

### GET /health
Health check endpoint.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2026-01-26T10:30:00.000Z",
  "environment": "development"
}
```

## 🔄 Analysis Pipeline

For each component in the SBOM:

1. **Cache Check** - Query PostgreSQL for existing report (within 7 days)
2. **Fetch Metadata** - Get package info from npm registry
3. **Fetch Statistics** - Get quality metrics from npms.io
4. **Query News** - Search Qdrant for security-related news
5. **Build Feature Object** - Combine all data into structured format
6. **LLM Analysis** - Send to OpenAI for risk scoring
7. **Store Report** - Cache result in PostgreSQL
8. **Aggregate Results** - Combine all component reports

## 🎯 Feature Object Structure

```javascript
{
  component_name: "express",
  version: "4.18.2",
  created_date: "2023-01-15T10:00:00Z",
  modified_date: "2023-11-20T14:30:00Z",
  deprecated: false,
  dependents_count: 50000,
  stars: 60000,
  forks: 10000,
  rank: "Excellent",
  vulnerabilities: [],
  news_signals: [
    {
      text: "Express 4.18.2 security patch released...",
      source: "npm-blog",
      date: "2023-11-20"
    }
  ]
}
```

## 📊 Risk Classification

- **Safe** (0-39): Well-maintained, widely adopted, no significant risks
- **Medium Risk** (40-69): Some concerns, requires monitoring
- **High Risk** (70-100): Deprecated, vulnerabilities, or low adoption

## 🛠️ Database Schema

```sql
CREATE TABLE component_reports (
  id SERIAL PRIMARY KEY,
  component_name VARCHAR(255) NOT NULL,
  version VARCHAR(100) NOT NULL,
  risk_score INTEGER,
  risk_classification VARCHAR(50),
  reasoning TEXT,
  recommendation VARCHAR(50),
  metadata JSONB,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(component_name, version)
);

CREATE INDEX idx_component_lookup 
ON component_reports(component_name, version, created_at DESC);
```

## 🧪 Testing

Create a test SBOM file (`test-sbom.json`):

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "components": [
    {
      "name": "express",
      "version": "4.18.2",
      "type": "library"
    },
    {
      "name": "axios",
      "version": "1.6.0",
      "type": "library"
    }
  ]
}
```

Test the endpoint:
```bash
curl -X POST http://localhost:3000/api/sbom/upload \
  -F "sbom=@test-sbom.json"
```

## 📝 Supported SBOM Formats

- **CycloneDX** (recommended)
- **SPDX**
- **Simple JSON** (array of components)
- **package.json** (npm dependencies)

## 🔧 Configuration

### Cache Expiry
Reports are cached for 7 days by default. Adjust via `CACHE_EXPIRY_DAYS` environment variable.

### LLM Model
Change the OpenAI model via `OPENAI_MODEL` (e.g., `gpt-4`, `gpt-3.5-turbo`).

### Performance
- Components are processed sequentially to avoid rate limits
- Consider implementing parallel processing for large SBOMs
- Cache hit rate improves with repeated analyses

## 🚨 Error Handling

The system includes comprehensive error handling:
- Fallback heuristic analysis if LLM fails
- Graceful degradation if Qdrant is unavailable
- Detailed error logging for debugging

## 📈 Future Enhancements

- [ ] CVE database integration for vulnerability scanning
- [ ] License compliance checking
- [ ] Dependency graph visualization
- [ ] Webhook notifications for new risks
- [ ] Multi-ecosystem support (PyPI, Maven, etc.)

## 📄 License

ISC

## 🤝 Contributing

Contributions welcome! Please follow standard Git workflow.

## 📞 Support

For issues or questions, please open a GitHub issue.
