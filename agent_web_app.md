# Codex Multi-Agent Web Application - Backend Implementation Plan

## Overview

Build a production multi-agent system where 4 specialized agents orchestrate `codex-cli` to perform software development tasks. Agents act as intelligent prompt engineers - they don't write code directly but instead formulate precise prompts and execute them through codex-cli, leveraging your ChatGPT subscription.

**Architecture Principle**: Agents are orchestrators, not workers. They analyze, plan, and prompt codex-cli to do the actual implementation work.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Google Cloud Ubuntu VM                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐    ┌──────────────────────────────────────────────────┐   │
│  │   Frontend   │◄──►│                  FastAPI Backend                  │   │
│  │   (React)    │    │                                                   │   │
│  └──────────────┘    │  ┌─────────────────────────────────────────────┐ │   │
│                      │  │            Agent Orchestrator                │ │   │
│                      │  │                                              │ │   │
│                      │  │   ┌─────────┐  ┌─────────┐  ┌─────────┐     │ │   │
│                      │  │   │ Planner │─►│ Builder │─►│  QA/QC  │     │ │   │
│                      │  │   │  Agent  │  │  Agent  │  │  Agent  │     │ │   │
│                      │  │   └────┬────┘  └────┬────┘  └────┬────┘     │ │   │
│                      │  │        │            │            │          │ │   │
│                      │  │        │     ┌──────┴──────┐     │          │ │   │
│                      │  │        │     │  ProdReady  │◄────┘          │ │   │
│                      │  │        │     │    Agent    │                │ │   │
│                      │  │        │     └──────┬──────┘                │ │   │
│                      │  └────────┼────────────┼───────────────────────┘ │   │
│                      │           │            │                         │   │
│                      │  ┌────────▼────────────▼───────────────────────┐ │   │
│                      │  │         Codex Process Manager               │ │   │
│                      │  │                                              │ │   │
│                      │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐     │ │   │
│                      │  │  │ codex-cli│ │ codex-cli│ │ codex-cli│     │ │   │
│                      │  │  │ (proj 1) │ │ (proj 2) │ │ (proj N) │     │ │   │
│                      │  │  └──────────┘ └──────────┘ └──────────┘     │ │   │
│                      │  └──────────────────────────────────────────────┘ │   │
│                      └──────────────────────────────────────────────────┘   │
│                                          │                                   │
│                      ┌───────────────────▼───────────────────┐              │
│                      │           PostgreSQL Database          │              │
│                      │  ┌─────────┐ ┌─────────┐ ┌─────────┐  │              │
│                      │  │Projects │ │Sessions │ │  Tasks  │  │              │
│                      │  └─────────┘ └─────────┘ └─────────┘  │              │
│                      └───────────────────────────────────────┘              │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Project Workspaces                                 │   │
│  │   /workspaces/project_1/    /workspaces/project_2/    ...            │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Infrastructure Setup

### 1.1 Google Cloud VM Provisioning

**Checklist:**
- [ ] Create Ubuntu 22.04 LTS VM (e2-standard-4 minimum: 4 vCPU, 16GB RAM)
- [ ] Configure firewall rules (ports 22, 80, 443, 8000)
- [ ] Set up static external IP
- [ ] Configure SSH keys for secure access
- [ ] Install system dependencies

```bash
# VM creation command
gcloud compute instances create codex-agents-vm \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=100GB \
  --boot-disk-type=pd-ssd \
  --tags=http-server,https-server

# Firewall rules
gcloud compute firewall-rules create allow-web \
  --allow tcp:80,tcp:443,tcp:8000 \
  --target-tags=http-server,https-server
```

### 1.2 System Dependencies Installation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python 3.11+
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt install python3.11 python3.11-venv python3.11-dev -y

# Install Node.js 20 LTS (for codex-cli)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install PostgreSQL 15
sudo apt install -y postgresql-15 postgresql-contrib-15

# Install Redis (for task queue)
sudo apt install -y redis-server

# Install process management
sudo apt install -y supervisor

# Install uv for Python package management
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 1.3 Codex-CLI Installation & Authentication

```bash
# Install codex-cli globally
npm install -g @openai/codex

# Authenticate with ChatGPT account
codex auth login

# Verify installation
codex --version
codex whoami
```

### 1.4 PostgreSQL Database Setup

```bash
# Create database and user
sudo -u postgres psql << EOF
CREATE USER codex_agent WITH PASSWORD 'your_secure_password';
CREATE DATABASE codex_agents_db OWNER codex_agent;
GRANT ALL PRIVILEGES ON DATABASE codex_agents_db TO codex_agent;
\c codex_agents_db
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
EOF
```

---

## Phase 2: Database Schema Design

### 2.1 Core Tables

**File: `backend/models/database.py`**

```python
import uuid
from datetime import datetime
from enum import Enum
from typing import Optional, List
from sqlalchemy import (
    Column, String, Text, DateTime, ForeignKey,
    JSON, Boolean, Integer, Enum as SQLEnum, Index
)
from sqlalchemy.dialects.postgresql import UUID, JSONB, ARRAY
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, relationship


class Base(DeclarativeBase):
    pass


class TaskStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"


class AgentType(str, Enum):
    PLANNER = "planner"
    BUILDER = "builder"
    QA_QC = "qa_qc"
    PROD_READY = "prod_ready"


# ============== PROJECT MANAGEMENT ==============

class Project(Base):
    """Projects are isolated workspaces for different codebases."""
    __tablename__ = "projects"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(255), nullable=False)
    description = Column(Text)
    workspace_path = Column(String(500), nullable=False, unique=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    is_active = Column(Boolean, default=True)
    settings = Column(JSONB, default=dict)  # Project-specific agent settings

    # Relationships
    sessions = relationship("Session", back_populates="project", cascade="all, delete-orphan")
    files = relationship("ProjectFile", back_populates="project", cascade="all, delete-orphan")

    __table_args__ = (
        Index("idx_project_name", "name"),
        Index("idx_project_active", "is_active"),
    )


class ProjectFile(Base):
    """Uploaded files for project context."""
    __tablename__ = "project_files"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id = Column(UUID(as_uuid=True), ForeignKey("projects.id"), nullable=False)
    filename = Column(String(255), nullable=False)
    file_path = Column(String(500), nullable=False)
    file_type = Column(String(50))
    file_size = Column(Integer)
    content_hash = Column(String(64))  # SHA-256 for deduplication
    uploaded_at = Column(DateTime, default=datetime.utcnow)
    metadata = Column(JSONB, default=dict)

    project = relationship("Project", back_populates="files")


# ============== SESSION & CONVERSATION ==============

class Session(Base):
    """A session represents a complete workflow run (all 4 agents)."""
    __tablename__ = "sessions"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id = Column(UUID(as_uuid=True), ForeignKey("projects.id"), nullable=False)
    title = Column(String(255))
    user_request = Column(Text, nullable=False)  # Original user query
    status = Column(SQLEnum(TaskStatus), default=TaskStatus.PENDING)
    current_agent = Column(SQLEnum(AgentType), nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    completed_at = Column(DateTime, nullable=True)

    # Final outputs
    agents_md_content = Column(Text)  # Generated AGENTS.md
    final_summary = Column(Text)

    # Relationships
    project = relationship("Project", back_populates="sessions")
    conversations = relationship("Conversation", back_populates="session", cascade="all, delete-orphan")
    codex_tasks = relationship("CodexTask", back_populates="session", cascade="all, delete-orphan")

    __table_args__ = (
        Index("idx_session_project", "project_id"),
        Index("idx_session_status", "status"),
        Index("idx_session_created", "created_at"),
    )


class Conversation(Base):
    """Individual agent conversations within a session."""
    __tablename__ = "conversations"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    session_id = Column(UUID(as_uuid=True), ForeignKey("sessions.id"), nullable=False)
    agent_type = Column(SQLEnum(AgentType), nullable=False)
    started_at = Column(DateTime, default=datetime.utcnow)
    ended_at = Column(DateTime, nullable=True)
    status = Column(SQLEnum(TaskStatus), default=TaskStatus.PENDING)

    # Relationships
    session = relationship("Session", back_populates="conversations")
    messages = relationship("Message", back_populates="conversation", cascade="all, delete-orphan")

    __table_args__ = (
        Index("idx_conv_session", "session_id"),
        Index("idx_conv_agent", "agent_type"),
    )


class Message(Base):
    """Individual messages in a conversation."""
    __tablename__ = "messages"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    conversation_id = Column(UUID(as_uuid=True), ForeignKey("conversations.id"), nullable=False)
    role = Column(String(50), nullable=False)  # user, assistant, system, tool
    content = Column(Text, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    metadata = Column(JSONB, default=dict)  # tokens, model, etc.

    conversation = relationship("Conversation", back_populates="messages")

    __table_args__ = (
        Index("idx_message_conv", "conversation_id"),
        Index("idx_message_created", "created_at"),
    )


# ============== CODEX TASK MANAGEMENT ==============

class CodexTask(Base):
    """Async codex-cli execution tasks."""
    __tablename__ = "codex_tasks"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    session_id = Column(UUID(as_uuid=True), ForeignKey("sessions.id"), nullable=False)
    agent_type = Column(SQLEnum(AgentType), nullable=False)

    # Task details
    prompt = Column(Text, nullable=False)
    working_directory = Column(String(500), nullable=False)
    status = Column(SQLEnum(TaskStatus), default=TaskStatus.PENDING)

    # Process tracking
    process_id = Column(Integer, nullable=True)
    started_at = Column(DateTime, nullable=True)
    completed_at = Column(DateTime, nullable=True)

    # Results
    stdout = Column(Text)
    stderr = Column(Text)
    exit_code = Column(Integer, nullable=True)
    output_files = Column(ARRAY(String))  # Files created/modified

    # Callback tracking
    callback_sent = Column(Boolean, default=False)
    callback_acknowledged = Column(Boolean, default=False)

    session = relationship("Session", back_populates="codex_tasks")

    __table_args__ = (
        Index("idx_codex_session", "session_id"),
        Index("idx_codex_status", "status"),
        Index("idx_codex_agent", "agent_type"),
    )


# ============== DATABASE INITIALIZATION ==============

DATABASE_URL = "postgresql+asyncpg://codex_agent:your_secure_password@localhost:5432/codex_agents_db"

engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
    echo=False
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)


async def init_db():
    """Initialize database tables."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)


async def get_db():
    """Dependency for database sessions."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

**Checklist:**
- [ ] Create database.py with all models
- [ ] Run migrations to create tables
- [ ] Test database connectivity
- [ ] Create indexes for performance

---

## Phase 3: Codex Process Manager

### 3.1 Async Codex Executor

The core component that runs codex-cli asynchronously and notifies agents on completion.

**File: `backend/services/codex_manager.py`**

```python
import asyncio
import os
import signal
import uuid
from datetime import datetime
from typing import Optional, Callable, Dict, Any, AsyncGenerator
from dataclasses import dataclass
from enum import Enum
import logging

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update

