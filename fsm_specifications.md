```mermaid
---
config:
  theme: default
  layout: adaptive
---
stateDiagram-v2
    [*] --> INIT
    INIT --> WAITING_FOR_PLAYERS: Server Started & Listening
    WAITING_FOR_PLAYERS --> GAME_START: 2 Clients Connected
    GAME_START --> PLAYER_TURN: Initialize Board, Assign Player 1 = X and Player 2 = O, Player 1 moves first
    PLAYER_TURN --> EVALUATE_MOVE: Active Player Sends MOVE
    PLAYER_TURN --> PLAYER_TURN: Out-of-turn or Malformed Message (Send ERROR)
    PLAYER_TURN --> GAME_OVER: CLIENT_DISCONNECTED (DISCONNECT msg, EOF, or Socket Error) Send GAME_OVER FORFEIT to opponent
    EVALUATE_MOVE --> PLAYER_TURN: Valid Move (Update board, swap turn, send STATE_UPDATE)
    EVALUATE_MOVE --> PLAYER_TURN: Invalid Move (Send ERROR to Client)
    EVALUATE_MOVE --> GAME_OVER: Victory or Draw Detected (Send STATE_UPDATE and GAME_OVER)
    GAME_OVER --> CLEANUP: Broadcast Final Results
    CLEANUP --> WAITING_FOR_PLAYERS: Reset State
```

- INIT: Initialization and server starts
- WAITING_FOR_PLAYERS: Waits until 2 players connect. 
- `GAME_START`: Initialize Board, Assign Player 1 = X and Player 2 = O, Player 1 moves first
- `PLAYER_TURN`: Waits for `MOVE` from the active player. Out-of-turn or malformed messages get `ERROR`.
- `EVALUATE_MOVE`: Validates the move, updates the board, checks for win or draw.
- `GAME_OVER`: Sends `GAME_OVER` (WIN, DRAW, or FORFEIT).
- `CLEANUP`: Closes sockets and reset state.

### Valid Moves
If invalid moves checks do not fail, then:
- Player's symbol (X or O) is placed in the cell.
- Checks for a win or draw.
- If there is no result, it swaps the active player, sends `STATE_UPDATE` to both clients, and stays in `PLAYER_TURN`.
- If there is a win or draw, it sends `STATE_UPDATE`, then `GAME_OVER`, and moves to `GAME_OVER`.

### Invalid Moves
1. The line has invalid syntax: `MALFORMED_MESSAGE`. Example: `MOVE|Player_1|abc`.
2. The server is not in `PLAYER_TURN`: `WRONG_STATE`. Example: a `MOVE` sent before `GAME_START`.
3. It is not the sender's turn: `OUT_OF_TURN`. Example: Player 2 moves on Player 1's turn.
4. The row or column is outside the grid: `INVALID_COORDS`. Example: `MOVE|Player_1|3,1|1727000008`.
5. The cell is already filled: `CELL_OCCUPIED`. Example: moving on a cell that holds `X`.

DISCONNECTIONS: Triggers CLIENT_DISCONNECTED
