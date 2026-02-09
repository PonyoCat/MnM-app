name: "MnM-App Phase 1: Python Backend - League of Legends Live Stats Tracker"
description: |

## Purpose
Build a Python backend that connects to the League of Legends client, polls live game data for all 10 players during active matches, enriches it with pre/post-game data from the LCU and Riot APIs, stores structured stats in a local SQLite database, and exposes the data through a FastAPI REST API. This is Phase 1 (backend only); the frontend will be addressed in a separate PRP.

## Core Principles
1. **Context is King**: Include ALL necessary documentation, examples, and caveats
2. **Validation Loops**: Provide executable tests/lints the AI can run and fix
3. **Information Dense**: Use keywords and patterns from the codebase
4. **Progressive Success**: Start simple, validate, then enhance
5. **Global rules**: Be sure to follow all rules in AGENTS.md

---

## Goal
Create a production-ready Python backend service that:
1. Detects when the League of Legends client is running and connects to the LCU API
2. Detects when an active game is in progress and polls the Live Client Data API every second
3. Stores per-game snapshots and final stats in a local SQLite database using delta-compression
4. Enriches game data with ranked info, champion mastery, and match history from the Riot Games API
5. Exposes all collected data through a FastAPI REST API with WebSocket support for live updates
6. Runs as a lightweight background process (~50-100 MB RAM, <5% CPU)

## Why
- **User value**: Provides MnM hobby league members with detailed game-by-game stats, performance trends, and live match data without relying on third-party websites
- **Integration**: Lays the foundation for a future frontend dashboard (Phase 2) and team analytics features
- **Problems solved**: No existing tool combines live in-game polling, pre-game scouting (opponent ranks/mastery), and persistent historical storage in a single local application
- **Privacy**: All data stays local on the player's machine; only Riot API calls go to remote servers

## What
A Python backend service that runs alongside the League client, providing:
- Automatic detection of client state (closed, lobby, champ select, in-game, post-game)
- Real-time polling of all 10 players' stats during active games (KDA, CS, items, gold, runes, abilities)
- Pre-game enrichment: opponent rank, champion mastery, recent match history
- Post-game capture: full match details from Match-v5 API
- Persistent storage of all game data with efficient delta compression
- REST API + WebSocket for live data consumption by future frontend

### Success Criteria
- [ ] Backend detects League client state transitions (closed -> lobby -> champ select -> in-game -> post-game)
- [ ] Live Client Data API polled every 1 second during active games with <50ms processing overhead
- [ ] All 10 players' KDA, CS, items, runes, and summoner spells captured each poll
- [ ] Active player's extended stats captured (gold, abilities, full champion stats)
- [ ] Delta-only storage keeps per-game data under 15 MB
- [ ] LCU API provides lobby/champ-select/post-game data via WebSocket events
- [ ] Riot API enriches players with rank, mastery, and match history (respecting rate limits)
- [ ] SQLite database stores structured game data with queryable schemas
- [ ] FastAPI exposes REST endpoints for games list, game detail, player stats, live game state
- [ ] WebSocket endpoint streams live game updates to connected clients
- [ ] Process uses <100 MB RAM and <5% CPU during active polling
- [ ] All tests pass, linting clean, type checking clean

## Prerequisites

### Riot API Key (required for enrichment features)
You need a Riot Games API key for ranked/mastery/match-history lookups. The core live polling works without it.

1. Go to https://developer.riotgames.com
2. Log in with your Riot account
3. Generate a **Development API Key** (free, expires every 24 hours)
4. For a permanent key: register a "Personal Project" application (takes a few days for approval)

**Development key rate limits**: 20 requests/second, 100 requests/2 minutes (per region)

## All Needed Context

### API Architecture Overview

Three complementary APIs serve different phases of the game lifecycle:

| API | When Available | Data Provided | Auth | Rate Limits |
|-----|----------------|---------------|------|-------------|
| **Live Client Data API** (localhost:2999) | During active game only | Real-time KDA, CS, items, gold, runes, abilities, events for all 10 players | None (local) | None |
| **LCU API** (localhost:random port) | When League client is open | Client state transitions, champ select picks/bans, post-game stats | HTTP Basic (riot:lockfile_password) | None |
| **Riot Games Official API** (remote) | Anytime | Ranked tier/LP, champion mastery, match history, account lookup (Riot ID -> PUUID) | X-Riot-Token header | 20 req/sec, 100 req/2min (dev key) |

### What CAN be logged in real-time (all 10 players)
- Kills, deaths, assists, ward score
- CS (caveat: updates in increments of 10 - known Riot limitation)
- Items, runes, summoner spells, champion, level, team
- Game events with timestamps (champion kills, dragon, baron, turrets, inhibitors)
- Whether a player is dead + respawn timer

### What CAN be logged for the active player only
- Current gold, full champion stats (~30 fields: AD, AP, armor, MR, attack speed, etc.)
- Ability levels (Q/W/E/R)
- Full rune details

### What CANNOT be obtained live
- Player positions (x,y coordinates)
- Exact opponent gold amounts
- Ability cooldowns for opponents
- Fog-of-war restricted information
- Damage dealt/taken breakdowns (only available post-game via Match-v5)

### Resource Budget
| Metric | Estimate |
|--------|----------|
| RAM | 50-100 MB (Python process) |
| CPU | <5% of a modern CPU |
| Network | Almost entirely local (localhost). ~10-50 remote Riot API calls per game |
| Response size per poll | ~4-5 KB (game start) growing to ~30-80 KB (mid-game with 10 players + items + events) |
| Raw data per 30-min game | ~54-144 MB if storing every 1-sec poll unchanged |
| With delta compression | ~5-15 MB per game (only store changed fields) |
| DB storage per game | 1-5 MB structured in SQLite |

