# Impact Story Generator Agent - Project Overview

## Executive Summary

The Impact Story Generator Agent transforms raw program data (outcomes metrics, beneficiary information) into compelling narratives optimized for multiple audiences: annual reports, grant proposals, board presentations, and social media campaigns. 

**Key Capability**: One data input → Six output formats (annual report, social posts, PDF deck, board deck outline, grant narrative, newsletter)

**Architecture**: Async Python backend with LLM integration (OpenAI) for intelligent narrative generation.

**Status**: Production-ready MVP with optional cloud scaling capabilities.

---

## 1. System Architecture

### High-Level Flow

```
Program Setup
    ↓
[Data Ingestion Layer] ← CSV outcomes, Beneficiary JSON, API sources
    ↓
[Story Generation Layer] ← OpenAI LLM (or template fallback)
    ↓
[Output Generation Layer] ← Multi-format generation
    ↓
[REST API] ← FastAPI endpoints for all operations
```

### Component Breakdown

#### 1.1 Data Ingestion Service
**File**: `app/data_ingestion.py`

Handles three data sources:
1. **CSV Outcomes** - Metrics like "Students Served: 250", "Graduation Rate: 92%"
2. **Beneficiary JSON** - Individual stories and impact data
3. **External APIs** - Async HTTP fetching with Bearer token support

**Key Methods**:
- `ingest_csv(csv_string)` → List of outcome dicts
- `ingest_beneficiary_data(json_list)` → List of beneficiary dicts (validated)
- `fetch_api_data(url, bearer_token)` → Raw response data

**Validation**: Each row validates required fields; errors logged without stopping pipeline.

#### 1.2 Story Generation Service
**File**: `app/llm_service.py`

Generates narratives via OpenAI GPT API with template fallbacks:

1. **Individual Stories** (2-3 paragraphs)
   - Input: Name, role, program, impact description
   - Output: Personalized narrative (e.g., "Maria's Journey from Homelessness to Hope")
   - Process: GPT prompt → narrative OR template fallback

2. **Program Summary** (1 paragraph)
   - Input: Program name, outcome metrics, key achievements
   - Output: Executive summary for annual reports
   - Process: GPT-3.5-turbo structured prompt

3. **Social Posts** (Platform-specific)
   - Input: Story, platform (Twitter/LinkedIn/Instagram/Facebook)
   - Output: Character-limited, hashtag-optimized posts
   - Formats: Twitter (280 char), LinkedIn (3000 char), Instagram (2200 char)

4. **Grant Narratives** (Section-specific)
   - Input: Section type (need/approach/impact/evaluation/sustainability), story content
   - Output: Proposal-ready narrative sections
   - Use: Directly embed in grant applications

**Key Methods**:
- `generate_individual_story(name, role, program_name, impact_description)`
- `generate_program_summary(program_name, outcomes)`
- `generate_social_posts(story_title, story_content, platforms)`
- `generate_grant_narrative(section, program_name, story_content, outcomes)`

**Fallback Strategy**: If OpenAI API unavailable, uses template stories with dynamic personalization.

#### 1.3 Output Generator Service
**File**: `app/output_generator.py`

Produces six output formats from a single ImpactStory:

1. **Annual Report Page** (Markdown)
   - Structure: Title → Programs → Top Stories → Metrics
   - Usage: Embedded in nonprofit annual reports
   - Format: Markdown (easy to convert to PDF/HTML)

2. **Social Content Pack** (JSON)
   - Structure: {twitter: {...}, linkedin: {...}, instagram: {...}, facebook: {...}}
   - Usage: Batch social media scheduling
   - Includes: Post text, hashtags, optimal post times, engagement targets

3. **PDF Annual Report** (ReportLab)
   - Structure: Title page → Program section → Story pages
   - Usage: Professional downloadable report
   - Requires: `reportlab` (optional dependency)
   - Fallback: Links to non-PDF version if reportlab unavailable

4. **Board Deck Outline** (JSON/Structured text)
   - Slides: 
     1. Impact Overview (mission + key metrics)
     2. Program Highlights (top 3 stories)
     3. Financials & Efficiency
     4. Next Steps & Fundraising Needs
   - Usage: PowerPoint/Google Slides creation
   - Format: Bullet points + speaker notes

5. **Grant Narrative Document** (Multi-section)
   - Sections:
     - Statement of Need
     - Program Approach
     - Anticipated Impact
     - Evaluation Plan
     - Sustainability Plan
   - Usage: Proposal component ready for submission
   - Format: Word-doc ready text

6. **Newsletter Content** (HTML)
   - Structure: Featured story → Program updates → Donor impact → CTA
   - Usage: Email distribution (MailChimp, SendGrid compatible)
   - Responsive: Mobile-friendly HTML

