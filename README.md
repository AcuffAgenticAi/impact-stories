# Impact Story Generator Agent - Backend API

Transform raw program data into compelling impact narratives. Automatically ingest outcomes, beneficiary stories, and generate professional outputs for annual reports, grant proposals, board presentations, and social campaigns.

## 🎯 Overview

The Impact Story Generator Agent orchestrates a complete data-to-narrative pipeline:

1. **Data Ingestion** - Ingest outcomes from CSV, beneficiary details from JSON/API
2. **Story Generation** - Create personalized individual and programmatic narratives via LLM
3. **Multi-Format Output** - Auto-generate annual reports, PDFs, social posts, board decks, grant narratives
4. **Workflow Automation** - Full-cycle operation with async processing

## 🚀 Quick Start

### 1. Install Dependencies

```bash
cd backend
pip install -r requirements.txt
```

### 2. Configure Environment

```bash
cp .env.example .env
# Edit .env with your configuration
```

**Required:**
- `DATABASE_URL` - SQLite or PostgreSQL connection string
- `OPENAI_API_KEY` - For LLM story generation (optional for template mode)

### 3. Run Server

```bash
# Development
uvicorn app.main:app --reload

# Production
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

**API Documentation:** http://localhost:8000/docs

## 📊 Data Models

### Program
- id: int (primary key)
- name: str - Program name
- description: str - Program overview
- budget: float - Budget amount (optional)
- created_at: datetime
- outcomes: List[Outcome] - associated outcome metrics
- beneficiaries: List[Beneficiary] - program participants
- stories: List[ImpactStory] - generated narratives

### Outcome
- id: int
- program_id: int (foreign key)
- metric_name: str - "Students Served", "Graduation Rate", etc.
- actual_value: float - Achievement value
- target_value: float (optional) - Goal value for comparison
- unit: str - "students", "%", "jobs", etc.
- description: str - Metric definition

### Beneficiary
- id: int
- program_id: int
- name: str (optional) - Individual name
- role: str - "Student", "Participant", "Volunteer"
- location: str - Geographic location
- impact_area: str - "Education", "Employment", "Health", etc.
- impact_description: str - Story of change

### ImpactStory
- id: int
- program_id: int
- title: str - Story headline
- content: str - Full narrative (2-5 paragraphs)
- story_type: enum - "INDIVIDUAL", "ORGANIZATIONAL", "PROGRAMMATIC", "THEMATIC"
- status: str - "draft", "approved", "published"
- created_at, updated_at: datetime
- outputs: List[StoryOutput] - generated formats

### StoryOutput
- id: int
- story_id: int
- format: enum - "ANNUAL_REPORT", "SOCIAL_POST", "BOARD_DECK", "GRANT_NARRATIVE", "NEWSLETTER", "BLOG_POST", "VIDEO_SCRIPT"
- content: str/dict - Output content
- created_at: datetime

## 🔌 API Endpoints

### Programs

**Create Program**
```http
POST /programs
?name=Education Initiative
&description=K-12 literacy program
&budget=100000
```

**List Programs**
```http
GET /programs
```

**Get Program**
```http
GET /programs/{program_id}
```

### Outcomes

**Create Outcome**
```http
POST /outcomes
?program_id=1
&metric_name=Students Served
&actual_value=250
&target_value=200
&unit=students
&description=Total K-12 students served annually
```

**List Outcomes**
```http
GET /outcomes?program_id=1
```

### Beneficiaries

**Create Beneficiary**
```http
POST /beneficiaries
?program_id=1
&name=Maria Garcia
&role=Student
&location=Urban Center
&impact_area=Education
&impact_description=Completed high school, enrolled in college
```

**List Beneficiaries**
```http
GET /beneficiaries?program_id=1
```

### Stories

**List Stories**
```http
GET /stories?program_id=1&status=approved
```

**Get Story**
```http
GET /stories/{story_id}
```

**Update Story Status**
```http
PATCH /stories/{story_id}
?status=published
```

### Outputs

**List Outputs**
```http
GET /outputs?story_id=5&format=social_post
```

**Get Output**
```http
GET /outputs/{output_id}
```

### Agent Operations

**Ingest Program Data**
```http
POST /agent/ingest/{program_id}
?outcomes_csv="metric_name,actual_value,target_value,unit,description
Students,250,200,people,K-12 students
Graduation,92,85,%,On-time graduation"
&beneficiary_json="[{\"name\":\"Sarah\",\"role\":\"Student\",\"impact_area\":\"Education\",\"impact_description\":\"Completed program\"}]"
```

**Generate Individual Stories**
```http
POST /agent/generate-stories/{program_id}
?max_stories=5
```

**Generate Program Summary**
```http
POST /agent/generate-summary/{program_id}
```

**Generate Multi-Format Outputs**
```http
POST /agent/generate-outputs/{story_id}
?formats=social_post,board_deck_outline,grant_narrative,newsletter
```

**Full Cycle (End-to-End)**
```http
POST /agent/full-cycle/{program_id}
?outcomes_csv="..."
&beneficiary_json="[...]"
```

Returns:
```json
{
  "success": true,
  "ingestion": {
    "outcomes_created": 3,
    "beneficiaries_created": 8
  },
  "individual_stories": {
    "stories_created": 5,
    "stories": [...]
  },
  "program_summary": {
    "story_id": 10,
    "content": "..."
  },
  "outputs": {
    "annual_report": {...},
    "social_posts": {...},
    "board_deck": {...},
    "grant_narrative": {...}
  }
}
```

## 🤖 Agent Workflow Example

### Full Impact Story Generation Cycle

```python
import httpx
import asyncio