### Documentation & References
```yaml
# MUST READ - Include these in your context window

# --- Live Client Data API (PRIMARY for in-game) ---
- url: https://developer.riotgames.com/docs/lol#game-client-api
  why: Official documentation for Live Client Data API endpoints
  critical: Only available during active games on localhost:2999, uses self-signed SSL cert

- url: https://static.developer.riotgames.com/docs/lol/liveclientdata_sample.json
  why: Sample JSON response from /liveclientdata/allgamedata showing full structure

# --- LCU API (pre/post game client data) ---
- url: https://hextechdocs.dev/getting-started-with-the-lcu-api/
  why: Unofficial but comprehensive LCU API guide - lockfile parsing, authentication, endpoints
  critical: LCU API is not officially supported by Riot; it may change without notice

- url: https://lcu-driver.readthedocs.io/
  why: Python library docs for lcu-driver v3.0.2 - handles lockfile, auth, WebSocket automatically

- url: https://www.mingweisamuel.com/lcu-schema/tool/
  why: Interactive LCU API endpoint browser - complete endpoint reference

# --- Riot Games Official API (remote, historical/ranked) ---
- url: https://developer.riotgames.com/apis
  why: Official Riot API endpoint reference for all endpoints

- url: https://developer.riotgames.com/docs/lol#routing-values
  why: Regional routing (americas, europe, asia for Account-v1 and Match-v5)
  critical: Account-v1 and Match-v5 use REGIONAL routes; Summoner-v4 and League-v1 use PLATFORM routes

- url: https://developer.riotgames.com/docs/portal#web-apis_rate-limiting
  why: Rate limiting details - must implement proper rate limiting to avoid 429 errors

# --- Python Libraries ---
- url: https://fastapi.tiangolo.com/
  why: FastAPI framework for REST API + WebSocket endpoints

- url: https://docs.sqlalchemy.org/en/20/
  why: SQLAlchemy 2.0 ORM for SQLite database models
  critical: Use 2.0-style mapped_column and DeclarativeBase, NOT legacy 1.x patterns

- url: https://docs.pydantic.dev/latest/
  why: Pydantic v2 for data validation and serialization
  critical: Use model_validator not validator, use ConfigDict not class Config

- url: https://www.python-httpx.org/
  why: httpx for async HTTP client (Live Client Data API + Riot API calls)
  critical: Use verify=False for Live Client Data API (self-signed SSL cert)
```

### Current Codebase tree
```bash
MnM-app/
├── .claude/
│   ├── commands/
│   │   └── commands/
│   │       ├── execute-prp.md
│   │       ├── generate-prp.md
│   │       └── ultimate_validate_command.md
│   └── mcp.servers.json
├── examples/
│   └── examples/
│       └── .gitkeep
├── PRPs/
│   └── PRPs/
│       ├── templates/
│       │   └── prp_base.md
│       ├── EXAMPLE_multi_agent_prp.md
│       └── MnM-app-prp.md            # THIS FILE
├── .gitignore
├── AGENTS.md
├── CLAUDE-example.md
└── README.md
```

### Desired Codebase tree with files to be added and responsibility of file
```bash
MnM-app/
├── .claude/                            # (existing) Claude configuration
├── PRPs/                               # (existing) Product Requirement Plans
├── examples/                           # (existing)
├── backend/                            # NEW: All backend code
│   ├── __init__.py                     # Package init
│   ├── main.py                         # FastAPI app entry point + lifespan events
│   ├── config.py                       # Settings via pydantic-settings (API keys, polling interval, DB path)
│   ├── models/
│   │   ├── __init__.py
│   │   ├── database.py                 # SQLAlchemy ORM models (Game, PlayerInGame, GameSnapshot, GameEvent, KnownPlayer)
│   │   └── schemas.py                  # Pydantic v2 schemas for API responses and internal data
│   ├── services/
│   │   ├── __init__.py
│   │   ├── client_watcher.py           # Detects League client state, manages lifecycle (state machine)
│   │   ├── live_data_poller.py         # Polls Live Client Data API every 1sec during active games
│   │   ├── lcu_connector.py            # Connects to LCU API, handles WebSocket events
│   │   ├── riot_api_client.py          # Riot Games API client with rate limiting
│   │   └── data_processor.py           # Delta compression, data transformation, storage orchestration
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes.py                   # FastAPI REST routes (games, players, live, stats)
│   │   └── websocket.py               # WebSocket endpoint for live game streaming
│   ├── db/
│   │   ├── __init__.py
│   │   ├── engine.py                   # SQLAlchemy engine + session factory (SQLite via aiosqlite)
│   │   └── repository.py              # Database CRUD operations (queries, inserts, aggregations)
│   └── utils/
│       ├── __init__.py
│       ├── rate_limiter.py             # Token bucket rate limiter for Riot API (dual window)
│       └── logging_config.py           # Structured logging setup
├── tests/
│   ├── __init__.py
│   ├── conftest.py                     # Shared fixtures (test DB, mock HTTP responses, sample game data)
│   ├── test_config.py
│   ├── test_live_data_poller.py
│   ├── test_lcu_connector.py
│   ├── test_riot_api_client.py
│   ├── test_data_processor.py
│   ├── test_models.py
│   ├── test_repository.py
│   └── test_api.py                     # FastAPI endpoint tests (TestClient)
├── .env.example                        # Environment variable template
├── pyproject.toml                      # Project metadata, dependencies, tool configs (ruff, mypy, pytest)
├── .gitignore                          # (update) Add *.db, .env, __pycache__, etc.
├── AGENTS.md                           # (update) Add project context section
└── README.md                           # (update) Add architecture, setup, usage docs
```