**Key Methods**:
- `generate_annual_report_page(program_summaries, top_stories)`
- `generate_social_content_pack(stories)`
- `generate_pdf_annual_report(programs, stories)`
- `generate_board_deck_outline(metrics, top_stories)`
- `generate_grant_narrative_document(need, approach, impact, evaluation, sustainability)`
- `generate_newsletter_content(featured_story, program_updates, cta)`

#### 1.4 Impact Story Agent (Orchestrator)
**File**: `app/agent.py`

Coordinates entire workflow with error handling and progress tracking:

**Methods**:
1. `ingest_program_data(program_id, csv_outcomes, beneficiary_data)`
   - Loads data into database
   - Returns: # outcomes created, # beneficiaries created

2. `generate_individual_stories(program_id, max_stories=5)`
   - Selects top beneficiaries by impact potential
   - Generates stories for each
   - Returns: List of Story objects with metadata

3. `generate_program_summary(program_id)`
   - Creates one ImpactStory of type PROGRAMMATIC
   - Aggregates outcomes and lessons learned
   - Returns: Story object with full content

4. `generate_outputs(story_id, formats=[])`
   - Selects story and generates requested formats
   - Parallelizes output generation where possible
   - Returns: Dict mapping format → output content

5. `run_full_cycle(program_id, csv_outcomes, beneficiary_data)`
   - Executes: ingest → individual stories → summary → outputs
   - Single method for end-to-end operation
   - Returns: Comprehensive result dict with all stages

---

## 2. Database Schema

### Models

#### Program
```python
class Program(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
    description: str = ""
    budget: float | None = None
    created_at: datetime = Field(default_factory=utcnow)
    
    # Relationships
    outcomes: List["Outcome"] = Relationship(back_populates="program")
    beneficiaries: List["Beneficiary"] = Relationship(back_populates="program")
    stories: List["ImpactStory"] = Relationship(back_populates="program")
```

#### Outcome
```python
class Outcome(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    program_id: int = Field(foreign_key="program.id")
    metric_name: str  # "Graduation Rate", "Jobs Created"
    actual_value: float
    target_value: float | None = None
    unit: str  # "%", "students", "jobs"
    description: str = ""
    created_at: datetime = Field(default_factory=utcnow)
    
    # Relationship
    program: Program = Relationship(back_populates="outcomes")
```

#### Beneficiary
```python
class Beneficiary(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    program_id: int = Field(foreign_key="program.id")
    name: str | None = None
    role: str | None = None  # "Student", "Volunteer"
    location: str | None = None
    impact_area: str | None = None
    impact_description: str | None = None
    created_at: datetime = Field(default_factory=utcnow)
    
    # Relationship
    program: Program = Relationship(back_populates="beneficiaries")
```

#### ImpactStory
```python
class ImpactStory(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    program_id: int = Field(foreign_key="program.id")
    title: str
    content: str  # Full narrative (2-5 paragraphs)
    story_type: str  # INDIVIDUAL, ORGANIZATIONAL, PROGRAMMATIC, THEMATIC
    status: str = "draft"  # draft, approved, published
    beneficiary_id: int | None = Field(foreign_key="beneficiary.id", default=None)
    created_at: datetime = Field(default_factory=utcnow)
    updated_at: datetime = Field(default_factory=utcnow)
    
    # Relationships
    program: Program = Relationship(back_populates="stories")
    outputs: List["StoryOutput"] = Relationship(back_populates="story")
```

#### StoryOutput
```python
class StoryOutput(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    story_id: int = Field(foreign_key="impactstory.id")
    format: str  # ANNUAL_REPORT, SOCIAL_POST, BOARD_DECK, etc.
    content: str | dict  # JSON or markdown content
    created_at: datetime = Field(default_factory=utcnow)
    
    # Relationship
    story: ImpactStory = Relationship(back_populates="outputs")
```

### Relationships
```
Program (1) ← → (Many) Outcome
Program (1) ← → (Many) Beneficiary
Program (1) ← → (Many) ImpactStory
          ↓
       Beneficiary (1) ← → (Many) ImpactStory

ImpactStory (1) ← → (Many) StoryOutput
```

---

## 3. API Endpoints

### Programs
- `POST /programs` - Create
- `GET /programs` - List all
- `GET /programs/{id}` - Get by ID

### Outcomes
- `POST /outcomes` - Create
- `GET /outcomes?program_id=X` - List by program

### Beneficiaries
- `POST /beneficiaries` - Create
- `GET /beneficiaries?program_id=X` - List by program

### Stories
- `GET /stories` - List (filterable by program_id, status)
- `GET /stories/{id}` - Get by ID
- `PATCH /stories/{id}` - Update status

### Outputs
- `GET /outputs` - List (filterable by story_id, format)
- `GET /outputs/{id}` - Get by ID

