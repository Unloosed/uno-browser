# Functional Design Specification (FDS)

## Web-Based UNO Game for itch.io

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Development Team  
**Status:** Final Specification

---

## 1. EXECUTIVE SUMMARY

This document specifies the design and implementation of **UNO Web**, a real-time multiplayer card game hosted on itch.io. The game uses:

- **Frontend:** HTML5 + JavaScript/TypeScript running in the browser
- **Backend:** Python (FastAPI) WebSocket server for game logic and state management
- **Hosting:** Static HTML5 client on itch.io, Python backend on external VPS/service (Render, Railway, etc.)

The game supports 2–4 players, local browser sessions (no account system), and deterministic game state synchronized via WebSocket.

### Key Features

- Real-time multiplayer gameplay (2–4 players)
- Turn-based UNO rules with all standard card actions
- Responsive UI for desktop and tablet
- Room-based matchmaking (players create/join rooms via URL)
- Spectator mode (read-only observation)
- Chat system (optional for V2)

---

## 2. TECHNICAL ARCHITECTURE

### 2.1 High-Level System Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    PLAYER BROWSERS                              │
│  (itch.io served HTML5 + JS client, runs locally on device)     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    WebSocket over HTTPS
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│          PYTHON BACKEND (FastAPI + WebSocket)                   │
│  Deployed on: Render, Railway, DigitalOcean, or similar         │
│  Responsibilities:                                              │
│  • Game state management (players, hands, deck, turn order)     │
│  • Rule enforcement (valid move validation)                     │
│  • Room management (create, join, leave, cleanup)               │
│  • Real-time state broadcast to all clients                     │
│  • Timeout handling (disconnect, turn timeout)                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Frontend Stack

| Component | Technology | Notes |
|-----------|------------|-------|
| **Language** | TypeScript / JavaScript | Single-file app or modular |
| **HTML5 Canvas** | Canvas 2D API | For rendering card UI |
| **WebSocket Client** | `WebSocket` API (native) | Real-time communication |
| **Styling** | CSS3 (no framework for itch.io) | Responsive, no build step required |
| **Build Output** | Single `index.html` | Bundles CSS + JS inline or external |

**Deployment:** Uploaded as ZIP to itch.io under "HTML game" classification.

### 2.3 Backend Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| **Framework** | FastAPI (Python 3.10+) | Async, built-in WebSocket support |
| **WebSocket Protocol** | python-socketio or fastapi-ws | Broadcast-capable, room management |
| **Game Logic** | Custom Python module | `game_engine.py` handles rules |
| **Concurrency** | asyncio | Handles multiple rooms/players |
| **Persistence** | Optional SQLite | Store game history, leaderboards (V2) |
| **Environment** | `.env` file for secrets | API keys, allowed origins, etc. |

**Deployment:** Container (Docker) or direct Python on:

- **Render** (free tier: ~7.5 hrs/month)
- **Railway** (pay-as-you-go, ~5 USD/month for idle)
- **DigitalOcean App Platform** (~5 USD/month)

---

## 3. GAME MECHANICS & RULES

### 3.1 Card Deck Composition

Standard UNO deck (108 cards):

- **Number Cards (0–9):** 4 colors × 9 denominations + 1 zero = 76 cards
  - 0: 1 per color
  - 1–9: 2 per color
- **Action Cards:** 8 per color (32 total)
  - Skip (2 per color)
  - Reverse (2 per color)
  - Draw 2 (2 per color)
- **Wild Cards (4 total)**
  - Wild (color change)
  - Wild Draw 4
- **Total:** 108 cards

### 3.2 Game Flow

#### Setup Phase

1. **Dealer Role:** First player (room creator or random).
2. **Deal:** Each player receives 7 cards.
3. **Discard Pile:** Top card of deck starts the discard pile.
   - If Action or Wild → flip next non-action card (or apply action to first player if Reverse/Skip initially).
4. **Turn Order:** Clockwise from dealer.

