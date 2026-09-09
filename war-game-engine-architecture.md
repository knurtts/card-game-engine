# Reusable Card Game Engine Architecture

## Goal

Build a reusable card game engine that supports *War* as the first implementation, while allowing future expansion into more complex card games.

### Design Philosophy

Do not build a "War game."

Build a generic card game engine and implement War as the first ruleset.

---

# High-Level Architecture

```text
┌─────────────────────┐
│ React UI            │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ Game Client State   │
└──────────┬──────────┘
           │
     Reverb Events
           │
┌──────────▼──────────┐
│ Game Engine         │
│ (Pure PHP)          │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ Persistence Layer   │
│ PostgreSQL          │
└─────────────────────┘
```

The engine should know nothing about:

- React
- Reverb
- Inertia
- Database

The engine should only know:

```text
GameState -> Action -> New GameState
```

This keeps it portable and reusable.

---

# Core Concepts

## Card

```php
class Card
{
    public function __construct(
        public string $suit,
        public string $rank,
        public int $value,
    ) {}
}
```

Example:

```php
new Card(
    suit: 'hearts',
    rank: 'K',
    value: 13
);
```

---

## Deck

```php
class Deck
{
    /** @var Card[] */
    private array $cards = [];

    public function shuffle(): void {}

    public function draw(): ?Card {}
}
```

Responsibilities:

- Create a standard deck
- Shuffle cards
- Draw cards
- Support multiple decks in the future

Example:

```php
new Deck(2);
```

---

## Pile

Use piles as a generic building block.

Examples:

- Draw pile
- Discard pile
- Points pile
- Play area pile

```php
class Pile
{
    /** @var Card[] */
    private array $cards = [];
}
```

Operations:

```php
add(Card $card);

addMany(array $cards);

draw();

count();
```

---

## Hand

A Hand can simply extend Pile.

```php
class Hand extends Pile
{
}
```

War:

```php
$player->hand->draw();
```

Poker:

```php
$player->hand->cards();
```

---

# Player Model

Avoid game-specific player implementations.

```php
class Player
{
    public string $id;

    public string $name;

    public Hand $hand;

    /** @var Pile[] */
    public array $piles = [];
}
```

Example:

```php
$player->piles['points'];
```

Future uses:

- Graveyards
- Mana piles
- Treasure piles
- Discard piles

---

# Game State

This is the most important object.

Everything should derive from GameState.

```php
class GameState
{
    public string $id;

    public string $status;

    public int $turn;

    /** @var Player[] */
    public array $players;

    public array $board = [];
}
```

For War:

```php
$board = [
    'battlePile' => [],
];
```

Shared game information should live on the board.

---

# Actions

Controllers should never manipulate state directly.

Use actions.

Examples:

```php
PlayCardAction
PassAction
DrawCardAction
```

Every game should follow:

```text
Player Input
    ↓
Action
    ↓
Engine
    ↓
GameState Update
```

For War:

```php
PlayTopCardAction
```

---

# Generic Engine Contract

```php
interface GameEngine
{
    public function applyAction(
        GameState $state,
        GameAction $action
    ): GameState;
}
```

War implementation:

```php
class WarEngine implements GameEngine
{
}
```

Future implementations:

```php
PokerEngine
UnoEngine
BlackjackEngine
```

All share the same contract.

---

# Simultaneous Turns

War is not truly turn-based.

Both players act simultaneously.

Model the flow as:

```text
waiting_for_actions
```

Player A submits:

```php
PlayTopCardAction
```

Player B submits:

```php
PlayTopCardAction
```

Once both actions exist:

```php
resolveRound();
```

---

# Round Resolution

```php
public function resolveRound()
{
    $cardA = ...;
    $cardB = ...;

    if ($cardA->value > $cardB->value)
    {
        ...
    }

    if ($cardB->value > $cardA->value)
    {
        ...
    }

    if ($cardA->value === $cardB->value)
    {
        ...
    }
}
```

---

# Handling War (Tie)

Do not immediately score tied cards.

Use a shared battle pile.

```text
P1 plays 8
P2 plays 8
```

Battle pile:

```text
[
  P1: 8,
  P2: 8
]
```

Next reveal:

```text
P1: K
P2: 4
```

Player 1 wins.

All cards move from:

```text
battlePile
```

to:

```text
player1.pointsPile
```

---

# Game Flow State Machine

Avoid phase booleans.

Use explicit game phases.

```php
enum GamePhase: string
{
    case Lobby;
    case Dealing;
    case WaitingForPlays;
    case Resolving;
    case Complete;
}
```

Transitions:

```text
Lobby
 ↓
Dealing
 ↓
WaitingForPlays
 ↓
Resolving
 ↓
WaitingForPlays
 ↓
Complete
```

This scales well to more advanced games.

---

# Database Design

Store game state as JSON.

## games

```sql
id
ruleset
status
phase
created_at
updated_at
```

Examples:

```text
war
poker
blackjack
```

---

## game_players

```sql
id
game_id
user_id
seat
created_at
```

---

## game_snapshots

```sql
id
game_id
version
state_json
created_at
```

Example state:

```json
{
  "players": [],
  "board": {},
  "turn": 5,
  "phase": "waiting_for_plays"
}
```

---

## game_actions

Provides lightweight event sourcing.

```sql
id
game_id
player_id
action_type
payload
created_at
```

Example payload:

```json
{
  "action": "play_top_card"
}
```

Benefits:

- Replay games
- Anti-cheat auditing
- Debugging
- Spectator mode
- Analytics

---

# Reverb Integration

Use Reverb only as a transport layer.

When a game changes:

```php
broadcast(new GameUpdated($game));
```

Frontend:

```typescript
Echo.private(`game.${gameId}`);
```

Receives:

```typescript
{
  state: ...
}
```

Updates the UI.

---

# Hidden Information

Never trust the frontend.

The server should construct a player-specific view model.

Player A receives:

```json
{
  "myHandCount": 12,
  "opponentHandCount": 14,
  "points": 26
}
```

Not:

```json
{
  "opponentCards": [...]
}
```

The complete game state should remain server-side.

---

# Suggested Folder Structure

```text
app/

├─ GameEngine/
│
├─ Core/
│  ├─ Card.php
│  ├─ Deck.php
│  ├─ Pile.php
│  ├─ Hand.php
│  ├─ Player.php
│  ├─ GameState.php
│
├─ Actions/
│  ├─ GameAction.php
│  ├─ PlayTopCardAction.php
│
├─ Contracts/
│  ├─ GameEngine.php
│
├─ Rulesets/
│  └─ War/
│      ├─ WarEngine.php
│      ├─ WarResolver.php
│      └─ WarSetup.php
│
├─ Events/
├─ DTOs/
└─ Services/
```

---

# Recommended Core Abstractions

Keep the engine centered around:

```text
Card
Pile
Player
GameState
Action
GameEngine
Ruleset
```

Everything else becomes game-specific.

This architecture can support:

- War
- Blackjack
- Poker
- Hearts
- Spades
- Trading card games
- Deck-building games

without rewriting the core engine.

---

# Guiding Principle

The engine should always behave like a pure state machine:

```text
GameState + Action = New GameState
```

If that invariant remains true, Laravel, React, Reverb, PostgreSQL, Docker, and other infrastructure are implementation details surrounding a reusable card game platform instead of being tightly coupled to the game rules.