### Agent Operations
- `POST /agent/ingest/{program_id}` - Load data
- `POST /agent/generate-stories/{program_id}` - Create individual narratives
- `POST /agent/generate-summary/{program_id}` - Create program summary
- `POST /agent/generate-outputs/{story_id}` - Multi-format output
- `POST /agent/full-cycle/{program_id}` - End-to-end operation

---

## 4. Use Cases

### Use Case 1: Annual Report Generation
**Scenario**: Nonprofit compiles year-end impact data

1. Data Ingestion
   - CSV: "Students Served: 500, Graduation Rate: 94%"
   - JSON: 15 beneficiary stories
   
2. Story Generation
   - LLM creates 3 individual narratives
   - LLM creates 1 program summary
   
3. Outputs
   - Annual report page (markdown)
   - PDF report (with cover page, metrics, stories)
   - Social posts (6 platforms × 3 stories = 18 posts)
   
4. Result
   - Nonprofit has professional report + social media calendar

### Use Case 2: Grant Application Support
**Scenario**: Foundation requires specific narrative sections

1. Existing Stories
   - Database has 20 published impact stories
   
2. Agent Operation
   - Select top 3 stories by relevance
   - Generate grant-specific narrative sections:
     - "Statement of Need" (using program context)
     - "Our Approach" (from program outcomes)
     - "Anticipated Impact" (from beneficiary stories)
     - "Evaluation Plan" (from metrics)
     - "Sustainability" (from program model)
   
3. Result
   - Grant writer has 5 ready-to-use narrative sections
   - Estimated 10 hours saved per proposal

### Use Case 3: Board Meeting Preparation
**Scenario**: Executive director needs 10-minute presentation

1. Input
   - Select program from database
   - Trigger full cycle with latest data
   
2. Outputs Generated
   - Annual report page (1-page overview)
   - Board deck outline (4 slides with speaker notes)
   - 3 featured stories (for storytelling)
   
3. Result
   - Full presentation outline ready in 2 minutes

### Use Case 4: Social Media Content Calendar
**Scenario**: Communications team needs 30 days of content

1. Batch Operation
   - Pull 15 published stories from database
   - Generate social posts for each
   - Specify platforms: Twitter, LinkedIn, Instagram, Facebook
   
2. Output
   - 60 unique posts (4 platforms × 15 stories)
   - Platform-optimized length & formatting
   - Suggested hashtags & posting times
   
3. Result
   - 1 month of content calendar, auto-populated

### Use Case 5: Donor Cultivation
**Scenario**: Major donor meeting prep

1. Agent Operation
   - Input: Specific program the donor funded
   - Output: Personalized annual report + story PDF
   
2. Customization
   - Watermark report with donor name
   - Highlight metrics related to donor's interest area
   - Include personal story of someone donor's gift helped
   
3. Result
   - Customized, compelling thank-you material (5 min prep)

---

## 5. Implementation Details

### LLM Integration (OpenAI)

**Model**: GPT-3.5-turbo (default) or GPT-4 (premium)

**Prompts**:
1. Individual story template (~150-250 words)
2. Program summary (~75-100 words)
3. Social posts (4 platforms with character limits)
4. Grant narrative (section-aware prompts)

**Token Estimation**:
- Individual story: ~200-500 tokens
- Program summary: ~100-200 tokens
- Social pack (4 posts): ~150-300 tokens
- Grant narrative (5 sections): ~500-1000 tokens
- **Per-program cost**: ~$0.50-2.00 USD

**Fallback**: If API fails or key missing, template-based system provides instant results.

### PDF Generation (ReportLab)

**Features**:
- Multi-page reports
- Title pages with logos
- Metrics summary pages
- Story pages with formatting
- Page numbers & footer branding

**Optional Dependency**: Can be installed or skipped without breaking system.

### Async Database

**Driver**: asyncpg (PostgreSQL) or aiosqlite (SQLite)

**Schema Creation**:
```python
from app.db import init_db
init_db()  # Creates all tables
```

**Transaction Pattern**:
```python
async with get_async_session() as session:
    story = ImpactStory(...)
    session.add(story)
    await session.commit()
```

---

## 6. Performance Characteristics

| Operation | Time | Notes |
|-----------|------|-------|
| CSV Ingestion (1000 rows) | <1 sec | In-memory parsing |
| Individual Story Generation | 3-5 sec | OpenAI API call |
| Program Summary | 2-3 sec | Smaller LLM request |
| Social Posts (4 platforms) | 5-10 sec | 4 parallel API calls |
| PDF Generation | 2-5 sec | ReportLab processing |
| Full Cycle (5 stories + outputs) | 30-60 sec | Parallelized where possible |

**Optimization**: 
- Async/await throughout
- LLM calls parallelized
- Database indexed on foreign keys
- Optional caching layer (Redis future)