from models.database import CodexTask, TaskStatus, AgentType

logger = logging.getLogger(__name__)


@dataclass
class CodexResult:
    """Result from a codex-cli execution."""
    task_id: str
    success: bool
    stdout: str
    stderr: str
    exit_code: int
    output_files: list[str]
    duration_seconds: float


class CodexProcessManager:
    """
    Manages async codex-cli processes.

    Key features:
    - Non-blocking execution
    - Real-time output streaming
    - Task completion callbacks to agents
    - Process lifecycle management
    - Per-project workspace isolation
    """

    def __init__(self, workspaces_root: str = "/workspaces"):
        self.workspaces_root = workspaces_root
        self.active_processes: Dict[str, asyncio.subprocess.Process] = {}
        self.output_buffers: Dict[str, list[str]] = {}
        self.completion_callbacks: Dict[str, Callable] = {}

    async def execute_codex(
        self,
        task_id: str,
        prompt: str,
        working_directory: str,
        db: AsyncSession,
        on_output: Optional[Callable[[str], None]] = None,
        on_complete: Optional[Callable[[CodexResult], None]] = None,
        timeout_seconds: int = 600
    ) -> None:
        """
        Execute codex-cli asynchronously.

        Args:
            task_id: Unique task identifier
            prompt: The prompt to send to codex
            working_directory: Project workspace path
            db: Database session for status updates
            on_output: Callback for streaming output lines
            on_complete: Callback when task finishes
            timeout_seconds: Maximum execution time
        """
        # Update task status to running
        await self._update_task_status(db, task_id, TaskStatus.RUNNING)

        # Prepare the codex command
        # Using --yes for non-interactive mode, --quiet for cleaner output
        cmd = [
            "codex",
            "--yes",  # Auto-approve file changes
            "--quiet",  # Reduce noise
            prompt
        ]

        self.output_buffers[task_id] = []
        start_time = datetime.utcnow()

        try:
            # Create subprocess
            process = await asyncio.create_subprocess_exec(
                *cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                cwd=working_directory,
                env={
                    **os.environ,
                    "CODEX_NONINTERACTIVE": "1",
                    "NO_COLOR": "1"  # Disable ANSI colors for cleaner logs
                }
            )

            self.active_processes[task_id] = process

            # Update task with process ID
            await db.execute(
                update(CodexTask)
                .where(CodexTask.id == task_id)
                .values(process_id=process.pid, started_at=datetime.utcnow())
            )
            await db.commit()

            # Stream output with timeout
            stdout_lines = []
            stderr_lines = []

            async def read_stream(stream, lines_list, is_stderr=False):
                while True:
                    line = await stream.readline()
                    if not line:
                        break
                    decoded = line.decode('utf-8', errors='replace').rstrip()
                    lines_list.append(decoded)
                    self.output_buffers[task_id].append(
                        f"{'[stderr] ' if is_stderr else ''}{decoded}"
                    )
                    if on_output:
                        await on_output(decoded)

            try:
                await asyncio.wait_for(
                    asyncio.gather(
                        read_stream(process.stdout, stdout_lines),
                        read_stream(process.stderr, stderr_lines, True)
                    ),
                    timeout=timeout_seconds
                )
                await process.wait()

            except asyncio.TimeoutError:
                process.kill()
                await process.wait()
                stderr_lines.append(f"Process timed out after {timeout_seconds} seconds")

            # Calculate duration
            duration = (datetime.utcnow() - start_time).total_seconds()

            # Detect output files (files modified in workspace)
            output_files = await self._detect_modified_files(working_directory, start_time)

            # Create result
            result = CodexResult(
                task_id=task_id,
                success=process.returncode == 0,
                stdout="\n".join(stdout_lines),
                stderr="\n".join(stderr_lines),
                exit_code=process.returncode or -1,
                output_files=output_files,
                duration_seconds=duration
            )

            # Update database
            await db.execute(
                update(CodexTask)
                .where(CodexTask.id == task_id)
                .values(
                    status=TaskStatus.COMPLETED if result.success else TaskStatus.FAILED,
                    stdout=result.stdout,
                    stderr=result.stderr,
                    exit_code=result.exit_code,
                    output_files=result.output_files,
                    completed_at=datetime.utcnow()
                )
            )
            await db.commit()

            # Trigger completion callback
            if on_complete:
                await on_complete(result)

        except Exception as e:
            logger.exception(f"Codex execution failed for task {task_id}")
            await self._update_task_status(db, task_id, TaskStatus.FAILED)

            if on_complete:
                await on_complete(CodexResult(
                    task_id=task_id,
                    success=False,
                    stdout="",
                    stderr=str(e),
                    exit_code=-1,
                    output_files=[],
                    duration_seconds=0
                ))
        finally:
            # Cleanup
            self.active_processes.pop(task_id, None)
            self.output_buffers.pop(task_id, None)

    async def stream_output(self, task_id: str) -> AsyncGenerator[str, None]:
        """Stream output lines for a running task."""
        last_index = 0
        while task_id in self.active_processes or task_id in self.output_buffers:
            buffer = self.output_buffers.get(task_id, [])
            while last_index < len(buffer):
                yield buffer[last_index]
                last_index += 1
            await asyncio.sleep(0.1)

    async def cancel_task(self, task_id: str, db: AsyncSession) -> bool:
        """Cancel a running codex task."""
        process = self.active_processes.get(task_id)
        if process:
            try:
                process.send_signal(signal.SIGTERM)
                await asyncio.sleep(2)
                if process.returncode is None:
                    process.kill()
                await self._update_task_status(db, task_id, TaskStatus.CANCELLED)
                return True
            except Exception as e:
                logger.error(f"Failed to cancel task {task_id}: {e}")
        return False

    async def get_task_output(self, task_id: str) -> list[str]:
        """Get buffered output for a task."""
        return self.output_buffers.get(task_id, [])

    async def _update_task_status(
        self,
        db: AsyncSession,
        task_id: str,
        status: TaskStatus
    ) -> None:
        """Update task status in database."""
        await db.execute(
            update(CodexTask)
            .where(CodexTask.id == task_id)
            .values(status=status)
        )
        await db.commit()

    async def _detect_modified_files(
        self,
        directory: str,
        since: datetime
    ) -> list[str]:
        """Detect files modified since a given time."""
        modified = []
        since_ts = since.timestamp()

        for root, dirs, files in os.walk(directory):
            # Skip hidden directories and common non-source dirs
            dirs[:] = [d for d in dirs if not d.startswith('.') and d not in ['node_modules', '__pycache__', 'venv']]

            for file in files:
                filepath = os.path.join(root, file)
                try:
                    if os.path.getmtime(filepath) > since_ts:
                        # Return relative path
                        modified.append(os.path.relpath(filepath, directory))
                except OSError:
                    continue

        return modified