async def main():
    async with httpx.AsyncClient(base_url="http://localhost:8000") as client:
        # 1. Create program
        prog = await client.post("/programs", params={
            "name": "Community Tech Skills",
            "budget": 150000
        })
        program_id = prog.json()["id"]
        
        # 2. Run full cycle with CSV outcomes
        outcomes_csv = """metric_name,actual_value,target_value,unit,description
Students Trained,120,100,people,Bootcamp completions
Employment Rate,92,80,%,6-month placement
Salary Increase,45000,35000,$,Average participant salary increase"""
        
        beneficiary_json = """[
            {
                "name": "James Chen",
                "role": "Bootcamp Graduate",
                "impact_area": "Employment",
                "impact_description": "Stepped into senior developer role at Fortune 500 company, supporting family of 4"
            }
        ]"""
        
        result = await client.post(
            f"/agent/full-cycle/{program_id}",
            params={
                "outcomes_csv": outcomes_csv,
                "beneficiary_json": beneficiary_json
            }
        )
        
        print(result.json())

asyncio.run(main())
```

## 🧪 Testing

Run all tests:
```bash
pytest tests/ -v
```

Run specific test file:
```bash
pytest tests/test_impact_stories_api.py -v
```

Run with coverage:
```bash
pytest tests/ --cov=app --cov-report=html
```

## 📁 Project Structure

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app + endpoints
│   ├── models.py              # SQLModel database schemas
│   ├── db.py                  # Async database config
│   ├── data_ingestion.py      # CSV/JSON/API data loading
│   ├── llm_service.py         # OpenAI integration + LLM prompts
│   ├── output_generator.py    # Multi-format output generation
│   └── agent.py               # ImpactStoryAgent orchestrator
├── requirements.txt            # Python dependencies
├── .env.example                # Configuration template
└── README.md                   # This file
```

## 🔧 Core Services

### DataIngestionService
```python
from app.data_ingestion import DataIngestionService

service = DataIngestionService()

# CSV import
outcomes = service.ingest_csv(csv_string)

# Beneficiary JSON
beneficiaries = service.ingest_beneficiary_data(json_list)

# External API
api_data = await service.fetch_api_data(url, bearer_token)
```