---

## 7. Deployment Options

### Option 1: Local Development
```bash
pip install -r requirements.txt
python -c "from app.db import init_db; init_db()"
uvicorn app.main:app --reload
```

### Option 2: Docker Container
```dockerfile
FROM python:3.9-slim
WORKDIR /app
RUN pip install -r requirements.txt
COPY app/ app/
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Option 3: Cloud Deployment (GCP Cloud Run)
```yaml
service: impact-story-agent
runtime: python39
entrypoint: uvicorn app.main:app --host 0.0.0.0 --port 8080
env_variables:
  DATABASE_URL: postgresql+asyncpg://...
  OPENAI_API_KEY: sk-...
```

### Option 4: Serverless (AWS Lambda)
- Wrapper: Mangum for ASGI → AWS Lambda
- Database: RDS PostgreSQL (persisted)
- Storage: S3 for PDF artifacts

---

## 8. Security Considerations

### API Security
- **CORS**: Configure per your domain
- **Rate Limiting**: Add slowapi for public endpoints
- **Authentication**: Optional JWT for user-gated access
- **HTTPS**: Required for production (use nginx reverse proxy)

### Database Security
- **Credentials**: Store in environment variables (not code)
- **SQL Injection**: SQLModel + SQLAlchemy protect automatically
- **Backups**: Database auto-backups (PostgreSQL recommended)

### LLM API Security
- **Key Storage**: `.env` file (not in git) or secrets manager
- **API Costs**: Monitor usage with rate limiting

---

## 9. Future Enhancements

### Phase 2 (Q2)
- [ ] Canva API integration for auto-designed graphics
- [ ] Email scheduling (SendGrid/Mailgun integration)
- [ ] Dashboard for story preview/approval workflow

### Phase 3 (Q3)
- [ ] Video script generation
- [ ] Podcast transcript generation
- [ ] Interactive web storybook (Next.js frontend)

### Phase 4 (Q4)
- [ ] AI-powered image sourcing (Unsplash integration)
- [ ] Multi-donor customization
- [ ] Advanced analytics (story engagement tracking)

---

## 10. Troubleshooting & Support

### Database Issues
**Problem**: "table already exists"
**Solution**: Delete `.db` file if using SQLite, or drop schema in PostgreSQL

### LLM Failures
**Problem**: "OpenAI API rate limited"
**Solution**: Implement exponential backoff retry logic (in roadmap)

**Problem**: "OPENAI_API_KEY not set"
**Solution**: System gracefully falls back to template stories (no error)

### PDF Generation Fails
**Problem**: "reportlab is not installed"
**Solution**: Install with `pip install reportlab` OR system returns markdown fallback

### Async Session Management
**Problem**: "asyncio.run() from running loop"
**Solution**: Ensure endpoints use `Depends(get_session)` pattern

---

## 11. Testing Strategy

### Unit Tests
- DataIngestionService: CSV parsing, JSON validation
- StoryGenerationService: Prompt templates, fallback logic
- OutputGenerator: Format-specific output validation

### Integration Tests
- Agent: Full-cycle operation with mock data
- API: Endpoint inputs/outputs with test database
- Database: CRUD operations, relationships

### Performance Tests
- Benchmark: CSV ingestion speed (target: <1 sec for 1000 rows)
- Benchmark: LLM generation (track API latency)
- Load: 100 concurrent story requests

### Test Coverage Target
- Models: 100%
- Services: 95%+
- API: 85%+
- Agent: 80%+

**Run Tests**:
```bash
pytest tests/ -v --cov=app
```

---

## 12. Architecture Decision Records (ADRs)

### ADR-1: Why SQLModel instead of Django ORM?
- **Decision**: Use SQLModel + SQLAlchemy
- **Rationale**: 
  - Full async support (required for performance)
  - SQLite + PostgreSQL flexibility
  - Type safety with Pydantic integration
  - Lighter than Django

### ADR-2: Why OpenAI GPT vs. Open-Source LLM?
- **Decision**: OpenAI GPT-3.5-turbo with template fallback
- **Rationale**:
  - Superior narrative quality
  - No infrastructure costs
  - API-based (no GPU needed)
  - Fallback templates ensure offline operation

### ADR-3: Why Async Everything?
- **Decision**: Async/await throughout (FastAPI, database, HTTP)
- **Rationale**:
  - High I/O concurrency (LLM calls, database)
  - Foundation for scaling
  - Single-threaded (simpler deployment)

---

## 13. Contact & Support

**Repository**: [Link to GitHub repo]
**Maintainer**: [Your Team]
**Docs**: [Link to wiki/docs site]
**Issues**: GitHub Issues
**Email**: support@nonprofit-ai.example.com

---

**Last Updated**: [Current Date]
**Version**: 1.0.0
**Status**: Production Ready