# Global instance
codex_manager = CodexProcessManager()
```

**Checklist:**
- [ ] Implement CodexProcessManager class
- [ ] Add streaming output support
- [ ] Implement task cancellation
- [ ] Add file modification detection
- [ ] Test with real codex-cli commands

---

## Phase 4: Agent Definitions

### 4.1 Agent Base Architecture

Using the OpenAI Agents SDK patterns for agent definitions.

**File: `backend/agents/definitions.py`**

```python
from typing import List, Optional, Callable, Any
from dataclasses import dataclass
from enum import Enum

from agents import Agent, handoff, function_tool, RunContextWrapper
from agents.tool import FunctionTool


class AgentRole(str, Enum):
    PLANNER = "planner"
    BUILDER = "builder"
    QA_QC = "qa_qc"
    PROD_READY = "prod_ready"


@dataclass
class AgentConfig:
    """Configuration for each agent."""
    role: AgentRole
    name: str
    description: str
    instructions: str
    handoff_description: str
    codex_prompt_template: str


# ============== AGENT CONFIGURATIONS ==============

AGENT_CONFIGS = {
    AgentRole.PLANNER: AgentConfig(
        role=AgentRole.PLANNER,
        name="Planner Agent",
        description="Expert business analyst that creates detailed requirements",
        instructions="""You are an expert business analyst and software architect.

Your role is to analyze user requests and create comprehensive requirements documents.

WORKFLOW:
1. Analyze the user's request thoroughly
2. Ask clarifying questions if needed
3. Break down the request into clear, actionable requirements
4. Create a detailed AGENTS.md specification

OUTPUT FORMAT - You must create an AGENTS.md file with:
- Project Overview
- Functional Requirements (numbered list)
- Technical Requirements
- File Structure (what files to create/modify)
- Acceptance Criteria
- Dependencies needed

RULES:
- Be specific and unambiguous
- Include edge cases
- Define clear success criteria
- Specify technology choices when relevant

When ready to execute, use the run_codex tool with a prompt that instructs codex to:
1. Analyze the user's request
2. Generate a complete AGENTS.md file
3. Save it to the project workspace""",
        handoff_description="Transfer to Planner for requirements analysis and AGENTS.md creation",
        codex_prompt_template="""Analyze the following user request and create a detailed AGENTS.md requirements document.

USER REQUEST:
{user_request}

PROJECT CONTEXT:
{project_context}

Create AGENTS.md with:
1. Project Overview - What we're building
2. Functional Requirements - Numbered list of features
3. Technical Requirements - Stack, dependencies, architecture
4. File Structure - Files to create/modify
5. Acceptance Criteria - How to verify completion
6. Implementation Steps - Ordered steps for the builder

Save the file as AGENTS.md in the current directory."""
    ),

    AgentRole.BUILDER: AgentConfig(
        role=AgentRole.BUILDER,
        name="Builder Agent",
        description="Expert developer that implements requirements step by step",
        instructions="""You are an expert software developer.

Your role is to implement the requirements defined in AGENTS.md step by step.

WORKFLOW:
1. Read and understand the AGENTS.md requirements
2. Plan the implementation order
3. Execute codex commands for each implementation step
4. Verify each step before moving to the next

RULES:
- Follow AGENTS.md exactly
- Implement one feature at a time
- Write clean, production-ready code
- Include proper error handling
- Add necessary comments
- Create tests where specified

When executing codex, be specific:
- Reference exact file paths
- Include full context for each task
- Specify expected behavior clearly

After completing implementation, summarize what was built and any deviations from the plan.""",
        handoff_description="Transfer to Builder for step-by-step implementation",
        codex_prompt_template="""Implement the following from AGENTS.md:

CURRENT STEP:
{current_step}

REQUIREMENTS FROM AGENTS.md:
{requirements}

EXISTING CODE CONTEXT:
{code_context}

Instructions:
1. Implement this specific step
2. Follow the coding standards in AGENTS.md
3. Ensure code is production-ready
4. Add appropriate error handling
5. Include inline comments for complex logic

Do not implement anything beyond the current step."""
    ),

    AgentRole.QA_QC: AgentConfig(
        role=AgentRole.QA_QC,
        name="QA/QC Agent",
        description="Quality assurance expert that tests and debugs the build",
        instructions="""You are an expert QA engineer and debugger.

Your role is to verify the implementation meets AGENTS.md requirements and fix issues.

WORKFLOW:
1. Review AGENTS.md requirements
2. Examine the implemented code
3. Run tests and identify issues
4. Debug and fix problems via codex
5. Verify fixes

TESTING APPROACH:
- Check all acceptance criteria
- Test edge cases
- Verify error handling
- Check for security issues
- Validate performance concerns

DEBUGGING PROCESS:
- Identify root cause
- Create minimal fix
- Verify fix doesn't break other functionality
- Document the issue and resolution

When using codex for debugging:
- Provide full error context
- Include relevant code snippets
- Specify expected vs actual behavior""",
        handoff_description="Transfer to QA/QC for testing, debugging, and quality verification",
        codex_prompt_template="""Debug and fix the following issue:

ISSUE:
{issue_description}

ERROR OUTPUT:
{error_output}

RELEVANT CODE:
{relevant_code}

AGENTS.MD REQUIREMENTS:
{requirements}

Instructions:
1. Analyze the root cause
2. Implement the minimal fix
3. Ensure the fix doesn't break other functionality
4. Add tests to prevent regression if applicable"""
    ),

    AgentRole.PROD_READY: AgentConfig(
        role=AgentRole.PROD_READY,
        name="Production Readiness Agent",
        description="DevOps expert that validates production readiness",
        instructions="""You are an expert DevOps engineer and production readiness reviewer.

Your role is to ensure the build is ready for production deployment.

CHECKLIST:
1. All AGENTS.md requirements implemented
2. All tests passing
3. No security vulnerabilities
4. Proper error handling
5. Logging and monitoring ready
6. Documentation complete
7. Dependencies properly specified
8. Environment configuration correct

VERIFICATION:
- Run full test suite
- Check for hardcoded secrets
- Verify environment variables
- Validate build process
- Check deployment configuration

OUTPUT:
- Production readiness report
- List of any blocking issues
- Recommendations for improvement
- Final approval or rejection with reasons

When using codex:
- Focus on verification commands
- Run security scans
- Validate configurations
- Generate deployment checklists""",
        handoff_description="Transfer to Production Readiness for final validation",
        codex_prompt_template="""Perform production readiness check:

AGENTS.MD REQUIREMENTS:
{requirements}

IMPLEMENTATION SUMMARY:
{implementation_summary}

QA REPORT:
{qa_report}

Verify:
1. All requirements from AGENTS.md are implemented
2. Run test suite and report results
3. Check for security issues (hardcoded secrets, SQL injection, XSS)
4. Verify error handling is comprehensive
5. Check logging is in place
6. Validate environment configuration
7. Generate production readiness report

Output a structured report with:
- Requirements checklist (pass/fail for each)
- Test results summary
- Security scan results
- Blocking issues (if any)
- Final verdict: APPROVED or NEEDS_WORK with specific items"""
    )
}


def get_agent_config(role: AgentRole) -> AgentConfig:
    """Get configuration for an agent role."""
    return AGENT_CONFIGS[role]


def get_all_agent_configs() -> List[AgentConfig]:
    """Get all agent configurations."""
    return list(AGENT_CONFIGS.values())
```

### 4.2 Agent Implementation with Codex Tools

**File: `backend/agents/agent_factory.py`**

```python
import asyncio
from typing import Optional, Dict, Any, List
from datetime import datetime
import json

from agents import Agent, Runner, function_tool, handoff, RunContextWrapper
from agents.run import RunConfig
from sqlalchemy.ext.asyncio import AsyncSession