#### Turn Sequence (Per Player)

1. **Check Hand:** Player views their cards.
2. **Play or Draw:**
   - **Valid Play:** Play a card matching color, number, or symbol to discard pile.
   - **No Valid Play:** Draw 1 card from deck.
     - If drawn card matches, may play immediately.
     - If not, turn ends.
3. **Action Resolution:** If card is action/wild, resolve immediately.
4. **UNO Call:** If hand size = 1, must announce "UNO" before next player's turn begins.
5. **Turn Ends:** Next player in turn order plays.

#### Card Actions

| Card | Effect |
|------|--------|
| **Skip** | Next player skips their turn. |
| **Reverse** | Reverse turn order direction. If 2 players, acts as Skip. |
| **Draw 2** | Next player draws 2 cards; skip their turn. |
| **Wild** | Current player chooses new active color. No effect on play beyond color change. |
| **Wild Draw 4** | Current player chooses color; next player draws 4 and skips turn. **Challengeable:** Next player may challenge if current player had matching color. |
| **Challenge Rule:** If challenge succeeds (challenger was right), challenger draws 6; if fails, challenger draws 4. |

### 3.3 Winning Conditions

**Round Win:** First player to have 0 cards.
**Game Win:** Cumulative scoring to 500 points.

#### Scoring (Per Round)

| Card Type | Points |
|-----------|--------|
| Number (0–9) | Face value |
| Skip, Reverse, Draw 2 | 20 |
| Wild | 50 |
| Wild Draw 4 | 50 |

Winner tallies all opponent card values; first to 500 wins overall game.

### 3.4 Special Rules Implemented

- **UNO Call:** Mandatory when hand = 1. Penalty if caught without calling: +2 cards (enforced by other players via "UNO Challenge" button or automatic if next turn starts).
- **Draw Pile Depletion:** If draw pile empty and player must draw, discard pile shuffles back into draw pile (except top card).
- **Jump-In (Optional, V1 Off):** Players may play exact match (color + number) out of turn. Can be toggled in room settings.
- **Seven-O Rule (Optional, V1 Off):** Play of 7 → swap hands with another player; play of 0 → all pass hands to next player. Can be toggled in room settings.

---

## 4. SYSTEM DESIGN

### 4.1 Frontend Architecture

#### Module Structure

```
index.html
├── <head>
│   ├── Meta tags (viewport, charset)
│   ├── <style> (all CSS inline)
│   └── Favicon / manifest (minimal)
├── <body>
│   ├── <canvas id="gameCanvas">
│   ├── <div id="ui-container"> (buttons, turn indicator, chat)
│   └── <script type="module"> (all JS inline or src link to external bundle)
└── (optional) assets folder (card sprites, sounds—hosted separately or inlined as data URIs)
```

#### UI Components

1. **Main Menu Screen**
   - Title "UNO Web"
   - Input field: Player name (default: "Player X")
   - Button: "Create Game" → generates room code
   - Button: "Join Game" → input room code field
   - Button: "Spectate" → input room code

2. **Waiting Room Screen**
   - Display: Room code, Players list, Ready status
   - Button: "Start Game" (only for room creator; requires 2–4 players)
   - Button: "Leave Room"
   - Chat display (optional V1)

3. **Game Screen**
   - **Left Panel:** Opponent hands (card count per player, position indicator)
   - **Center:** Discard pile (top card), Draw pile
   - **Bottom:** Current player's hand (fan of draggable cards)
   - **Top:** Turn indicator, Current color, Active player highlight
   - **Right Panel:** Chat log, Spectators count, Settings button
   - **Action Buttons:**
     - "Play Card" (after selecting from hand)
     - "Draw Card" (if no playable card)
     - "Challenge" (if Draw 4 played)
     - "UNO" (when hand = 1; auto-highlight when needed)
     - "Forfeit" (with confirmation)

4. **End Game Screen**
   - Winner announcement
   - Score summary (this round + cumulative)
   - Button: "Play Again" or "Return to Menu"