### Known Gotchas & Library Quirks
```python
# CRITICAL: Live Client Data API uses a self-signed SSL certificate
# You MUST use verify=False (or Riot's root cert) when calling localhost:2999
# httpx example: httpx.AsyncClient(verify=False)

# CRITICAL: Live Client Data API is ONLY available during an active game
# Calling it when no game is running returns connection refused, not a 404
# The poller must handle ConnectionRefusedError / httpx.ConnectError gracefully

# CRITICAL: creepScore (CS) in Live Client Data updates in INCREMENTS OF 10
# This is a known Riot limitation - do NOT assume CS values are exact

# CRITICAL: LCU API port changes every time the client restarts
# Parse the lockfile at: C:\Riot Games\League of Legends\lockfile
# Format: processName:pid:port:password:protocol
# Use HTTP Basic auth with username "riot" and password from lockfile

# CRITICAL: lcu-driver v3.0.2 handles lockfile parsing and auth automatically
# But it uses aiohttp internally, which may conflict with httpx
# Use lcu-driver for LCU only and httpx for everything else

# CRITICAL: Riot API uses Riot ID (gameName#tagLine) as the primary identifier
# Summoner name endpoints are DEPRECATED - always use Account-v1 -> PUUID flow
# Lookup chain: Riot ID -> Account-v1 (PUUID) -> Summoner-v4 (summoner ID) -> League-v1 (rank)

# CRITICAL: Riot API regional routing matters
# Account-v1 and Match-v5 use REGIONAL routes (americas, europe, asia, sea)
# Summoner-v4, League-v1, Champion-Mastery-v4 use PLATFORM routes (euw1, na1, kr, etc.)
# For EUW: platform=euw1, regional=europe

# CRITICAL: SQLAlchemy 2.0 patterns required
# Use DeclarativeBase, mapped_column(), Mapped[] type hints
# Do NOT use legacy Column(), declarative_base()

# CRITICAL: Pydantic v2 patterns required
# Use model_validator(mode='before') NOT @validator
# Use ConfigDict(from_attributes=True) NOT class Config: orm_mode = True

# CRITICAL: FastAPI async context
# All endpoint functions and background tasks must be async
# Use asyncio.create_task() for background polling, NOT threading

# CRITICAL: SQLite + async requires aiosqlite
# Use create_async_engine("sqlite+aiosqlite:///path/to/db.sqlite")

# GOTCHA: Live Client Data /liveclientdata/allgamedata grows over time
# ~4-5 KB at game start to ~30-80 KB mid-game as events accumulate
# The events array grows unbounded throughout the game
# For delta storage: only store new events since last poll, not the full array each time

# GOTCHA: Rate limiter for Riot API must be global across all coroutines
# Dev key limits: 20 requests per second, 100 requests per 2 minutes
# Use a token bucket or sliding window limiter shared across the application
```

## Implementation Blueprint

### Data models and structure

```python
# ============================================================
# backend/models/database.py - SQLAlchemy ORM Models
# ============================================================
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import String, Integer, Float, Boolean, DateTime, JSON, ForeignKey
from datetime import datetime
from typing import Optional, List

class Base(DeclarativeBase):
    pass

class Game(Base):
    """A single League of Legends game session."""
    __tablename__ = "games"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    riot_match_id: Mapped[Optional[str]] = mapped_column(String(30), unique=True, nullable=True)
    # e.g., "EUW1_1234567890" from Match-v5 API (populated post-game)

    game_mode: Mapped[str] = mapped_column(String(30))  # "CLASSIC", "ARAM", etc.
    game_type: Mapped[str] = mapped_column(String(30))   # "MATCHED_GAME", "CUSTOM_GAME"
    map_name: Mapped[str] = mapped_column(String(50))    # "Map11" (Summoner's Rift)

    started_at: Mapped[datetime] = mapped_column(DateTime)
    ended_at: Mapped[Optional[datetime]] = mapped_column(DateTime, nullable=True)
    game_length_seconds: Mapped[Optional[float]] = mapped_column(Float, nullable=True)

    winning_team: Mapped[Optional[str]] = mapped_column(String(10), nullable=True)
    # "ORDER" (blue) or "CHAOS" (red), null if not yet finished

    is_live: Mapped[bool] = mapped_column(Boolean, default=True)

    # Relationships
    players: Mapped[List["PlayerInGame"]] = relationship(back_populates="game", cascade="all, delete-orphan")
    snapshots: Mapped[List["GameSnapshot"]] = relationship(back_populates="game", cascade="all, delete-orphan")
    events: Mapped[List["GameEvent"]] = relationship(back_populates="game", cascade="all, delete-orphan")

    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)


class PlayerInGame(Base):
    """A player's participation in a specific game (10 per game)."""
    __tablename__ = "players_in_games"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    game_id: Mapped[int] = mapped_column(ForeignKey("games.id"))

    # Identity
    summoner_name: Mapped[str] = mapped_column(String(50))
    riot_id_name: Mapped[Optional[str]] = mapped_column(String(50), nullable=True)
    riot_id_tagline: Mapped[Optional[str]] = mapped_column(String(10), nullable=True)
    puuid: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)

    # In-game info (from Live Client Data)
    champion_name: Mapped[str] = mapped_column(String(30))
    team: Mapped[str] = mapped_column(String(10))  # "ORDER" (blue) or "CHAOS" (red)
    position: Mapped[Optional[str]] = mapped_column(String(10), nullable=True)
    is_active_player: Mapped[bool] = mapped_column(Boolean, default=False)

    # Summoner spells
    spell_d: Mapped[Optional[str]] = mapped_column(String(30), nullable=True)
    spell_f: Mapped[Optional[str]] = mapped_column(String(30), nullable=True)

    # Runes
    rune_primary_tree: Mapped[Optional[str]] = mapped_column(String(30), nullable=True)
    rune_secondary_tree: Mapped[Optional[str]] = mapped_column(String(30), nullable=True)
    runes_json: Mapped[Optional[dict]] = mapped_column(JSON, nullable=True)

    # Pre-game enrichment (from Riot API)
    ranked_tier: Mapped[Optional[str]] = mapped_column(String(20), nullable=True)
    ranked_division: Mapped[Optional[str]] = mapped_column(String(5), nullable=True)
    ranked_lp: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    champion_mastery_level: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    champion_mastery_points: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)

    # Final stats (populated at game end or from Match-v5)
    final_kills: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    final_deaths: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    final_assists: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    final_cs: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    final_gold: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    final_items_json: Mapped[Optional[list]] = mapped_column(JSON, nullable=True)
    final_damage_dealt: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    final_damage_taken: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    final_vision_score: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)

    # Post-game detailed stats from Match-v5 (stored as JSON for flexibility)
    match_v5_stats_json: Mapped[Optional[dict]] = mapped_column(JSON, nullable=True)

    game: Mapped["Game"] = relationship(back_populates="players")


class GameSnapshot(Base):
    """A delta snapshot of player stats at a point in time during the game.
    Only stores fields that changed since the previous snapshot."""
    __tablename__ = "game_snapshots"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    game_id: Mapped[int] = mapped_column(ForeignKey("games.id"))
    game_time_seconds: Mapped[float] = mapped_column(Float)

    polled_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    # Delta data: only fields that changed since last snapshot, per player
    # Structure: {"SummonerName": {"kills": 3, "deaths": 1, "items": [...]}, ...}
    player_deltas_json: Mapped[dict] = mapped_column(JSON)

    # Active player extended data delta (gold, abilities, stats - only for local player)
    active_player_delta_json: Mapped[Optional[dict]] = mapped_column(JSON, nullable=True)

    game: Mapped["Game"] = relationship(back_populates="snapshots")


class GameEvent(Base):
    """In-game events (kills, dragon, baron, turret, inhibitor, etc.)."""
    __tablename__ = "game_events"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    game_id: Mapped[int] = mapped_column(ForeignKey("games.id"))

    event_name: Mapped[str] = mapped_column(String(50))  # "ChampionKill", "DragonKill", etc.
    event_time_seconds: Mapped[float] = mapped_column(Float)
    event_data_json: Mapped[dict] = mapped_column(JSON)

    game: Mapped["Game"] = relationship(back_populates="events")


class KnownPlayer(Base):
    """Cached player identity and rank info to reduce Riot API calls."""
    __tablename__ = "known_players"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    puuid: Mapped[str] = mapped_column(String(100), unique=True, index=True)
    riot_id_name: Mapped[str] = mapped_column(String(50))
    riot_id_tagline: Mapped[str] = mapped_column(String(10))
    summoner_id: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)

    ranked_tier: Mapped[Optional[str]] = mapped_column(String(20), nullable=True)
    ranked_division: Mapped[Optional[str]] = mapped_column(String(5), nullable=True)
    ranked_lp: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)
    rank_updated_at: Mapped[Optional[datetime]] = mapped_column(DateTime, nullable=True)

    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

```python
# ============================================================
# backend/models/schemas.py - Pydantic v2 Schemas (key models only)
# ============================================================
from pydantic import BaseModel, Field, ConfigDict
from typing import Optional, List
from datetime import datetime