from .definitions import AgentRole, AgentConfig, get_agent_config, AGENT_CONFIGS
from services.codex_manager import codex_manager, CodexResult
from models.database import (
    CodexTask, Session, Conversation, Message,
    TaskStatus, AgentType, AsyncSessionLocal
)


class AgentContext:
    """Context passed to agents during execution."""

    def __init__(
        self,
        project_id: str,
        session_id: str,
        workspace_path: str,
        db: AsyncSession,
        agents_md_content: Optional[str] = None,
        previous_outputs: Optional[Dict[str, str]] = None
    ):
        self.project_id = project_id
        self.session_id = session_id
        self.workspace_path = workspace_path
        self.db = db
        self.agents_md_content = agents_md_content
        self.previous_outputs = previous_outputs or {}
        self.codex_results: List[CodexResult] = []


class CodexAgentFactory:
    """Factory for creating agents that orchestrate codex-cli."""

    def __init__(self):
        self._agents: Dict[AgentRole, Agent] = {}
        self._initialize_agents()

    def _initialize_agents(self):
        """Initialize all agents with their tools."""
        for role, config in AGENT_CONFIGS.items():
            self._agents[role] = self._create_agent(config)

    def _create_agent(self, config: AgentConfig) -> Agent:
        """Create an agent with codex execution tools."""

        @function_tool
        async def run_codex(
            ctx: RunContextWrapper[AgentContext],
            prompt: str,
            wait_for_completion: bool = True
        ) -> str:
            """
            Execute a codex-cli command in the project workspace.

            Args:
                prompt: The prompt to send to codex-cli
                wait_for_completion: Whether to wait for codex to finish

            Returns:
                Result of the codex execution
            """
            context: AgentContext = ctx.context

            # Create task record
            task = CodexTask(
                session_id=context.session_id,
                agent_type=AgentType(config.role.value),
                prompt=prompt,
                working_directory=context.workspace_path,
                status=TaskStatus.PENDING
            )
            context.db.add(task)
            await context.db.commit()
            await context.db.refresh(task)

            result_holder = {"result": None}

            async def on_complete(result: CodexResult):
                result_holder["result"] = result
                context.codex_results.append(result)

            if wait_for_completion:
                # Execute and wait
                await codex_manager.execute_codex(
                    task_id=str(task.id),
                    prompt=prompt,
                    working_directory=context.workspace_path,
                    db=context.db,
                    on_complete=on_complete
                )

                result = result_holder["result"]
                if result:
                    return f"""Codex execution completed.
Exit code: {result.exit_code}
Success: {result.success}

Output:
{result.stdout}

{f'Errors: {result.stderr}' if result.stderr else ''}

Files modified: {', '.join(result.output_files) if result.output_files else 'None detected'}"""
                else:
                    return "Codex execution completed but no result captured."
            else:
                # Start async execution
                asyncio.create_task(
                    codex_manager.execute_codex(
                        task_id=str(task.id),
                        prompt=prompt,
                        working_directory=context.workspace_path,
                        db=context.db,
                        on_complete=on_complete
                    )
                )
                return f"Codex task started with ID: {task.id}. Use check_codex_status to monitor."

        @function_tool
        async def check_codex_status(
            ctx: RunContextWrapper[AgentContext],
            task_id: str
        ) -> str:
            """
            Check the status of a running codex task.

            Args:
                task_id: The task ID to check

            Returns:
                Current status and output
            """
            context: AgentContext = ctx.context

            # Get task from database
            from sqlalchemy import select
            result = await context.db.execute(
                select(CodexTask).where(CodexTask.id == task_id)
            )
            task = result.scalar_one_or_none()

            if not task:
                return f"Task {task_id} not found."

            output = await codex_manager.get_task_output(task_id)

            return f"""Task Status: {task.status.value}
Started: {task.started_at}
Completed: {task.completed_at}

Recent output:
{chr(10).join(output[-50:]) if output else 'No output yet'}"""

        @function_tool
        async def read_file(
            ctx: RunContextWrapper[AgentContext],
            file_path: str
        ) -> str:
            """
            Read a file from the project workspace.

            Args:
                file_path: Relative path to the file

            Returns:
                File contents
            """
            import os
            context: AgentContext = ctx.context
            full_path = os.path.join(context.workspace_path, file_path)

            try:
                with open(full_path, 'r') as f:
                    return f.read()
            except FileNotFoundError:
                return f"File not found: {file_path}"
            except Exception as e:
                return f"Error reading file: {str(e)}"

        @function_tool
        async def list_files(
            ctx: RunContextWrapper[AgentContext],
            directory: str = "."
        ) -> str:
            """
            List files in a directory within the project workspace.

            Args:
                directory: Relative path to directory (default: root)

            Returns:
                List of files and directories
            """
            import os
            context: AgentContext = ctx.context
            full_path = os.path.join(context.workspace_path, directory)

            try:
                entries = []
                for entry in os.listdir(full_path):
                    entry_path = os.path.join(full_path, entry)
                    if os.path.isdir(entry_path):
                        entries.append(f"[DIR]  {entry}/")
                    else:
                        size = os.path.getsize(entry_path)
                        entries.append(f"[FILE] {entry} ({size} bytes)")
                return "\n".join(entries) if entries else "Directory is empty"
            except Exception as e:
                return f"Error listing directory: {str(e)}"

        @function_tool
        async def get_agents_md(
            ctx: RunContextWrapper[AgentContext]
        ) -> str:
            """
            Get the current AGENTS.md requirements document.

            Returns:
                Contents of AGENTS.md or message if not found
            """
            context: AgentContext = ctx.context
            if context.agents_md_content:
                return context.agents_md_content

            # Try to read from file
            import os
            agents_md_path = os.path.join(context.workspace_path, "AGENTS.md")
            try:
                with open(agents_md_path, 'r') as f:
                    content = f.read()
                    context.agents_md_content = content
                    return content
            except FileNotFoundError:
                return "AGENTS.md not found. It will be created by the Planner agent."

        @function_tool
        async def save_agents_md(
            ctx: RunContextWrapper[AgentContext],
            content: str
        ) -> str:
            """
            Save the AGENTS.md requirements document.

            Args:
                content: The AGENTS.md content to save

            Returns:
                Confirmation message
            """
            import os
            context: AgentContext = ctx.context
            agents_md_path = os.path.join(context.workspace_path, "AGENTS.md")

            try:
                with open(agents_md_path, 'w') as f:
                    f.write(content)
                context.agents_md_content = content

                # Also update the session
                from sqlalchemy import update
                from models.database import Session as SessionModel
                await context.db.execute(
                    update(SessionModel)
                    .where(SessionModel.id == context.session_id)
                    .values(agents_md_content=content)
                )
                await context.db.commit()

                return f"AGENTS.md saved successfully ({len(content)} characters)"
            except Exception as e:
                return f"Error saving AGENTS.md: {str(e)}"

        @function_tool
        async def handoff_to_next_agent(
            ctx: RunContextWrapper[AgentContext],
            summary: str,
            next_agent: str
        ) -> str:
            """
            Signal completion and provide summary for the next agent.

            Args:
                summary: Summary of work completed
                next_agent: Name of the next agent (builder, qa_qc, prod_ready)

            Returns:
                Confirmation of handoff
            """
            context: AgentContext = ctx.context
            context.previous_outputs[config.role.value] = summary
            return f"Handoff prepared for {next_agent}. Summary recorded."

        # Create the agent
        tools = [
            run_codex,
            check_codex_status,
            read_file,
            list_files,
            get_agents_md,
        ]

        # Planner can save AGENTS.md
        if config.role == AgentRole.PLANNER:
            tools.append(save_agents_md)

        # All agents can handoff
        tools.append(handoff_to_next_agent)

        return Agent(
            name=config.name,
            instructions=config.instructions,
            tools=tools,
            model="gpt-4o",
            handoff_description=config.handoff_description
        )

    def get_agent(self, role: AgentRole) -> Agent:
        """Get an agent by role."""
        return self._agents[role]

    async def run_agent(
        self,
        role: AgentRole,
        message: str,
        context: AgentContext,
        chat_history: Optional[List[Dict[str, str]]] = None
    ) -> str:
        """Run an agent with the given context."""
        agent = self.get_agent(role)

        # Build input with context
        input_parts = [message]

        if context.agents_md_content and role != AgentRole.PLANNER:
            input_parts.append(f"\n\nCurrent AGENTS.md:\n{context.agents_md_content}")

        if context.previous_outputs:
            input_parts.append("\n\nPrevious agent outputs:")
            for agent_name, output in context.previous_outputs.items():
                input_parts.append(f"\n{agent_name}: {output}")

        full_input = "\n".join(input_parts)

        # Run the agent
        result = await Runner.run(
            agent,
            input=full_input,
            context=context
        )

        return result.final_output