#### State Management (Client-Side)

```typescript
interface ClientGameState {
  roomCode: string;
  playerId: string;
  playerName: string;
  gamePhase: 'menu' | 'waiting' | 'playing' | 'gameOver';
  
  myHand: Card[];
  otherPlayers: { id: string; name: string; cardCount: number; position: number }[];
  currentPlayer: string;
  turnOrder: string[]; // Array of player IDs in play order
  discardPile: Card[];
  deckSize: number;
  activeColor: Color; // Current color in play
  
  isMyTurn: boolean;
  validMoves: Card[]; // Which cards in my hand are playable now
  gameLog: LogEntry[];
  
  roundScore: { [playerId: string]: number };
  cumulativeScore: { [playerId: string]: number };
}
```

### 4.2 Backend Architecture

#### Core Modules

**`main.py` (FastAPI App)**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi_ws import WebSocketManager

app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=[...])

# Room manager, game logic router
room_manager = RoomManager()
socket_manager = WebSocketManager()

@app.websocket("/ws/{room_code}/{player_id}")
async def websocket_endpoint(websocket, room_code, player_id):
    # Handle connections, route events to room handler
```

**`game_engine.py` (Game Rules & State)**

```python
class Card:
    def __init__(self, color: str, value: str):
        self.color = color
        self.value = value

class Deck:
    def __init__(self):
        self.cards = [...]  # Initialize standard 108-card deck
    def shuffle(self):
        random.shuffle(self.cards)
    def draw(self, n: int) -> List[Card]:
        # Draw n cards, reshuffle discard if needed
    def reset_from_discard(self):
        # Move discard back to draw pile

class GameRoom:
    def __init__(self, room_code: str, max_players: int = 4):
        self.room_code = room_code
        self.players: List[Player] = []
        self.deck = Deck()
        self.discard_pile = []
        self.current_player_idx = 0
        self.turn_order = ['p1', 'p2', ...]
        self.direction = 1  # 1 = clockwise, -1 = reverse
        self.active_color = None
        self.game_state = 'waiting'  # 'waiting', 'playing', 'over'
        
    def add_player(self, player_id: str, name: str) -> bool:
        # Add player to room, return success
    
    def remove_player(self, player_id: str):
        # Remove player; if in-game, handle forfeit
    
    def start_game(self):
        # Deal 7 cards each, set first player, broadcast state
    
    def is_valid_move(self, player_id: str, card: Card) -> bool:
        # Check if card matches discard pile top (color/number/action)
    
    def play_card(self, player_id: str, card: Card, chosen_color: str = None) -> GameAction:
        # Execute card play, resolve action, advance turn
        # Return action object for broadcast
    
    def draw_card(self, player_id: str) -> Card:
        # Draw 1 card, return it, check if playable
    
    def advance_turn(self):
        # Move to next player in turn order, handle direction
    
    def check_uno_call(self, player_id: str) -> bool:
        # Verify if player called UNO when hand = 1
    
    def challenge_draw4(self, challenger_id: str, draw4_player_id: str) -> bool:
        # Check if draw4 player had matching color; apply penalty
    
    def get_game_state(self) -> dict:
        # Return serializable game state for broadcast
    
    def calculate_scores(self) -> dict:
        # Tally scores for round end
```

**`room_manager.py` (Room & Player Management)**

```python
class RoomManager:
    def __init__(self):
        self.rooms = {}  # { room_code: GameRoom }
    
    def create_room(self, room_code: str) -> GameRoom:
        # Generate room, store, return
    
    def join_room(self, room_code: str, player_id: str, name: str) -> GameRoom:
        # Find room, add player, return room
    
    def get_room(self, room_code: str) -> GameRoom:
        # Lookup room
    
    def delete_room(self, room_code: str):
        # Clean up empty room
    
    def generate_room_code(self) -> str:
        # Generate 4–6 alphanumeric code (e.g., "ABC123")