# --- Live Client Data API Response Schemas ---

class LivePlayerScore(BaseModel):
    assists: int = 0
    creepScore: int = 0  # NOTE: updates in increments of 10
    deaths: int = 0
    kills: int = 0
    wardScore: float = 0.0

class LivePlayer(BaseModel):
    """A player from /liveclientdata/playerlist."""
    championName: str
    isBot: bool = False
    isDead: bool = False
    items: list = []
    level: int = 1
    position: str = ""
    respawnTimer: float = 0.0
    runes: dict = {}
    scores: LivePlayerScore = LivePlayerScore()
    summonerName: str = ""
    summonerSpells: dict = {}
    team: str = ""  # "ORDER" or "CHAOS"

class LiveActivePlayerChampionStats(BaseModel):
    """~30 champion stat fields for the active player."""
    abilityHaste: float = 0.0
    abilityPower: float = 0.0
    armor: float = 0.0
    armorPenetrationFlat: float = 0.0
    armorPenetrationPercent: float = 0.0
    attackDamage: float = 0.0
    attackRange: float = 0.0
    attackSpeed: float = 0.0
    bonusArmorPenetrationPercent: float = 0.0
    bonusMagicPenetrationPercent: float = 0.0
    critChance: float = 0.0
    critDamage: float = 0.0
    currentHealth: float = 0.0
    healthRegenRate: float = 0.0
    lifeSteal: float = 0.0
    magicPenetrationFlat: float = 0.0
    magicPenetrationPercent: float = 0.0
    magicResist: float = 0.0
    maxHealth: float = 0.0
    moveSpeed: float = 0.0
    physicalLethality: float = 0.0
    resourceMax: float = 0.0
    resourceRegenRate: float = 0.0
    resourceType: str = ""
    resourceValue: float = 0.0
    tenacity: float = 0.0

class LiveActivePlayer(BaseModel):
    """Extended data for the local active player from /liveclientdata/activeplayer."""
    abilities: dict = {}
    championStats: LiveActivePlayerChampionStats = LiveActivePlayerChampionStats()
    currentGold: float = 0.0
    level: int = 1
    summonerName: str = ""

class LiveGameStats(BaseModel):
    """Game-level stats from /liveclientdata/gamestats."""
    gameMode: str = ""
    gameTime: float = 0.0
    mapName: str = ""
    mapNumber: int = 0
    mapTerrain: str = ""

class LiveAllGameData(BaseModel):
    """Top-level response from /liveclientdata/allgamedata."""
    activePlayer: LiveActivePlayer = LiveActivePlayer()
    allPlayers: List[LivePlayer] = []
    events: dict = {}  # {"Events": [...]}
    gameData: LiveGameStats = LiveGameStats()

# --- API Response Schemas ---