# Global factory instance
agent_factory = CodexAgentFactory()
```

**Checklist:**
- [ ] Implement AgentContext class
- [ ] Implement CodexAgentFactory
- [ ] Create all 4 agent configurations
- [ ] Implement codex execution tools
- [ ] Implement file operation tools
- [ ] Test agent creation and tool execution

---

## Phase 5: Orchestration Engine

### 5.1 Workflow Orchestrator

**File: `backend/services/orchestrator.py`**

```python
import asyncio
from typing import Optional, Dict, Any, AsyncGenerator, Callable
from datetime import datetime
from enum import Enum
import logging

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update

from models.database import (
    Session, Conversation, Message, Project,
    TaskStatus, AgentType, AsyncSessionLocal
)
from agents.definitions import AgentRole, get_agent_config
from agents.agent_factory import agent_factory, AgentContext

logger = logging.getLogger(__name__)


class WorkflowStage(str, Enum):
    PLANNING = "planning"
    BUILDING = "building"
    QA_QC = "qa_qc"
    PROD_READY = "prod_ready"
    COMPLETED = "completed"
    FAILED = "failed"


class WorkflowOrchestrator:
    """
    Orchestrates the 4-agent workflow:
    Planner -> Builder -> QA/QC -> Production Ready

    Features:
    - Automatic handoffs between agents
    - Real-time progress streaming
    - Error recovery and retry logic
    - Manual intervention points
    """

    WORKFLOW_ORDER = [
        AgentRole.PLANNER,
        AgentRole.BUILDER,
        AgentRole.QA_QC,
        AgentRole.PROD_READY
    ]

    def __init__(self):
        self.active_workflows: Dict[str, WorkflowStage] = {}

    async def start_workflow(
        self,
        session_id: str,
        project_id: str,
        user_request: str,
        db: AsyncSession,
        on_progress: Optional[Callable[[str, str], None]] = None
    ) -> Dict[str, Any]:
        """
        Start a complete workflow from user request to production ready.

        Args:
            session_id: Session identifier
            project_id: Project identifier
            user_request: Original user request
            db: Database session
            on_progress: Callback for progress updates (stage, message)

        Returns:
            Workflow result with all agent outputs
        """
        # Get project workspace
        project = await db.get(Project, project_id)
        if not project:
            raise ValueError(f"Project {project_id} not found")

        workspace_path = project.workspace_path

        # Initialize workflow state
        self.active_workflows[session_id] = WorkflowStage.PLANNING
        results = {
            "session_id": session_id,
            "project_id": project_id,
            "stages": {},
            "final_status": None,
            "agents_md": None
        }

        # Create agent context
        context = AgentContext(
            project_id=project_id,
            session_id=session_id,
            workspace_path=workspace_path,
            db=db
        )

        try:
            for agent_role in self.WORKFLOW_ORDER:
                stage_name = agent_role.value
                self.active_workflows[session_id] = WorkflowStage(stage_name)

                if on_progress:
                    await on_progress(stage_name, f"Starting {agent_role.name}")

                # Update session with current agent
                await db.execute(
                    update(Session)
                    .where(Session.id == session_id)
                    .values(current_agent=AgentType(agent_role.value))
                )
                await db.commit()

                # Create conversation record
                conversation = Conversation(
                    session_id=session_id,
                    agent_type=AgentType(agent_role.value),
                    status=TaskStatus.RUNNING
                )
                db.add(conversation)
                await db.commit()
                await db.refresh(conversation)

                # Build agent input based on stage
                agent_input = self._build_agent_input(
                    agent_role, user_request, context, results
                )

                # Record user message
                user_msg = Message(
                    conversation_id=conversation.id,
                    role="user",
                    content=agent_input
                )
                db.add(user_msg)
                await db.commit()

                # Run agent
                try:
                    output = await agent_factory.run_agent(
                        role=agent_role,
                        message=agent_input,
                        context=context
                    )

                    # Record agent response
                    agent_msg = Message(
                        conversation_id=conversation.id,
                        role="assistant",
                        content=output
                    )
                    db.add(agent_msg)

                    # Update conversation status
                    conversation.status = TaskStatus.COMPLETED
                    conversation.ended_at = datetime.utcnow()
                    await db.commit()

                    # Store result
                    results["stages"][stage_name] = {
                        "status": "completed",
                        "output": output,
                        "codex_tasks": len(context.codex_results)
                    }

                    # Update context for next agent
                    context.previous_outputs[stage_name] = output

                    # Capture AGENTS.md if created
                    if context.agents_md_content:
                        results["agents_md"] = context.agents_md_content

                    if on_progress:
                        await on_progress(stage_name, f"Completed {agent_role.name}")

                except Exception as e:
                    logger.exception(f"Agent {agent_role.value} failed")
                    conversation.status = TaskStatus.FAILED
                    await db.commit()

                    results["stages"][stage_name] = {
                        "status": "failed",
                        "error": str(e)
                    }

                    # Don't continue workflow on failure
                    results["final_status"] = "failed"
                    self.active_workflows[session_id] = WorkflowStage.FAILED
                    return results

            # All stages completed
            results["final_status"] = "completed"
            self.active_workflows[session_id] = WorkflowStage.COMPLETED

            # Update session
            await db.execute(
                update(Session)
                .where(Session.id == session_id)
                .values(
                    status=TaskStatus.COMPLETED,
                    completed_at=datetime.utcnow(),
                    final_summary=results["stages"].get("prod_ready", {}).get("output")
                )
            )
            await db.commit()

            return results

        except Exception as e:
            logger.exception("Workflow failed")
            self.active_workflows[session_id] = WorkflowStage.FAILED
            results["final_status"] = "failed"
            results["error"] = str(e)
            return results
        finally:
            self.active_workflows.pop(session_id, None)

    def _build_agent_input(
        self,
        role: AgentRole,
        user_request: str,
        context: AgentContext,
        results: Dict[str, Any]
    ) -> str:
        """Build input message for each agent based on workflow stage."""

        config = get_agent_config(role)

        if role == AgentRole.PLANNER:
            return f"""User Request: {user_request}

Your task is to:
1. Analyze this request thoroughly
2. Use the run_codex tool to have codex create a comprehensive AGENTS.md file
3. The AGENTS.md should contain all requirements for the other agents

Project workspace is ready at: {context.workspace_path}

Begin by understanding the request, then execute codex to generate AGENTS.md."""

        elif role == AgentRole.BUILDER:
            planner_output = results["stages"].get("planner", {}).get("output", "")
            return f"""The Planner has created AGENTS.md with requirements.

Planner's summary: {planner_output[:1000]}

Your task is to:
1. Read AGENTS.md using get_agents_md tool
2. Implement each requirement step by step using run_codex
3. Execute codex for each feature/component separately
4. Verify each implementation before moving to the next

Begin by reading AGENTS.md, then implement the requirements."""

        elif role == AgentRole.QA_QC:
            builder_output = results["stages"].get("builder", {}).get("output", "")
            return f"""The Builder has implemented the requirements.

Builder's summary: {builder_output[:1000]}

Your task is to:
1. Read AGENTS.md for acceptance criteria
2. Test each requirement using run_codex (run tests, check functionality)
3. Debug and fix any issues found
4. Document test results

Begin by reviewing what was built, then run comprehensive tests."""

        elif role == AgentRole.PROD_READY:
            qa_output = results["stages"].get("qa_qc", {}).get("output", "")
            return f"""QA/QC testing is complete.

QA summary: {qa_output[:1000]}

Your task is to:
1. Verify all AGENTS.md requirements are met
2. Run final production checks via codex
3. Check for security issues, proper error handling
4. Generate a final production readiness report

Produce a final verdict: APPROVED or NEEDS_WORK with specific items."""

    async def run_single_agent(
        self,
        agent_role: AgentRole,
        session_id: str,
        project_id: str,
        message: str,
        db: AsyncSession
    ) -> str:
        """Run a single agent (for direct chat mode)."""
        project = await db.get(Project, project_id)
        if not project:
            raise ValueError(f"Project {project_id} not found")

        # Get or create session
        session = await db.get(Session, session_id)

        # Create context
        context = AgentContext(
            project_id=project_id,
            session_id=session_id,
            workspace_path=project.workspace_path,
            db=db,
            agents_md_content=session.agents_md_content if session else None
        )

        # Run agent
        output = await agent_factory.run_agent(
            role=agent_role,
            message=message,
            context=context
        )

        return output

    def get_workflow_status(self, session_id: str) -> Optional[WorkflowStage]:
        """Get current workflow stage."""
        return self.active_workflows.get(session_id)