### StoryGenerationService
```python
from app.llm_service import StoryGenerationService

service = StoryGenerationService(openai_api_key="sk-...", org_name="Nonprofit Name")

# Individual stories
story = await service.generate_individual_story(
    name="Maria", role="Student", program_name="Code Academy",
    impact_description="Became software engineer"
)

# Program summary
summary = await service.generate_program_summary(
    program_name="Skills Training", outcomes=[...]
)

# Social media
posts = await service.generate_social_posts(
    story_title="Breaking Barriers",
    story_content="...",
    platforms=["twitter", "linkedin", "instagram"]
)

# Grant sections
narrative = await service.generate_grant_narrative(
    section="impact", program_name="...", story_content="...", outcomes=[...]
)
```

### OutputGenerator
```python
from app.output_generator import OutputGenerator

generator = OutputGenerator(org_name="Nonprofit Name")

# Annual report page (markdown)
report = generator.generate_annual_report_page(
    program_summaries=[...], top_stories=[...]
)

# Social content pack (platform-specific JSON)
pack = generator.generate_social_content_pack(stories=[...])

# PDF annual report
pdf_bytes = generator.generate_pdf_annual_report(
    programs=[...], stories=[...]
)

# Board deck outline
deck = generator.generate_board_deck_outline(
    metrics={...}, top_stories=[...]
)

# Grant narrative (multi-section)
grant = generator.generate_grant_narrative_document(
    need_section="...", approach_section="...", impact_section="...",
    evaluation_section="...", sustainability_section="..."
)

# Newsletter HTML
newsletter = generator.generate_newsletter_content(
    featured_story="...", program_updates=[...]
)
```

### ImpactStoryAgent
```python
from app.agent import ImpactStoryAgent

agent = ImpactStoryAgent(db_session)

# Ingest program data
result = await agent.ingest_program_data(
    program_id=1,
    csv_outcomes="metric_name,actual_value,...",
    beneficiary_data=[...]
)

# Generate stories for beneficiaries
result = await agent.generate_individual_stories(program_id=1, max_stories=5)

# Generate programmatic summary
result = await agent.generate_program_summary(program_id=1)

# Create multi-format outputs
result = await agent.generate_outputs(
    story_id=10,
    formats=["annual_report", "social_post", "board_deck", "grant_narrative"]
)

# Full cycle
result = await agent.run_full_cycle(
    program_id=1,
    csv_outcomes="...",
    beneficiary_data=[...]
)
```

## 🌐 Integration Points

### Canva API (Optional - Planned)
- Auto-design graphics for social posts
- Create branded annual report templates
- Generate video storyboards

### External Data Sources
```python
# Fetch program metrics from external APIs
outcomes = await service.fetch_api_data(
    url="https://api.example.com/program/123/outcomes",
    bearer_token="token_xyz"
)
```

### Email / Scheduling (Future)
- Schedule newsletter distribution
- Trigger story generation on data updates
- Automated report delivery

## 📈 Performance Notes

- **Data Ingestion**: Handles CSV files with 1000+ rows efficiently
- **LLM Generation**: Calls OpenAI API; fallback templates instant
- **PDF Generation**: Optional reportlab dependency (gracefully degraded if missing)
- **Async Support**: All I/O operations are fully async

## 🐛 Troubleshooting

### "OPENAI_API_KEY not set"
The system will use template-based story generation instead of LLM calls. No error thrown.

### "Database locked" (SQLite)
Ensure only one process is writing to the database.

### PDF generation fails
Install reportlab: `pip install reportlab`

### Models not syncing to database
Run: `python -c "from app.db import init_db; init_db()"`

## 🚢 Production Deployment

### Docker
```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app/ app/
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Environment Variables
```bash
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db
OPENAI_API_KEY=sk-xxxxx
ENVIRONMENT=production
DEBUG=false
```

### Uvicorn with Gunicorn
```bash
gunicorn app.main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker
```

## 📄 License

Part of the Nonprofit Agentic Systems suite.