class GameSummary(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    game_mode: str
    started_at: datetime
    game_length_seconds: Optional[float] = None
    is_live: bool

class GameDetail(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    game_mode: str
    game_type: str
    started_at: datetime
    ended_at: Optional[datetime] = None
    game_length_seconds: Optional[float] = None
    is_live: bool
    winning_team: Optional[str] = None
    players: list = []
    events: list = []

class LiveGameState(BaseModel):
    """Current live game state for WebSocket streaming."""
    is_in_game: bool
    game_time: Optional[float] = None
    players: Optional[List[LivePlayer]] = None
    active_player: Optional[LiveActivePlayer] = None
    game_stats: Optional[LiveGameStats] = None
    recent_events: Optional[list] = None

class ClientState(BaseModel):
    """Current state of the League client."""
    client_running: bool = False
    client_state: str = "closed"
    # States: "closed", "lobby", "champ_select", "in_game", "post_game"
    current_game_id: Optional[int] = None
    active_player_name: Optional[str] = None
```

### List of tasks to be completed in order

```yaml
Task 1: Project Setup and Configuration
CREATE pyproject.toml:
  - Python 3.11+ requirement
  - Dependencies: fastapi, uvicorn[standard], httpx, sqlalchemy[asyncio], aiosqlite,
    pydantic, pydantic-settings, lcu-driver, websockets
  - Dev dependencies: pytest, pytest-asyncio, pytest-httpx, ruff, mypy
  - Tool configs: [tool.ruff], [tool.mypy], [tool.pytest.ini_options]

CREATE .env.example:
  - RIOT_API_KEY=RGAPI-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  - RIOT_PLATFORM=euw1
  - RIOT_REGION=europe
  - LEAGUE_CLIENT_PATH=C:\Riot Games\League of Legends
  - DB_PATH=./data/mnm.db
  - POLLING_INTERVAL_SECONDS=1.0
  - LOG_LEVEL=INFO
  - API_HOST=127.0.0.1
  - API_PORT=8000

CREATE backend/config.py:
  - Use pydantic_settings.BaseSettings loading from .env with defaults
  - Validate RIOT_API_KEY format if provided (starts with "RGAPI-")
  - Derive LEAGUE_LOCKFILE_PATH from LEAGUE_CLIENT_PATH

UPDATE .gitignore:
  - Add: .env, *.db, data/, __pycache__/, .mypy_cache/, .pytest_cache/

Task 2: Database Engine and ORM Models
CREATE backend/db/engine.py:
  - SQLAlchemy async engine with aiosqlite
  - async_sessionmaker factory
  - init_db() to create all tables
  - get_session() async context manager

CREATE backend/models/database.py:
  - All ORM models from the Data Models section above

CREATE backend/models/schemas.py:
  - All Pydantic v2 schemas from the Data Models section above

Task 3: Utility Layer (Rate Limiter + Logging)
CREATE backend/utils/rate_limiter.py:
  - Token bucket implementation (asyncio-based)
  - Two buckets: per-second (20 tokens) and per-2-minutes (100 tokens)
  - async acquire() that awaits until a token is available
  - Thread-safe via asyncio.Lock

CREATE backend/utils/logging_config.py:
  - Structured logging with log level from config
  - Separate loggers for: poller, lcu, riot_api, api, db

Task 4: Live Client Data Poller (Core Feature)
CREATE backend/services/live_data_poller.py:
  - LiveDataPoller class with start()/stop() async methods
  - Polls https://127.0.0.1:2999/liveclientdata/allgamedata every POLLING_INTERVAL
  - CRITICAL: httpx.AsyncClient(verify=False) for self-signed SSL
  - Handle ConnectionRefusedError gracefully (game not running)
  - Parse response into LiveAllGameData Pydantic schema
  - Compare with previous poll to compute deltas (only changed fields per player)
  - Track last_event_id to only capture new events
  - Detect game start (first successful poll) and game end (connection lost)

Task 5: LCU Connector
CREATE backend/services/lcu_connector.py:
  - LCUConnector class using lcu-driver library
  - Auto-detect League client via lockfile parsing
  - Subscribe to WebSocket events:
    - /lol-gameflow/v1/session -> state transitions (Lobby, ChampSelect, InProgress, EndOfGame)
    - /lol-champ-select/v1/session -> champ select picks/bans
    - /lol-end-of-game/v1/eog-stats-block -> post-game stats
  - Expose current_state property
  - Handle client disconnect/reconnect gracefully

Task 6: Riot API Client
CREATE backend/services/riot_api_client.py:
  - RiotAPIClient class with httpx.AsyncClient
  - Uses rate_limiter.acquire() before every request
  - Methods:
    - get_account_by_riot_id(name, tagline) -> puuid       (Account-v1, REGIONAL route)
    - get_summoner_by_puuid(puuid) -> summoner_id           (Summoner-v4, PLATFORM route)
    - get_ranked_info(summoner_id) -> tier, division, LP    (League-v1, PLATFORM route)
    - get_champion_mastery(puuid, champion_id) -> level/pts (Champion-Mastery-v4, PLATFORM route)
    - get_match_ids(puuid, count) -> list of match IDs      (Match-v5, REGIONAL route)
    - get_match_detail(match_id) -> full match data         (Match-v5, REGIONAL route)
  - Retry on 429 with Retry-After header
  - Cache results in KnownPlayer table to reduce API calls
  - Handle 403 (expired API key) with clear error message

Task 7: Data Processor (Delta Compression + Storage)
CREATE backend/services/data_processor.py:
  - DataProcessor class
  - on_game_start(data): Create Game + PlayerInGame records for all 10 players
  - on_poll_data(game_id, data, deltas, new_events):
    - Store GameSnapshot with delta JSON
    - Store new GameEvents
    - Update PlayerInGame with latest known values
  - on_game_end(game_id):
    - Mark game as is_live=False, set final stats
    - Trigger Riot API enrichment as background task
  - enrich_players(game_id):
    - For each player: lookup PUUID -> ranked info -> champion mastery
    - Update PlayerInGame records

Task 8: Client Watcher (Lifecycle Orchestrator)
CREATE backend/services/client_watcher.py:
  - ClientWatcher class - main lifecycle orchestrator
  - State machine:
    DISCONNECTED -> CLIENT_OPEN -> CHAMP_SELECT -> IN_GAME -> POST_GAME -> CLIENT_OPEN
  - Coordinates LiveDataPoller, LCUConnector, RiotAPIClient, DataProcessor
  - Starts/stops poller based on game state
  - Exposes current state for API consumption
  - Handles lcu-driver's blocking start() via run_in_executor

Task 9: Database Repository
CREATE backend/db/repository.py:
  - GameRepository class with async methods:
    - list_games(limit, offset) -> List[GameSummary]
    - get_game(game_id) -> GameDetail with players and events
    - get_live_game() -> Game or None
    - save_game(), save_snapshot(), save_events()
    - update_player_enrichment()

Task 10: FastAPI Routes
CREATE backend/api/routes.py:
  - GET /api/games -> paginated games list
  - GET /api/games/live -> current live game (404 if none)
  - GET /api/games/{game_id} -> game detail with players and events
  - GET /api/status -> client state, DB stats
  - GET /health -> health check

CREATE backend/api/websocket.py:
  - WebSocket /ws/live -> streams LiveGameState every poll
  - Handles multiple connected clients
  - Sends game_start, game_update, game_end message types

Task 11: FastAPI Application Entry Point
CREATE backend/main.py:
  - FastAPI app with lifespan context manager
  - On startup: init_db(), start ClientWatcher as background task
  - On shutdown: stop ClientWatcher, close DB connections
  - CORS middleware (allow localhost origins for future frontend)
  - Include routes from api/routes.py
  - Entry: uvicorn backend.main:app --host 127.0.0.1 --port 8000

Task 12: Comprehensive Tests
CREATE tests/conftest.py:
  - Shared fixtures: in-memory SQLite test DB, sample JSON responses

CREATE tests/test_models.py:
  - ORM model creation, Pydantic schema validation, from_attributes conversion

CREATE tests/test_live_data_poller.py:
  - Successful poll parsing, connection refused handling, delta computation

CREATE tests/test_riot_api_client.py:
  - Each API method mocked, rate limiter integration, 429 retry behavior

CREATE tests/test_data_processor.py:
  - Delta compression correctness, game lifecycle, player enrichment

CREATE tests/test_repository.py:
  - CRUD operations against in-memory SQLite

CREATE tests/test_api.py:
  - All REST endpoints via TestClient, WebSocket, error responses

Task 13: Documentation Updates
UPDATE README.md:
  - Architecture overview, setup instructions, API endpoint docs

UPDATE AGENTS.md:
  - Fill in Project Context section
```

### Per task pseudocode as needed

```python
# ============================================================
# Task 3: Rate Limiter
# ============================================================
class TokenBucketRateLimiter:
    def __init__(self, rate: float, capacity: int):
        self._rate = rate          # tokens per second
        self._capacity = capacity  # max burst size
        self._tokens = capacity
        self._last_refill = time.monotonic()
        self._lock = asyncio.Lock()

    async def acquire(self):
        while True:
            async with self._lock:
                self._refill()
                if self._tokens >= 1:
                    self._tokens -= 1
                    return
            await asyncio.sleep(1.0 / self._rate)

    def _refill(self):
        now = time.monotonic()
        elapsed = now - self._last_refill
        self._tokens = min(self._capacity, self._tokens + elapsed * self._rate)
        self._last_refill = now

# Composite: must satisfy BOTH Riot rate limit windows
class RiotRateLimiter:
    def __init__(self):
        self._per_second = TokenBucketRateLimiter(rate=20, capacity=20)
        self._per_2min = TokenBucketRateLimiter(rate=100/120, capacity=100)

    async def acquire(self):
        await self._per_second.acquire()
        await self._per_2min.acquire()


# ============================================================
# Task 4: Live Data Poller
# ============================================================
class LiveDataPoller:
    BASE_URL = "https://127.0.0.1:2999"

    def __init__(self, polling_interval: float = 1.0):
        self._interval = polling_interval
        self._running = False
        self._previous_state: dict | None = None
        self._last_event_id: int = -1
        # Callbacks
        self.on_game_detected = None   # async callable(LiveAllGameData)
        self.on_poll_data = None       # async callable(LiveAllGameData, deltas, new_events)
        self.on_game_ended = None      # async callable()

    async def start(self):
        self._running = True
        # CRITICAL: verify=False for self-signed SSL certificate
        self._client = httpx.AsyncClient(verify=False, timeout=5.0)
        self._task = asyncio.create_task(self._poll_loop())

    async def _poll_loop(self):
        was_in_game = False
        while self._running:
            try:
                resp = await self._client.get(f"{self.BASE_URL}/liveclientdata/allgamedata")
                data = LiveAllGameData.model_validate(resp.json())

                if not was_in_game:
                    was_in_game = True
                    if self.on_game_detected:
                        await self.on_game_detected(data)

                deltas = self._compute_deltas(data)
                new_events = self._extract_new_events(data)
                if self.on_poll_data:
                    await self.on_poll_data(data, deltas, new_events)
                self._previous_state = data.model_dump()

            except (httpx.ConnectError, httpx.ConnectTimeout):
                if was_in_game:
                    was_in_game = False
                    self._previous_state = None
                    self._last_event_id = -1
                    if self.on_game_ended:
                        await self.on_game_ended()
            except Exception as e:
                logger.error(f"Poll error: {e}")

            await asyncio.sleep(self._interval)

    def _compute_deltas(self, current: LiveAllGameData) -> dict:
        """Return only changed fields per player vs previous poll."""
        if self._previous_state is None:
            return {p.summonerName: p.model_dump() for p in current.allPlayers}

        deltas = {}
        prev_players = {p["summonerName"]: p for p in self._previous_state.get("allPlayers", [])}
        for player in current.allPlayers:
            prev = prev_players.get(player.summonerName, {})
            curr = player.model_dump()
            player_delta = {k: v for k, v in curr.items()
                           if k not in ("summonerName", "championName", "team")
                           and prev.get(k) != v}
            if player_delta:
                deltas[player.summonerName] = player_delta
        return deltas

    def _extract_new_events(self, data: LiveAllGameData) -> list:
        all_events = data.events.get("Events", [])
        new = [e for e in all_events if e.get("EventID", 0) > self._last_event_id]
        if new:
            self._last_event_id = max(e.get("EventID", 0) for e in new)
        return new


# ============================================================
# Task 5: LCU Connector
# ============================================================
from lcu_driver import Connector

class LCUConnector:
    def __init__(self):
        self._connector = Connector()
        self._state = "disconnected"
        self.on_state_change = None
        self.on_champ_select = None
        self.on_post_game = None

        @self._connector.ready
        async def on_connect(connection):
            self._state = "client_open"
            await connection.ws.subscribe(
                "/lol-gameflow/v1/session",
                event_handler=self._on_gameflow_event
            )

        @self._connector.close
        async def on_disconnect(connection):
            self._state = "disconnected"

    async def _on_gameflow_event(self, connection, event):
        phase = event.data.get("phase", "")
        # Phases: "None", "Lobby", "ChampSelect", "InProgress", "EndOfGame"
        if phase == "ChampSelect":
            self._state = "champ_select"
            resp = await connection.request("get", "/lol-champ-select/v1/session")
            if resp.status == 200 and self.on_champ_select:
                await self.on_champ_select(await resp.json())
        elif phase == "InProgress":
            self._state = "in_game"
        elif phase == "EndOfGame":
            self._state = "post_game"
            resp = await connection.request("get", "/lol-end-of-game/v1/eog-stats-block")
            if resp.status == 200 and self.on_post_game:
                await self.on_post_game(await resp.json())

    async def start(self):
        # lcu-driver.start() is blocking - run in executor
        self._connector.start()


# ============================================================
# Task 6: Riot API Client
# ============================================================
class RiotAPIClient:
    def __init__(self, api_key: str, platform: str = "euw1", region: str = "europe"):
        self._api_key = api_key
        self._platform = platform
        self._region = region
        self._limiter = RiotRateLimiter()
        self._client = httpx.AsyncClient(
            headers={"X-Riot-Token": api_key},
            timeout=10.0
        )

    def _platform_url(self, path: str) -> str:
        return f"https://{self._platform}.api.riotgames.com{path}"

    def _regional_url(self, path: str) -> str:
        return f"https://{self._region}.api.riotgames.com{path}"

    async def _request(self, url: str) -> dict:
        await self._limiter.acquire()
        response = await self._client.get(url)
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "5"))
            await asyncio.sleep(retry_after)
            return await self._request(url)
        if response.status_code == 403:
            raise RuntimeError("API key expired or invalid")
        response.raise_for_status()
        return response.json()

    async def get_account_by_riot_id(self, game_name: str, tag_line: str) -> dict:
        # Account-v1 uses REGIONAL route
        return await self._request(
            self._regional_url(f"/riot/account/v1/accounts/by-riot-id/{game_name}/{tag_line}")
        )

    async def get_summoner_by_puuid(self, puuid: str) -> dict:
        # Summoner-v4 uses PLATFORM route
        return await self._request(
            self._platform_url(f"/lol/summoner/v4/summoners/by-puuid/{puuid}")
        )

    async def get_ranked_info(self, summoner_id: str) -> list:
        return await self._request(
            self._platform_url(f"/lol/league/v4/entries/by-summoner/{summoner_id}")
        )

    async def get_match_detail(self, match_id: str) -> dict:
        # Match-v5 uses REGIONAL route
        return await self._request(
            self._regional_url(f"/lol/match/v5/matches/{match_id}")
        )