# Global orchestrator instance
orchestrator = WorkflowOrchestrator()
```

**Checklist:**
- [ ] Implement WorkflowOrchestrator class
- [ ] Implement automatic stage transitions
- [ ] Add progress callbacks for real-time updates
- [ ] Implement single agent mode
- [ ] Add error handling and recovery
- [ ] Test complete workflow execution

---

## Phase 6: API Layer

### 6.1 FastAPI Application

**File: `backend/main.py`**

```python
import os
import sys
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

from config import settings
from models.database import init_db
from routes import (
    projects_router,
    sessions_router,
    agents_router,
    files_router,
    websocket_router
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan events."""
    print("Starting Codex Multi-Agent System...")
    await init_db()
    print("Database initialized")

    # Ensure workspaces directory exists
    os.makedirs(settings.WORKSPACES_ROOT, exist_ok=True)
    print(f"Workspaces root: {settings.WORKSPACES_ROOT}")

    yield

    print("Shutting down...")


app = FastAPI(
    title="Codex Multi-Agent System",
    description="Multi-agent system orchestrating codex-cli for software development",
    version="1.0.0",
    lifespan=lifespan
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routes
app.include_router(projects_router, prefix="/api/projects", tags=["Projects"])
app.include_router(sessions_router, prefix="/api/sessions", tags=["Sessions"])
app.include_router(agents_router, prefix="/api/agents", tags=["Agents"])
app.include_router(files_router, prefix="/api/files", tags=["Files"])
app.include_router(websocket_router, prefix="/ws", tags=["WebSocket"])


@app.get("/api/health")
async def health_check():
    return {"status": "healthy", "version": "1.0.0"}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=settings.DEBUG)
```

### 6.2 API Routes

**File: `backend/routes/projects.py`**

```python
import os
import uuid
from typing import List, Optional
from datetime import datetime

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from pydantic import BaseModel

from models.database import Project, get_db
from config import settings

router = APIRouter()


class ProjectCreate(BaseModel):
    name: str
    description: Optional[str] = None
    settings: Optional[dict] = None


class ProjectResponse(BaseModel):
    id: str
    name: str
    description: Optional[str]
    workspace_path: str
    created_at: datetime
    updated_at: datetime
    is_active: bool
    settings: dict

    class Config:
        from_attributes = True


class ProjectList(BaseModel):
    projects: List[ProjectResponse]
    total: int


@router.post("/", response_model=ProjectResponse)
async def create_project(
    project: ProjectCreate,
    db: AsyncSession = Depends(get_db)
):
    """Create a new project with isolated workspace."""
    project_id = str(uuid.uuid4())
    workspace_path = os.path.join(settings.WORKSPACES_ROOT, project_id)

    # Create workspace directory
    os.makedirs(workspace_path, exist_ok=True)

    # Create project record
    db_project = Project(
        id=project_id,
        name=project.name,
        description=project.description,
        workspace_path=workspace_path,
        settings=project.settings or {}
    )

    db.add(db_project)
    await db.commit()
    await db.refresh(db_project)

    return ProjectResponse(
        id=str(db_project.id),
        name=db_project.name,
        description=db_project.description,
        workspace_path=db_project.workspace_path,
        created_at=db_project.created_at,
        updated_at=db_project.updated_at,
        is_active=db_project.is_active,
        settings=db_project.settings
    )


@router.get("/", response_model=ProjectList)
async def list_projects(
    skip: int = 0,
    limit: int = 50,
    active_only: bool = True,
    db: AsyncSession = Depends(get_db)
):
    """List all projects."""
    query = select(Project)
    if active_only:
        query = query.where(Project.is_active == True)
    query = query.order_by(Project.updated_at.desc()).offset(skip).limit(limit)

    result = await db.execute(query)
    projects = result.scalars().all()

    return ProjectList(
        projects=[
            ProjectResponse(
                id=str(p.id),
                name=p.name,
                description=p.description,
                workspace_path=p.workspace_path,
                created_at=p.created_at,
                updated_at=p.updated_at,
                is_active=p.is_active,
                settings=p.settings
            )
            for p in projects
        ],
        total=len(projects)
    )


@router.get("/{project_id}", response_model=ProjectResponse)
async def get_project(
    project_id: str,
    db: AsyncSession = Depends(get_db)
):
    """Get a specific project."""
    project = await db.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")

    return ProjectResponse(
        id=str(project.id),
        name=project.name,
        description=project.description,
        workspace_path=project.workspace_path,
        created_at=project.created_at,
        updated_at=project.updated_at,
        is_active=project.is_active,
        settings=project.settings
    )


@router.delete("/{project_id}")
async def delete_project(
    project_id: str,
    db: AsyncSession = Depends(get_db)
):
    """Soft delete a project."""
    project = await db.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")

    project.is_active = False
    await db.commit()

    return {"status": "deleted", "project_id": project_id}
```

**File: `backend/routes/sessions.py`**

```python
import uuid
from typing import List, Optional
from datetime import datetime

from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from pydantic import BaseModel

from models.database import Session, Conversation, Message, TaskStatus, get_db
from services.orchestrator import orchestrator
from agents.definitions import AgentRole

router = APIRouter()


class SessionCreate(BaseModel):
    project_id: str
    user_request: str
    title: Optional[str] = None


class SessionResponse(BaseModel):
    id: str
    project_id: str
    title: Optional[str]
    user_request: str
    status: str
    current_agent: Optional[str]
    created_at: datetime
    completed_at: Optional[datetime]
    agents_md_content: Optional[str]

    class Config:
        from_attributes = True


class MessageResponse(BaseModel):
    id: str
    role: str
    content: str
    created_at: datetime

    class Config:
        from_attributes = True


class ConversationResponse(BaseModel):
    id: str
    agent_type: str
    status: str
    started_at: datetime
    ended_at: Optional[datetime]
    messages: List[MessageResponse]

    class Config:
        from_attributes = True


@router.post("/", response_model=SessionResponse)
async def create_session(
    session: SessionCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db)
):
    """Create a new session and optionally start the workflow."""
    session_id = str(uuid.uuid4())

    db_session = Session(
        id=session_id,
        project_id=session.project_id,
        title=session.title or session.user_request[:100],
        user_request=session.user_request,
        status=TaskStatus.PENDING
    )

    db.add(db_session)
    await db.commit()
    await db.refresh(db_session)

    return SessionResponse(
        id=str(db_session.id),
        project_id=str(db_session.project_id),
        title=db_session.title,
        user_request=db_session.user_request,
        status=db_session.status.value,
        current_agent=db_session.current_agent.value if db_session.current_agent else None,
        created_at=db_session.created_at,
        completed_at=db_session.completed_at,
        agents_md_content=db_session.agents_md_content
    )


@router.post("/{session_id}/start")
async def start_workflow(
    session_id: str,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db)
):
    """Start the 4-agent workflow for a session."""
    session = await db.get(Session, session_id)
    if not session:
        raise HTTPException(status_code=404, detail="Session not found")

    if session.status != TaskStatus.PENDING:
        raise HTTPException(status_code=400, detail="Session already started")

    # Start workflow in background
    async def run_workflow():
        async with AsyncSessionLocal() as workflow_db:
            await orchestrator.start_workflow(
                session_id=session_id,
                project_id=str(session.project_id),
                user_request=session.user_request,
                db=workflow_db
            )

    background_tasks.add_task(run_workflow)

    return {"status": "started", "session_id": session_id}


@router.get("/{session_id}", response_model=SessionResponse)
async def get_session(
    session_id: str,
    db: AsyncSession = Depends(get_db)
):
    """Get session details."""
    session = await db.get(Session, session_id)
    if not session:
        raise HTTPException(status_code=404, detail="Session not found")

    return SessionResponse(
        id=str(session.id),
        project_id=str(session.project_id),
        title=session.title,
        user_request=session.user_request,
        status=session.status.value,
        current_agent=session.current_agent.value if session.current_agent else None,
        created_at=session.created_at,
        completed_at=session.completed_at,
        agents_md_content=session.agents_md_content
    )


@router.get("/{session_id}/conversations", response_model=List[ConversationResponse])
async def get_session_conversations(
    session_id: str,
    db: AsyncSession = Depends(get_db)
):
    """Get all conversations in a session."""
    result = await db.execute(
        select(Conversation)
        .where(Conversation.session_id == session_id)
        .order_by(Conversation.started_at)
    )
    conversations = result.scalars().all()

    response = []
    for conv in conversations:
        # Get messages for this conversation
        msg_result = await db.execute(
            select(Message)
            .where(Message.conversation_id == conv.id)
            .order_by(Message.created_at)
        )
        messages = msg_result.scalars().all()

        response.append(ConversationResponse(
            id=str(conv.id),
            agent_type=conv.agent_type.value,
            status=conv.status.value,
            started_at=conv.started_at,
            ended_at=conv.ended_at,
            messages=[
                MessageResponse(
                    id=str(m.id),
                    role=m.role,
                    content=m.content,
                    created_at=m.created_at
                )
                for m in messages
            ]
        ))

    return response


@router.get("/project/{project_id}", response_model=List[SessionResponse])
async def get_project_sessions(
    project_id: str,
    skip: int = 0,
    limit: int = 50,
    db: AsyncSession = Depends(get_db)
):
    """Get all sessions for a project."""
    result = await db.execute(
        select(Session)
        .where(Session.project_id == project_id)
        .order_by(Session.created_at.desc())
        .offset(skip)
        .limit(limit)
    )
    sessions = result.scalars().all()

    return [
        SessionResponse(
            id=str(s.id),
            project_id=str(s.project_id),
            title=s.title,
            user_request=s.user_request,
            status=s.status.value,
            current_agent=s.current_agent.value if s.current_agent else None,
            created_at=s.created_at,
            completed_at=s.completed_at,
            agents_md_content=s.agents_md_content
        )
        for s in sessions
    ]
```

**File: `backend/routes/agents.py`**

```python
from typing import List, Optional
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel

from models.database import get_db
from agents.definitions import AgentRole, get_agent_config, get_all_agent_configs
from services.orchestrator import orchestrator

router = APIRouter()


class AgentInfo(BaseModel):
    role: str
    name: str
    description: str
    handoff_description: str


class AgentChatRequest(BaseModel):
    project_id: str
    session_id: str
    agent_role: str
    message: str


class AgentChatResponse(BaseModel):
    agent_role: str
    response: str


@router.get("/", response_model=List[AgentInfo])
async def list_agents():
    """List all available agents."""
    configs = get_all_agent_configs()
    return [
        AgentInfo(
            role=c.role.value,
            name=c.name,
            description=c.description,
            handoff_description=c.handoff_description
        )
        for c in configs
    ]


@router.get("/{agent_role}", response_model=AgentInfo)
async def get_agent(agent_role: str):
    """Get info about a specific agent."""
    try:
        role = AgentRole(agent_role)
        config = get_agent_config(role)
        return AgentInfo(
            role=config.role.value,
            name=config.name,
            description=config.description,
            handoff_description=config.handoff_description
        )
    except ValueError:
        raise HTTPException(status_code=404, detail="Agent not found")


@router.post("/chat", response_model=AgentChatResponse)
async def chat_with_agent(
    request: AgentChatRequest,
    db: AsyncSession = Depends(get_db)
):
    """Chat directly with a specific agent."""
    try:
        role = AgentRole(request.agent_role)
    except ValueError:
        raise HTTPException(status_code=404, detail="Agent not found")

    response = await orchestrator.run_single_agent(
        agent_role=role,
        session_id=request.session_id,
        project_id=request.project_id,
        message=request.message,
        db=db
    )

    return AgentChatResponse(
        agent_role=request.agent_role,
        response=response
    )
```

**File: `backend/routes/websocket.py`**

```python
import json
import asyncio
from typing import Dict, Set
from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from models.database import get_db, AsyncSessionLocal
from services.orchestrator import orchestrator
from services.codex_manager import codex_manager
from agents.definitions import AgentRole

router = APIRouter()


class ConnectionManager:
    """Manages WebSocket connections."""

    def __init__(self):
        # session_id -> set of websockets
        self.active_connections: Dict[str, Set[WebSocket]] = {}

    async def connect(self, websocket: WebSocket, session_id: str):
        await websocket.accept()
        if session_id not in self.active_connections:
            self.active_connections[session_id] = set()
        self.active_connections[session_id].add(websocket)

    def disconnect(self, websocket: WebSocket, session_id: str):
        if session_id in self.active_connections:
            self.active_connections[session_id].discard(websocket)
            if not self.active_connections[session_id]:
                del self.active_connections[session_id]

    async def broadcast(self, session_id: str, message: dict):
        if session_id in self.active_connections:
            dead_connections = set()
            for connection in self.active_connections[session_id]:
                try:
                    await connection.send_json(message)
                except:
                    dead_connections.add(connection)

            for conn in dead_connections:
                self.active_connections[session_id].discard(conn)


manager = ConnectionManager()


@router.websocket("/session/{session_id}")
async def session_websocket(
    websocket: WebSocket,
    session_id: str
):
    """
    WebSocket for real-time session updates.

    Sends:
    - workflow_progress: Stage updates
    - agent_message: Agent responses
    - codex_output: Real-time codex output
    - error: Error messages
    """
    await manager.connect(websocket, session_id)

    try:
        while True:
            data = await websocket.receive_json()
            action = data.get("action")

            if action == "start_workflow":
                # Start workflow with progress callbacks
                async def on_progress(stage: str, message: str):
                    await manager.broadcast(session_id, {
                        "type": "workflow_progress",
                        "stage": stage,
                        "message": message
                    })

                async with AsyncSessionLocal() as db:
                    project_id = data.get("project_id")
                    user_request = data.get("user_request")

                    result = await orchestrator.start_workflow(
                        session_id=session_id,
                        project_id=project_id,
                        user_request=user_request,
                        db=db,
                        on_progress=on_progress
                    )

                    await websocket.send_json({
                        "type": "workflow_complete",
                        "result": result
                    })

            elif action == "chat":
                # Direct chat with agent
                async with AsyncSessionLocal() as db:
                    agent_role = AgentRole(data.get("agent_role"))
                    message = data.get("message")
                    project_id = data.get("project_id")

                    response = await orchestrator.run_single_agent(
                        agent_role=agent_role,
                        session_id=session_id,
                        project_id=project_id,
                        message=message,
                        db=db
                    )

                    await websocket.send_json({
                        "type": "agent_message",
                        "agent_role": agent_role.value,
                        "content": response
                    })

            elif action == "subscribe_codex":
                # Subscribe to codex output stream
                task_id = data.get("task_id")

                async for line in codex_manager.stream_output(task_id):
                    await websocket.send_json({
                        "type": "codex_output",
                        "task_id": task_id,
                        "line": line
                    })

            elif action == "ping":
                await websocket.send_json({"type": "pong"})

    except WebSocketDisconnect:
        manager.disconnect(websocket, session_id)
    except Exception as e:
        await websocket.send_json({
            "type": "error",
            "message": str(e)
        })
        manager.disconnect(websocket, session_id)


@router.websocket("/terminal/{task_id}")
async def terminal_websocket(
    websocket: WebSocket,
    task_id: str
):
    """WebSocket for streaming codex terminal output."""
    await websocket.accept()

    try:
        async for line in codex_manager.stream_output(task_id):
            await websocket.send_json({
                "type": "output",
                "line": line
            })

        await websocket.send_json({
            "type": "complete"
        })
    except WebSocketDisconnect:
        pass
    except Exception as e:
        await websocket.send_json({
            "type": "error",
            "message": str(e)
        })
```

**File: `backend/routes/files.py`**

```python
import os
import hashlib
import aiofiles
from typing import List
from datetime import datetime

from fastapi import APIRouter, Depends, HTTPException, UploadFile, File, Form
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from pydantic import BaseModel

from models.database import Project, ProjectFile, get_db
from config import settings

router = APIRouter()


class FileResponse(BaseModel):
    id: str
    filename: str
    file_type: str
    file_size: int
    uploaded_at: datetime

    class Config:
        from_attributes = True


@router.post("/upload/{project_id}", response_model=FileResponse)
async def upload_file(
    project_id: str,
    file: UploadFile = File(...),
    db: AsyncSession = Depends(get_db)
):
    """Upload a file to a project's workspace."""
    project = await db.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")

    # Read file content
    content = await file.read()
    content_hash = hashlib.sha256(content).hexdigest()

    # Save file to workspace
    file_path = os.path.join(project.workspace_path, file.filename)

    async with aiofiles.open(file_path, 'wb') as f:
        await f.write(content)

    # Create database record
    db_file = ProjectFile(
        project_id=project_id,
        filename=file.filename,
        file_path=file_path,
        file_type=file.content_type,
        file_size=len(content),
        content_hash=content_hash
    )

    db.add(db_file)
    await db.commit()
    await db.refresh(db_file)

    return FileResponse(
        id=str(db_file.id),
        filename=db_file.filename,
        file_type=db_file.file_type,
        file_size=db_file.file_size,
        uploaded_at=db_file.uploaded_at
    )


@router.get("/project/{project_id}", response_model=List[FileResponse])
async def list_project_files(
    project_id: str,
    db: AsyncSession = Depends(get_db)
):
    """List all uploaded files for a project."""
    result = await db.execute(
        select(ProjectFile)
        .where(ProjectFile.project_id == project_id)
        .order_by(ProjectFile.uploaded_at.desc())
    )
    files = result.scalars().all()

    return [
        FileResponse(
            id=str(f.id),
            filename=f.filename,
            file_type=f.file_type,
            file_size=f.file_size,
            uploaded_at=f.uploaded_at
        )
        for f in files
    ]


@router.delete("/{file_id}")
async def delete_file(
    file_id: str,
    db: AsyncSession = Depends(get_db)
):
    """Delete an uploaded file."""
    file = await db.get(ProjectFile, file_id)
    if not file:
        raise HTTPException(status_code=404, detail="File not found")

    # Delete from filesystem
    if os.path.exists(file.file_path):
        os.remove(file.file_path)

    # Delete from database
    await db.delete(file)
    await db.commit()

    return {"status": "deleted", "file_id": file_id}
```

**File: `backend/routes/__init__.py`**

```python
from .projects import router as projects_router
from .sessions import router as sessions_router
from .agents import router as agents_router
from .files import router as files_router
from .websocket import router as websocket_router

__all__ = [
    "projects_router",
    "sessions_router",
    "agents_router",
    "files_router",
    "websocket_router"
]
```

**File: `backend/config.py`**

```python
import os
from typing import List
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    # Application
    APP_NAME: str = "Codex Multi-Agent System"
    DEBUG: bool = os.getenv("DEBUG", "false").lower() == "true"

    # Database
    DATABASE_URL: str = os.getenv(
        "DATABASE_URL",
        "postgresql+asyncpg://codex_agent:your_secure_password@localhost:5432/codex_agents_db"
    )

    # Workspaces
    WORKSPACES_ROOT: str = os.getenv("WORKSPACES_ROOT", "/workspaces")

    # CORS
    CORS_ORIGINS: List[str] = ["http://localhost:3000", "http://localhost:5173"]

    # OpenAI (for agents)
    OPENAI_API_KEY: str = os.getenv("OPENAI_API_KEY", "")

    # Redis (for task queue)
    REDIS_URL: str = os.getenv("REDIS_URL", "redis://localhost:6379")

    class Config:
        env_file = ".env"


settings = Settings()
```

**Checklist:**
- [ ] Create main.py FastAPI application
- [ ] Implement all route modules
- [ ] Implement WebSocket handlers
- [ ] Add file upload support
- [ ] Create configuration module
- [ ] Test all API endpoints

---

## Phase 7: Deployment

### 7.1 Systemd Service Configuration

**File: `/etc/systemd/system/codex-agents.service`**

```ini
[Unit]
Description=Codex Multi-Agent System
After=network.target postgresql.service redis.service

[Service]
Type=exec
User=codex
Group=codex
WorkingDirectory=/opt/codex-agents/backend
Environment="PATH=/opt/codex-agents/.venv/bin:/usr/local/bin:/usr/bin"
Environment="OPENAI_API_KEY=your_api_key"
Environment="DATABASE_URL=postgresql+asyncpg://codex_agent:password@localhost:5432/codex_agents_db"
Environment="WORKSPACES_ROOT=/workspaces"
ExecStart=/opt/codex-agents/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 7.2 Nginx Configuration

**File: `/etc/nginx/sites-available/codex-agents`**

```nginx
server {
    listen 80;
    server_name your-domain.com;

    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Frontend static files
    location / {
        root /opt/codex-agents/frontend/dist;
        try_files $uri $uri/ /index.html;
    }

    # API proxy
    location /api {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket proxy
    location /ws {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }
}
```

### 7.3 Deployment Script

**File: `deploy.sh`**

```bash
#!/bin/bash
set -e

echo "=== Codex Multi-Agent System Deployment ==="

# Configuration
APP_DIR="/opt/codex-agents"
REPO_URL="your-repo-url"

# Update system
sudo apt update && sudo apt upgrade -y

# Clone/update repository
if [ -d "$APP_DIR" ]; then
    cd $APP_DIR && git pull
else
    git clone $REPO_URL $APP_DIR
fi

cd $APP_DIR

# Backend setup
echo "Setting up backend..."
cd backend
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt

# Run migrations
python -c "import asyncio; from models.database import init_db; asyncio.run(init_db())"

# Frontend setup
echo "Setting up frontend..."
cd ../frontend
npm install
npm run build

# System services
echo "Configuring services..."
sudo cp $APP_DIR/deploy/codex-agents.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable codex-agents
sudo systemctl restart codex-agents

# Nginx
sudo cp $APP_DIR/deploy/nginx.conf /etc/nginx/sites-available/codex-agents
sudo ln -sf /etc/nginx/sites-available/codex-agents /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

echo "=== Deployment Complete ==="
echo "Service status:"
sudo systemctl status codex-agents --no-pager
```

**Checklist:**
- [ ] Create systemd service file
- [ ] Configure nginx reverse proxy
- [ ] Set up SSL with Let's Encrypt
- [ ] Create deployment script
- [ ] Configure firewall rules
- [ ] Set up log rotation
- [ ] Create backup scripts

---

## Project Structure

```
codex-agents/
├── backend/
│   ├── main.py
│   ├── config.py
│   ├── requirements.txt
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── definitions.py
│   │   └── agent_factory.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── database.py
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── projects.py
│   │   ├── sessions.py
│   │   ├── agents.py
│   │   ├── files.py
│   │   └── websocket.py
│   └── services/
│       ├── __init__.py
│       ├── codex_manager.py
│       └── orchestrator.py
├── frontend/
│   └── (see agent_web_app_frontend.md)
├── deploy/
│   ├── codex-agents.service
│   ├── nginx.conf
│   └── deploy.sh
├── workspaces/
│   └── (project workspaces created here)
└── README.md
```

---

## Implementation Checklist

### Phase 1: Infrastructure
- [ ] Provision Google Cloud VM
- [ ] Install system dependencies
- [ ] Install and authenticate codex-cli
- [ ] Set up PostgreSQL database
- [ ] Configure Redis

### Phase 2: Database
- [ ] Implement all SQLAlchemy models
- [ ] Run initial migrations
- [ ] Test database operations

### Phase 3: Codex Manager
- [ ] Implement async codex execution
- [ ] Add output streaming
- [ ] Implement task cancellation
- [ ] Test with real codex commands

### Phase 4: Agents
- [ ] Define all 4 agent configurations
- [ ] Implement agent factory
- [ ] Create codex execution tools
- [ ] Test individual agents

### Phase 5: Orchestration
- [ ] Implement workflow orchestrator
- [ ] Add automatic handoffs
- [ ] Implement single-agent mode
- [ ] Test complete workflows

### Phase 6: API
- [ ] Implement all REST endpoints
- [ ] Implement WebSocket handlers
- [ ] Add file upload support
- [ ] Test all endpoints

### Phase 7: Deployment
- [ ] Configure systemd service
- [ ] Set up nginx
- [ ] Configure SSL
- [ ] Deploy and verify

---

## Key Design Decisions

1. **Agents prompt codex, don't code directly**: All 4 agents use `run_codex` tool to execute codex-cli. They analyze, plan, and formulate prompts but never write code themselves.

2. **Async codex execution**: Codex runs in background processes. Agents can start tasks and check status later, or wait for completion.

3. **Task completion callbacks**: When codex finishes, the system updates the database and can notify waiting agents or WebSocket clients.

4. **Project isolation**: Each project has its own workspace directory. Codex runs in that directory, keeping projects separate.

5. **Session continuity**: AGENTS.md and all conversation history persist in PostgreSQL, allowing workflows to resume and providing full audit trail.

6. **Real-time streaming**: WebSocket connections stream codex output to the frontend for live terminal view.
