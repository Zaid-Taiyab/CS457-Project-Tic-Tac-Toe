---

## 2. Application-Layer Messaging Protocol Blueprint (Sprint 1 Deliverable)

### 2.1 Message Transport & Serialization Format
- **Transport Protocol:** TCP
- **Serialization Format:** Delimited Text 
- **Framing Mechanism:** Text Delimited Protocol

- **Framing Rule:** 
1) Message fields are separated by the '|' character
2) Newlines '(\n)' determine the end of the message. 
3) Subfields within messages use the ',' character. 
4) If message happens to be split across multiple recv() calls, I store the incomplete bytes and once \n is recieved then the message is considered complete.
5) If one recv() contains several messages, I split the buffer on \n. The complete messages are processed accordingly and the remaining bytes stay in buffer for future.

- **Wire Stream Example:** CONNECT|Player_1|1727000000\nMOVE|Player_1|0,2|1727000005\nSTATE_UPDATE|SERVER|-,-,X,-,-,-,-,-,-|Player_2|1727000006\n
 
- **Field Grammar Example:**  Format: CONNECT|<PLAYER_ID>|<TIMESTAMP>\n
                              Example: CONNECT|Player_1|1727000005\n
                              Fields:
                                - MSG_TYPE   (string)  : "CONNECT"
                                - PLAYER_ID  (string)  : Alphanumeric alias of connecting player (e.g. "Player_1")
                                - TIMESTAMP  (integer) : Unix epoch timestamp in seconds

                              Format: MOVE|<PLAYER_ID>|<ROW>,<COL>|<TIMESTAMP>\n
                              Example: MOVE|Player_1|0,2|1727000005\n
                              Fields:
                                - MSG_TYPE   (string)  : "MOVE"
                                - PLAYER_ID  (string)  : Alphanumeric alias of active player (e.g. "Player_1")
                                - PAYLOAD    (integers): <row>,<col> zero-indexed grid coordinates (e.g. "0,2")
                                - TIMESTAMP  (integer) : Unix epoch timestamp in seconds

- **TCP Stream Packet Framing & Boundary Handling:**

```python
# Handling Fragmentation
def fragmentation(buffer, data):
    return buffer + data

# Handling Coalescing
def coalescing(buffer):
    parts = buffer.split(b"\n")
    messages = parts[:-1]
    buffer = parts[-1]
    return messages, buffer

# Detecting TCP Infinite Loops + Handling Disconnects
buffer = b""
while True:
    try:
        data = sock.recv(1024)
        # Handle infinite loop
        if not data:
            logger.info("Remote peer disconnected (EOF received).")
            sock.close()
            trigger_state_transition("CLIENT_DISCONNECTED")
            break
        buffer = fragmentation(buffer, data)
        messages, buffer = coalescing(buffer)
        for msg in messages:
            handle_message(msg)
    except (ConnectionResetError, BrokenPipeError,
            ConnectionAbortedError, TimeoutError) as e:
        logger.warning(f"Connection lost abruptly: {e}")
        trigger_state_transition("CLIENT_DISCONNECTED")
        break
```
  
### 2.2 Message Schema Definitions

#### Message Types:
1. `CONNECT` (Client -> Server): Request to join the game room.
2. `LOBBY_WAIT` (Server -> Client): Notification that server is waiting for Player 2.
3. `GAME_START` (Server -> Clients): Game initiated, assigns roles (e.g. Player X vs Player O).
4. `MOVE` (Client -> Server): Player action (Current player selects a coordinate on grid to mark it).
5. `STATE_UPDATE` (Server -> Clients): Broadcast current game board / state and active player turn.
6. `GAME_OVER` (Server -> Clients): Victory / Draw notification with WIN/DRAW/FORFEIT.
7. `ERROR` (Server -> Client): Invalid move or malformed packet error.
8. `DISCONNECT` (Client -> Server): Player disconnects from the game. Results in forfeit win for opponent.