# ============================================================
# Task 8: Client Watcher (state machine orchestrator)
# ============================================================
class ClientWatcher:
    def __init__(self, config, db_session_factory):
        self._poller = LiveDataPoller(config.POLLING_INTERVAL_SECONDS)
        self._lcu = LCUConnector()
        self._riot_api = RiotAPIClient(config.RIOT_API_KEY, config.RIOT_PLATFORM, config.RIOT_REGION)
        self._processor = DataProcessor(db_session_factory, self._riot_api)
        self._state = "disconnected"
        self._current_game_id = None

        # Wire callbacks
        self._poller.on_game_detected = self._on_game_detected
        self._poller.on_poll_data = self._on_poll_data
        self._poller.on_game_ended = self._on_game_ended

    async def start(self):
        asyncio.create_task(self._run_lcu())
        await self._poller.start()

    async def _run_lcu(self):
        loop = asyncio.get_event_loop()
        await loop.run_in_executor(None, self._lcu._connector.start)

    async def _on_game_detected(self, data):
        self._state = "in_game"
        self._current_game_id = await self._processor.on_game_start(data)

    async def _on_poll_data(self, data, deltas, new_events):
        if self._current_game_id:
            await self._processor.on_poll_data(self._current_game_id, data, deltas, new_events)

    async def _on_game_ended(self):
        if self._current_game_id:
            await self._processor.on_game_end(self._current_game_id)
            self._current_game_id = None
        self._state = "post_game"

    @property
    def current_state(self) -> ClientState:
        return ClientState(
            client_running=self._lcu._state != "disconnected",
            client_state=self._state,
            current_game_id=self._current_game_id,
        )