```

**`socket_handler.py` (WebSocket Event Routing)**

```python
class SocketHandler:
    def __init__(self, socket_manager, room_manager):
        self.socket_manager = socket_manager
        self.room_manager = room_manager
    
    async def on_join(self, room_code: str, player_id: str, name: str):
        # Add to room, emit 'playerJoined' to all in room
    
    async def on_play_card(self, room_code: str, player_id: str, card: Card, color: str):
        # Validate move, execute, broadcast state
    
    async def on_draw_card(self, room_code: str, player_id: str):
        # Draw, check playable, broadcast
    
    async def on_uno_call(self, room_code: str, player_id: str):
        # Record UNO call timestamp
    
    async def on_challenge(self, room_code: str, challenger_id: str, target_id: str):
        # Validate challenge, resolve, broadcast
    
    async def on_chat(self, room_code: str, player_id: str, message: str):
        # Broadcast message to all (no profanity filter V1)
    
    async def on_disconnect(self, room_code: str, player_id: str):
        # Remove player, mark as disconnected, handle timeout
    
    async def broadcast_game_state(self, room_code: str, state: dict):
        # Emit 'gameStateUpdate' to all players in room
```

#### Data Flow Example: Player Plays a Card

```
Client                            Server                          Broadcast
  │                                │                                  │
  ├─ emit('play_card', {...}) ──→  │                                  │
  │                                ├─ Validate card in hand           │
  │                                ├─ Validate matches top of discard │
  │                                ├─ Remove from player's hand       │
  │                                ├─ Add to discard pile             │
  │                                ├─ Resolve action (if any)         │
  │                                ├─ Advance turn                    │
  │                                ├─ Build game state                │
  │                                ├─ emit 'gameStateUpdate' ────────→ All clients
  │                                │
  ← ────────── All clients receive updated state ───────────────────
  │
  └─ Render new hand, discard, turn indicator
```

#### WebSocket Events (Client → Server → Broadcast)

| Event | Payload | Response |
|-------|---------|----------|
| `join` | `{room_code, player_id, name}` | `gameStateUpdate` to all in room |
| `play_card` | `{room_code, player_id, card, chosen_color?}` | `gameStateUpdate` broadcast |
| `draw_card` | `{room_code, player_id}` | `gameStateUpdate` broadcast |
| `uno_call` | `{room_code, player_id}` | `unoStatus` update |
| `challenge` | `{room_code, challenger_id, target_id}` | `challengeResult`, then `gameStateUpdate` |
| `chat` | `{room_code, player_id, message}` | `chatMessage` to all |
| `leave` | `{room_code, player_id}` | `playerLeft` to all |
| `start_game` | `{room_code}` | `gameStateUpdate` with dealt hands |

---

## 5. IMPLEMENTATION DETAILS

### 5.1 Card Play Validation Algorithm

```python
def is_valid_move(top_card: Card, play_card: Card, active_color: str) -> bool:
    """
    Determine if play_card is valid given top card of discard pile.
    - Color match (color == top_card.color), OR
    - Number/symbol match (value == top_card.value), OR
    - Wild card (can always play)
    """
    if play_card.color == "Wild":
        return True  # Wild cards always playable
    
    if play_card.color == active_color:
        return True  # Color match (or matches declared color after wild)
    
    if play_card.value == top_card.value:
        return True  # Number/symbol match
    
    return False