#### Example Text Delimited Protocol Schema:
```
  CONNECT|<PLAYER_ID>|<TIMESTAMP>\n
    Example: CONNECT|Player_1|1727000005\n
    Fields:
      - MSG_TYPE   (string)  : "CONNECT"
      - PLAYER_ID  (string)  : Alphanumeric alias of connecting player (e.g. "Player_1")
      - TIMESTAMP  (integer) : Unix epoch timestamp in seconds

  MOVE|<PLAYER_ID>|<ROW>,<COL>|<TIMESTAMP>\n
    Example: MOVE|Player_1|0,2|1727000005\n
    Fields:
      - MSG_TYPE   (string)  : "MOVE"
      - PLAYER_ID  (string)  : Alphanumeric alias of active player (e.g. "Player_1")
      - PAYLOAD    (integers): <row>,<col> zero-indexed grid coordinates (e.g. "0,2")
      - TIMESTAMP  (integer) : Unix epoch timestamp in seconds
    
  LOBBY_WAIT|SERVER|<STATUS>|<TIMESTAMP>\n
    Example: LOBBY_WAIT|SERVER|WAITING_FOR_PLAYER_2|1727000006\n
    Fields:
      - MSG_TYPE  (string)  : "LOBBY_WAIT"
      - SOURCE    (string)  : "SERVER"
      - STATUS    (string)  : Current lobby status (e.g. "WAITING_FOR_PLAYER_2")
      - TIMESTAMP (integer) : Unix epoch timestamp in seconds

  GAME_START|SERVER|<PLAYER_1>,<SYMBOL>|<PLAYER_2>,<SYMBOL>|<TIMESTAMP>\n
    Example: GAME_START|SERVER|Player_1,X|Player_2,O|1727000007\n
    Fields:
      - MSG_TYPE          (string)  : "GAME_START"
      - SOURCE            (string)  : "SERVER"
      - PLAYER_1,SYMBOL   (string)  : Player 1 identifier and assigned symbol
      - PLAYER_2,SYMBOL   (string)  : Player 2 identifier and assigned symbol
      - TIMESTAMP         (integer) : Unix epoch timestamp in seconds

  STATE_UPDATE|SERVER|<BOARD>|<ACTIVE_PLAYER>|<TIMESTAMP>\n
    Example: STATE_UPDATE|SERVER|X,-,-,-,O,-,-,-,-|Player_2|1727000011\n
    Fields:
      - MSG_TYPE       (string)  : "STATE_UPDATE"
      - SOURCE         (string)  : "SERVER"
      - BOARD          (string)  : Nine comma-separated cells representing the current board
      - ACTIVE_PLAYER  (string)  : Player whose turn is next
      - TIMESTAMP      (integer) : Unix epoch timestamp in seconds

  GAME_OVER|SERVER|<RESULT>|<PLAYER_ID>|<TIMESTAMP>\n
    Example: GAME_OVER|SERVER|WIN|Player_1|1727000015\n
    Fields:
      - MSG_TYPE   (string)  : "GAME_OVER"
      - SOURCE     (string)  : "SERVER"
      - RESULT     (string)  : Game result: "WIN", "DRAW", or "FORFEIT"
      - PLAYER_ID  (string)  : Winning player, or "-" if DRAW
      - TIMESTAMP  (integer) : Unix epoch timestamp in seconds

  ERROR|SERVER|<ERROR_CODE>|<TIMESTAMP>\n
    Example: ERROR|SERVER|OUT_OF_TURN|1727000012\n
    Fields:
      - MSG_TYPE    (string)  : "ERROR"
      - SOURCE      (string)  : "SERVER"
      - ERROR_CODE  (string)  : MALFORMED_MESSAGE | WRONG_STATE | OUT_OF_TURN | INVALID_COORDS | CELL_OCCUPIED
      - TIMESTAMP   (integer) : Unix epoch timestamp in seconds

  DISCONNECT|<PLAYER_ID>|<TIMESTAMP>\n
    Example: DISCONNECT|Player_1|1727000020\n
    Fields:
      - MSG_TYPE   (string)  : "DISCONNECT"
      - PLAYER_ID  (string)  : Alphanumeric alias of disconnected player
      - TIMESTAMP  (integer) : Unix epoch timestamp in seconds
```

### 2.3 Game State Machine (FSM) Design (Sprint 1 Deliverable)
- In stateDiagramv2 in fsm_specification.md


### 2.4 Connection Termination & Socket Lifecycle Management

1. Handling Disconnet Gracefully: the server receives `DISCONNECT|<PLAYER_ID>|<TIMESTAMP>\n`. Server sends `GAME_OVER|SERVER|FORFEIT|<surviving player>|<TIMESTAMP>\n` to opponent who wins by forfeit if game is in session.
2. Transport Layer Teardown (TCP FIN): `recv()` returns `b""`. Handled with trigger_state_transition("CLIENT_DISCONNECTED")
3. Abrupt Termination (TCP RST, network drop, timeout): `recv()` or `send()` raises an exception. Handled with trigger_state_transition("CLIENT_DISCONNECTED")
4. Socket Rule and Infinite Loop: when the peer closes cleanly, `recv()` does not raise an exception, it returns `b""`. Infinite Looping Handled with:

        ```python
        # Handle Socket Rule and Infinite Loop
        if not data:
            logger.info("Remote peer disconnected (EOF received).")
            sock.close()
            trigger_state_transition("CLIENT_DISCONNECTED")
            break
        ```
5. ConnectionRestError, BrokenPipeError, TimeoutError: Handled below

```python
buffer = b""
while True:
    try:
        data = sock.recv(1024)
        # Handle Socket Rule and Infinite Loop
        if not data:
            logger.info("Remote peer disconnected (EOF received).")
            sock.close()
            trigger_state_transition("CLIENT_DISCONNECTED")
            break
        buffer = fragmentation(buffer, data)
        messages, buffer = coalescing(buffer)
        for msg in messages:
            handle_message(msg)
    except (ConnectionResetError, BrokenPipeError,
            ConnectionAbortedError, TimeoutError) as e:
        logger.warning(f"Connection lost abruptly: {e}")
        trigger_state_transition("CLIENT_DISCONNECTED")
        break

```


---