# ============================================================
# Task 10/11: FastAPI Routes + Entry Point
# ============================================================
from fastapi import FastAPI, APIRouter, Depends, HTTPException
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    await init_db(settings.DB_PATH)
    watcher = ClientWatcher(settings, get_session_factory())
    app.state.watcher = watcher
    asyncio.create_task(watcher.start())
    yield
    await watcher.stop()

app = FastAPI(title="MnM-App Backend", version="0.1.0", lifespan=lifespan)

router = APIRouter(prefix="/api")

@router.get("/games", response_model=list[GameSummary])
async def list_games(limit: int = 20, offset: int = 0):
    ...

@router.get("/games/live", response_model=LiveGameState)
async def get_live_game():
    ...  # 404 if no active game

@router.get("/games/{game_id}", response_model=GameDetail)
async def get_game(game_id: int):
    ...

@router.get("/status", response_model=ClientState)
async def get_status():
    ...

@router.get("/health")
async def health_check():
    return {"status": "ok"}

# Entry: uvicorn backend.main:app --host 127.0.0.1 --port 8000
```

### Integration Points
```yaml
DATABASE:
  - engine: SQLite via aiosqlite (file: ./data/mnm.db)
  - tables: games, players_in_games, game_snapshots, game_events, known_players
  - init: Create all tables on startup via Base.metadata.create_all()

CONFIG:
  - file: backend/config.py (pydantic_settings.BaseSettings)
  - required: RIOT_API_KEY (optional if you skip enrichment), RIOT_PLATFORM, RIOT_REGION
  - defaults: DB_PATH=./data/mnm.db, POLLING_INTERVAL=1.0, LOG_LEVEL=INFO

EXTERNAL SERVICES:
  - Live Client Data API: https://127.0.0.1:2999 (local, no auth, self-signed SSL)
  - LCU API: https://127.0.0.1:{port} (local, Basic auth from lockfile)
  - Riot API: https://{platform/region}.api.riotgames.com (remote, API key header)