```

### 5.2 Turn Advancement with Action Resolution

```python
def advance_turn_with_action(game_room: GameRoom, card_played: Card, chosen_color: str = None):
    """
    Resolve card action and move to next valid player.
    Handles Skip, Reverse, Draw 2, Draw 4.
    """
    current_idx = game_room.current_player_idx
    next_idx = current_idx + game_room.direction
    next_idx %= len(game_room.turn_order)
    
    if card_played.value == "Skip":
        next_idx = (next_idx + game_room.direction) % len(game_room.turn_order)
    elif card_played.value == "Reverse":
        if len(game_room.turn_order) > 2:
            game_room.direction *= -1
            next_idx = (current_idx + game_room.direction) % len(game_room.turn_order)
        else:
            # 2-player: Reverse acts as Skip
            next_idx = (next_idx + game_room.direction) % len(game_room.turn_order)
    elif card_played.value == "Draw 2":
        game_room.players[next_idx].hand += game_room.deck.draw(2)
        next_idx = (next_idx + game_room.direction) % len(game_room.turn_order)
    elif card_played.value == "Wild Draw 4":
        if chosen_color:
            game_room.active_color = chosen_color
        game_room.players[next_idx].hand += game_room.deck.draw(4)
        next_idx = (next_idx + game_room.direction) % len(game_room.turn_order)
    
    game_room.current_player_idx = next_idx
```

### 5.3 UNO Call Detection

```python
def check_uno_violation(game_room: GameRoom, player_id: str) -> bool:
    """
    Called before advancing to next player.
    If player has 1 card and has NOT called UNO, return True (violation).
    """
    player = game_room.get_player(player_id)
    if len(player.hand) == 1:
        # Check if UNO was called within the last X seconds before next turn
        if not player.uno_called:
            return True  # Violation: penalty applies
        player.uno_called = False  # Reset for next turn
    return False

def penalize_uno_violation(game_room: GameRoom, player_id: str):
    """Apply +2 card penalty for uncalled UNO."""
    player = game_room.get_player(player_id)
    player.hand += game_room.deck.draw(2)
```

### 5.4 Game State Serialization (Sent to Clients)

```python
def serialize_game_state(game_room: GameRoom, observer_player_id: str = None) -> dict:
    """
    Return game state visible to observer_player_id.
    If observer is a player: include their hand.
    If observer is spectator: show only card counts.
    """
    state = {
        'room_code': game_room.room_code,
        'game_phase': game_room.game_state,
        'players': [
            {
                'id': p.id,
                'name': p.name,
                'position': idx,
                'cardCount': len(p.hand),
                'hand': p.hand if p.id == observer_player_id else None,
                'roundScore': game_room.round_scores.get(p.id, 0),
                'cumulativeScore': game_room.cumulative_scores.get(p.id, 0),
            }
            for idx, p in enumerate(game_room.players)
        ],
        'currentPlayerIdx': game_room.current_player_idx,
        'currentPlayerId': game_room.turn_order[game_room.current_player_idx],
        'discardPile': [
            {'color': c.color, 'value': c.value} for c in game_room.discard_pile[-3:]
        ],  # Last 3 for visual
        'topCard': {
            'color': game_room.discard_pile[-1].color,
            'value': game_room.discard_pile[-1].value,
        },
        'activeColor': game_room.active_color,
        'deckSize': len(game_room.deck.cards),
        'direction': 'clockwise' if game_room.direction == 1 else 'counter-clockwise',
        'timestamp': time.time(),
    }
    return state
