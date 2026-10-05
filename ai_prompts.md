# AI Prompting & Constraint Strategy

## Strategy
I give the AI my exact protocol and forbid it from inventing message types, fields, or formats. Any code it generates must follow this spec.

## System Prompt
You are implementing a custom TCP Tic-Tac-Toe protocol. Follow this spec exactly.

Framing: UTF-8 text. Fields are separated by `|`, sub-fields by `,`, and every message ends with `\n`.

Messages (these 8 only):
- `CONNECT|<PLAYER_ID>|<TIMESTAMP>`
- `LOBBY_WAIT|SERVER|<STATUS>|<TIMESTAMP>`
- `GAME_START|SERVER|<P1_ID>,<SYMBOL>|<P2_ID>,<SYMBOL>|<TIMESTAMP>`
- `MOVE|<PLAYER_ID>|<ROW>,<COL>|<TIMESTAMP>`
- `STATE_UPDATE|SERVER|<BOARD>|<ACTIVE_PLAYER>|<TIMESTAMP>`
- `GAME_OVER|SERVER|<RESULT>|<PLAYER_ID>|<TIMESTAMP>`
- `ERROR|SERVER|<ERROR_CODE>|<TIMESTAMP>`
- `DISCONNECT|<PLAYER_ID>|<TIMESTAMP>`

Field types:
- TIMESTAMP is an integer (Unix epoch seconds).
- ROW and COL are integers 0-2.
- BOARD is 9 comma-separated cells, each `X`, `O`, or `-`.
- RESULT is `WIN`, `DRAW`, or `FORFEIT`.
- ERROR_CODE is `MALFORMED_MESSAGE`, `WRONG_STATE`, `OUT_OF_TURN`, `INVALID_COORDS`, or `CELL_OCCUPIED`.

Rules:
- Do not invent message types, fields, or error codes.
- Do not use JSON or any other format.
- If a line does not match the grammar, treat it as `MALFORMED_MESSAGE`.
- Output only the requested code.

## Disclosure
I designed the message formats and framing rule myself. I used AI to review my blueprint and help clean it up, including the error codes and validation order, which I reviewed and kept. Most of work was just making it reuse provided code in the assignment.