DEPENDENCIES (pyproject.toml):
  runtime:
    - fastapi>=0.115.0
    - uvicorn[standard]>=0.32.0
    - httpx>=0.27.0
    - sqlalchemy[asyncio]>=2.0.0
    - aiosqlite>=0.20.0
    - pydantic>=2.9.0
    - pydantic-settings>=2.6.0
    - lcu-driver>=3.0.2
    - websockets>=13.0
  dev:
    - pytest>=8.3.0
    - pytest-asyncio>=0.24.0
    - pytest-httpx>=0.32.0
    - ruff>=0.8.0
    - mypy>=1.13.0

ENTRY POINT:
  uvicorn backend.main:app --host 127.0.0.1 --port 8000 --reload
```

## Validation Loop

### Level 1: Syntax & Style
```bash
ruff check backend/ tests/ --fix   # Auto-fix what's possible
ruff format backend/ tests/        # Format code
mypy backend/                      # Type checking

# Expected: No errors. If errors, READ the error and fix.
# Common mypy issue: missing type stubs for lcu-driver (use # type: ignore)
```

### Level 2: Unit Tests
```python
# tests/test_live_data_poller.py
@pytest.mark.asyncio
async def test_poll_parses_valid_response(sample_allgamedata_json):
    """Successful poll returns parsed LiveAllGameData."""
    # Mock httpx response with sample JSON
    # Verify parsing works without errors

@pytest.mark.asyncio
async def test_poll_handles_connection_refused():
    """Connection refused (no game) is handled gracefully."""
    # Should not raise, should call on_game_ended if was_in_game

@pytest.mark.asyncio
async def test_delta_computation_only_changed_fields():
    """Delta contains only fields that changed between polls."""
    # Set previous state, compute delta, assert only changed keys present

# tests/test_riot_api_client.py
@pytest.mark.asyncio
async def test_get_account_by_riot_id(mock_httpx):
    """Account-v1 returns PUUID for valid Riot ID."""
    # Mock response, verify correct URL (regional route) and parsing

@pytest.mark.asyncio
async def test_rate_limit_retry_on_429(mock_httpx):
    """429 response triggers retry after Retry-After seconds."""
    # First response: 429, second: 200. Assert client retries.

# tests/test_data_processor.py
@pytest.mark.asyncio
async def test_delta_storage_smaller_than_full():
    """Delta storage is significantly smaller than full snapshots."""
    # Feed 100 similar polls, assert total stored < 100 * full_size

# tests/test_api.py
def test_health_check():
    client = TestClient(app)
    response = client.get("/health")
    assert response.status_code == 200

def test_live_game_404_when_no_game():
    client = TestClient(app)
    response = client.get("/api/games/live")
    assert response.status_code == 404
```

```bash
# Run and iterate until passing:
pytest tests/ -v --tb=short

# With coverage:
pytest tests/ -v --cov=backend --cov-report=term-missing
# Target: 70%+ coverage for Phase 1
```

### Level 3: Integration Test
```bash
# 1. Start the backend (League client NOT required for basic tests)
uvicorn backend.main:app --host 127.0.0.1 --port 8000

# 2. Health check
curl http://127.0.0.1:8000/health
# Expected: {"status": "ok"}

# 3. Status (no client running)
curl http://127.0.0.1:8000/api/status
# Expected: {"client_running": false, "client_state": "disconnected", ...}

# 4. Empty games list
curl http://127.0.0.1:8000/api/games
# Expected: []

# 5. No live game
curl http://127.0.0.1:8000/api/games/live
# Expected: 404 {"detail": "No active game"}

# 6. (With League client open and in a game)
# Verify /api/status shows in_game
# Verify /api/games/live returns real-time data
# Verify WebSocket at ws://127.0.0.1:8000/ws/live streams updates

# 7. (After game ends)
# Verify game appears in /api/games
# Verify /api/games/{id} shows all 10 players with final stats
```

## Final Validation Checklist
- [ ] All tests pass: `pytest tests/ -v`
- [ ] No linting errors: `ruff check backend/ tests/`
- [ ] No type errors: `mypy backend/`
- [ ] Health endpoint returns 200
- [ ] Status endpoint shows "disconnected" without League client
- [ ] Games list returns empty array when no games played
- [ ] Live game returns 404 when no active game
- [ ] .env.example contains all variables with descriptions
- [ ] README.md has setup and usage instructions
- [ ] Error cases handled gracefully (no unhandled exceptions)
- [ ] Logs show state transitions and poll counts
- [ ] SQLite database created automatically on first run
- [ ] Process uses <100 MB RAM

---

## Anti-Patterns to Avoid
- Do not hardcode API keys, paths, or platform/region - use config.py and .env
- Do not use sync functions in async FastAPI endpoints or background tasks
- Do not use SQLAlchemy 1.x patterns (Column(), declarative_base()) - use 2.0 style
- Do not use Pydantic v1 patterns (class Config, @validator) - use v2 style
- Do not poll Live Client Data API when no game is running (wastes CPU, fills logs)
- Do not store full snapshots every second - use delta compression
- Do not ignore Riot API rate limits - always go through the rate limiter
- Do not assume the LCU port is fixed - always parse from lockfile
- Do not use verify=True on Live Client Data API (self-signed cert will fail)
- Do not assume creepScore is exact (increments of 10)
- Do not use threading for async work - use asyncio.create_task()
- Do not commit .env or *.db files to git

## Confidence Score: 8/10

High confidence due to:
- Well-documented APIs with official schemas and community resources
- Mature Python library ecosystem (FastAPI, SQLAlchemy 2.0, httpx, pydantic v2)
- Local-first architecture minimizes network complexity
- Phased approach (backend first) reduces scope

Uncertainty:
- lcu-driver library may have compatibility issues with modern Python or event loop conflicts
- LCU API is unofficial and may change without notice
- Live Client Data API's self-signed SSL needs testing across httpx versions
- Delta compression logic needs careful testing for data integrity