```

---

## 6. DEPLOYMENT & INFRASTRUCTURE

### 6.1 Frontend Deployment (itch.io)

**Steps:**

1. Build HTML5 artifact: `index.html` + inline CSS + inline JS (or bundle.js served from CDN).
2. Create ZIP: `uno-web.zip`

   ```
   uno-web/
   ├── index.html
   ├── assets/
   │   ├── cards.png (sprite sheet)
   │   └── sounds/ (optional)
   └── (bundle.js if separate)
   ```

3. Login to itch.io dashboard → My projects → Create new project.
4. Upload ZIP, configure:
   - Kind: HTML
   - Embed size: 1280 × 720 (16:9 viewport)
   - Frame title: "UNO Web"
5. Mark as public, set backend URL in client (via `.env` or hardcoded config).

**Environment Setup (Client):**

```typescript
const BACKEND_URL = import.meta.env.VITE_BACKEND_URL || "https://uno-backend.example.com";
const WEBSOCKET_URL = BACKEND_URL.replace(/^http/, 'ws');
```

### 6.2 Backend Deployment (VPS)

**Option A: Render (Recommended for simplicity)**

1. Create `render.yaml` in repo:

   ```yaml
   services:
     - type: web
       name: uno-backend
       runtime: python
       buildCommand: pip install -r requirements.txt
       startCommand: uvicorn main:app --host 0.0.0.0 --port 10000
       envVars:
         - key: PYTHONUNBUFFERED
           value: true
   ```

2. Push to GitHub, connect Render.
3. Deploy. Free tier: app spins down after 15 min inactivity (~7.5 hrs/month active time).

**Option B: Railway or DigitalOcean**
Similar Docker/Dockerfile setup; ~5–10 USD/month for always-on.

**Backend Structure:**

```
uno-backend/
├── main.py
├── game_engine.py
├── room_manager.py
├── socket_handler.py
├── models.py (Pydantic schemas)
├── requirements.txt
├── .env.example
├── Dockerfile (optional)
└── render.yaml
```

**`requirements.txt`:**

```
fastapi==0.104.1
uvicorn==0.24.0
python-socketio==5.9.0
python-multipart==0.0.6
python-dotenv==1.0.0
pydantic==2.4.2
```

### 6.3 CORS & WebSocket Configuration

```python
# main.py
from fastapi.middleware.cors import CORSMiddleware

allowed_origins = [
    "https://yourname.itch.io",  # itch.io subdomain where game is hosted
    "http://localhost:3000",      # Local dev
    "http://localhost:5173",      # Vite dev
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## 7. USER FLOWS

### 7.1 Start a New Game

```
1. Player opens itch.io page → loads index.html
2. Frontend UI: Main menu
3. Player enters name, clicks "Create Game"
4. Frontend: WebSocket connect to backend (/ws/{room_code}/{player_id})
5. Backend: Room created, broadcast state to player
6. Frontend: Show waiting room with room code
7. Other players visit same page, click "Join", enter code
8. Frontend: Connect to same room
9. Backend: Add players, broadcast updated room state
10. Room creator clicks "Start Game"
11. Backend: Deal 7 cards each, set first player, broadcast 'playing' state
12. All clients: Render game board, current player can play
```

### 7.2 Play a Card

```
1. Player (whose turn) clicks card in hand
2. Frontend: Highlight card, show "Play" button
3. Player clicks "Play"
4. Frontend: Validate card is in hand, emit 'play_card' event to backend
5. Backend: Validate move against top card of discard pile
   - If invalid: emit 'invalidMove' error back to client
   - If valid: proceed
6. Backend: Execute card action (if any), advance turn, update state
7. Backend: Broadcast 'gameStateUpdate' to all players
8. All clients: Render new discard, turn indicator, updated hands
9. Next player's turn begins
```

### 7.3 Challenge a Draw 4

```
1. Player A plays Wild Draw 4, selects color
2. Backend: Next player (B) is flagged as "can challenge"
3. Frontend (B): Show "Challenge" button next to "Draw 4 Cards" button
4. Player B clicks "Challenge"
5. Frontend: Emit 'challenge' event
6. Backend: Check if Player A's hand had matching color
   - Yes: Player A draws 4, play continues
   - No: Player B draws 6, play continues
7. Backend: Broadcast challenge result and updated state
8. All clients: Show challenge outcome
```

### 7.4 Win Condition

```
1. Player plays last card
2. Backend: Detects hand size = 0
3. Backend: Tally round scores, broadcast 'roundWon'
4. Frontend: Show winner and scores
5. User clicks "Play Again"
6. Backend: Reset deck, redeal, start new round (same players, cumulative scores)
   OR "Menu" → return to room selection
```

---

## 8. TESTING & VALIDATION

### 8.1 Unit Tests (Backend)

**Test Cases:**

- Card play validation (valid color, valid number, wild card)
- Turn advancement with each action (Skip, Reverse, Draw 2, Draw 4)
- UNO call detection and penalty
- Draw 4 challenge (valid and invalid)
- Hand depletion and deck reshuffle
- Disconnect handling (player left mid-game)

**Tools:** pytest + asyncio fixtures

### 8.2 Integration Tests

- WebSocket connection and room creation
- Multi-player state sync
- End-to-end game (create, play 3 turns, end round, score tally)

### 8.3 Manual Testing (QA)

- Desktop browsers: Chrome, Firefox, Safari (latest)
- Mobile browsers: Chrome, Safari (iPad, Android tablet)
- Network conditions: Normal, slow (2G), offline (handling)
- Edge cases:
  - 2, 3, 4 players
  - Player disconnect mid-turn
  - Rapid card plays
  - Challenge during draw 4

---

## 9. PERFORMANCE & SCALABILITY

### 9.1 Client Performance

- **Card Rendering:** Canvas 2D (no GPU acceleration needed)
- **Input Latency:** <50 ms (drag-and-drop is local, validated server-side)
- **Network:** 1–2 KB per state update (JSON), 10–50 ms round-trip

### 9.2 Server Performance

- **Concurrent Games:** Single Render free dyno can handle ~50–100 concurrent games (async I/O bottleneck is mostly I/O wait, not CPU).
- **Message Rate:** ~10–20 messages/second per game, bursty.
- **State Size:** ~5 KB per room.
- **Scaling:** If needed, move to Railway or DigitalOcean; use Redis for room state caching (V2).

### 9.3 Optimization

- **Frontend:** Minimize repaints; only redraw changed cards.
- **Backend:** Use connection pooling for DB (if added); room cleanup after 1 hour idle.

---

## 10. SECURITY CONSIDERATIONS

### 10.1 Client-Side

- **Input Validation:** Sanitize player names (max 32 chars, alphanumeric + spaces).
- **No PII:** No account system, no password storage.

### 10.2 Server-Side

- **Move Validation:** All moves validated server-side (client is untrusted).
- **Room Codes:** 6-char alphanumeric, 36^6 ≈ 2 billion possibilities (brute-force resistant).
- **Rate Limiting:** Limit events/second per player (prevent spam).
- **Disconnect Cleanup:** Remove rooms after all players disconnect.

### 10.3 Communication

- **HTTPS/WSS:** Enforce encrypted connections (itch.io is HTTPS; backend must use WSS).
- **CORS:** Restrict to itch.io domain.

---

## 11. FUTURE ENHANCEMENTS (V2+)

- **Persistent Leaderboards:** Store wins/ELO in SQLite.
- **House Rules Toggle:** Jump-In, Seven-O, Custom Deck.
- **Voice Chat:** WebRTC integration.
- **AI Opponent:** Single-player vs bot option.
- **Card Animations:** CSS/canvas animations for card plays.
- **Sound Effects:** Card play, UNO call, win sounds.
- **Mobile App:** React Native or Flutter wrapper (with adjusted UI).
- **Spectator Chat:** Only for observers.
- **Replay System:** Record and share game replays.

---

## 12. GLOSSARY

| Term | Definition |
|------|-----------|
| **Room Code** | 6-char alphanumeric identifier for a game session. |
| **Discard Pile** | Stack of played cards; top card is the current "active" card. |
| **Draw Pile** | Undealt cards; refilled from discard pile if depleted. |
| **Active Color** | Current color in play, set by Wild card. |
| **Action Card** | Skip, Reverse, Draw 2. |
| **Wild Card** | Card that can be played on any color; allows color selection. |
| **UNO** | Verbal declaration when player has 1 card left. |
| **Challenge** | Dispute validity of a Wild Draw 4 play. |
| **Round** | Single game from deal to one player's hand = 0. |
| **Game** | Series of rounds until one player reaches 500 points. |

---

## 13. DOCUMENT HISTORY

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Feb 2026 | Initial FDS; core mechanics, architecture, deployment. |

---

**End of Document**
