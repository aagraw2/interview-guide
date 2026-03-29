
# 1. Functional Requirements

- Two players play alternately
    
- Standard chess board (8x8)
    
- Support all pieces (King, Queen, Rook, Bishop, Knight, Pawn)
    
- Validate legal moves
    
- Capture opponent pieces
    
- Detect check, checkmate, stalemate
    
- Maintain game history
    

---

# 2. Non-Functional Requirements

- Low latency move validation
    
- Extensible (variants like blitz, AI)
    
- Thread-safe (for online play)
    
- Maintainable and modular
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Game|Orchestrates gameplay|
|Board|Represents chess board|
|Cell|Holds piece|
|Piece|Abstract base class|
|Move|Represents a move|
|Player|Player info|
|GameStatus|Tracks game state|

---

# 4. Design Approach

- Use **Strategy Pattern** for piece movement
    
- Use **Factory Pattern** for piece creation
    
- Use **Command Pattern** for moves (undo/redo possible)
    

---

# 5. Flows

---

## Game Start

1. Initialize board
    
2. Place pieces
    
3. Assign players
    
4. Set current turn
    

---

## Move Flow

1. Player selects piece
    
2. Validate move
    
3. Execute move
    
4. Check for check/checkmate
    
5. Switch turn
    

---

## Capture Flow

1. Move to occupied cell
    
2. Remove opponent piece
    

---

# 6. Code

```java
import java.util.*;

// ENUMS
enum Color { WHITE, BLACK }

enum GameStatus {
    ACTIVE, CHECK, CHECKMATE, STALEMATE
}

// POSITION
class Position {
    int row, col;

    public Position(int r, int c) {
        this.row = r;
        this.col = c;
    }
}

// ABSTRACT PIECE
abstract class Piece {
    Color color;

    public Piece(Color color) {
        this.color = color;
    }

    abstract List<Position> getValidMoves(Board board, Position pos);
}

// KING
class King extends Piece {
    public King(Color color) { super(color); }

    public List<Position> getValidMoves(Board board, Position pos) {
        List<Position> moves = new ArrayList<>();
        int[] dx = {-1, -1, -1, 0, 0, 1, 1, 1};
        int[] dy = {-1, 0, 1, -1, 1, -1, 0, 1};

        for (int i = 0; i < 8; i++) {
            int x = pos.row + dx[i];
            int y = pos.col + dy[i];

            if (board.isValid(x, y)) {
                moves.add(new Position(x, y));
            }
        }
        return moves;
    }
}

// ROOK
class Rook extends Piece {
    public Rook(Color color) { super(color); }

    public List<Position> getValidMoves(Board board, Position pos) {
        List<Position> moves = new ArrayList<>();

        // horizontal & vertical
        for (int i = 0; i < 8; i++) {
            if (i != pos.row) moves.add(new Position(i, pos.col));
            if (i != pos.col) moves.add(new Position(pos.row, i));
        }
        return moves;
    }
}

// BOARD
class Board {
    Piece[][] grid = new Piece[8][8];

    public boolean isValid(int r, int c) {
        return r >= 0 && c >= 0 && r < 8 && c < 8;
    }

    public Piece get(Position pos) {
        return grid[pos.row][pos.col];
    }

    public void set(Position pos, Piece piece) {
        grid[pos.row][pos.col] = piece;
    }
}

// MOVE (Command Pattern)
class Move {
    Position from;
    Position to;
    Piece piece;
    Piece captured;

    public Move(Position from, Position to, Piece piece) {
        this.from = from;
        this.to = to;
        this.piece = piece;
    }
}

// PLAYER
class Player {
    String name;
    Color color;

    public Player(String name, Color color) {
        this.name = name;
        this.color = color;
    }
}

// GAME
class Game {
    private Board board = new Board();
    private Player white;
    private Player black;
    private Player current;
    private GameStatus status = GameStatus.ACTIVE;
    private List<Move> history = new ArrayList<>();

    public Game(Player w, Player b) {
        this.white = w;
        this.black = b;
        this.current = w;
        initialize();
    }

    private void initialize() {
        board.set(new Position(0, 0), new Rook(Color.WHITE));
        board.set(new Position(7, 7), new Rook(Color.BLACK));
        board.set(new Position(0, 4), new King(Color.WHITE));
        board.set(new Position(7, 4), new King(Color.BLACK));
    }

    public void makeMove(Position from, Position to) {
        Piece piece = board.get(from);

        if (piece == null || piece.color != current.color) {
            throw new RuntimeException("Invalid move");
        }

        List<Position> validMoves = piece.getValidMoves(board, from);

        boolean allowed = validMoves.stream()
            .anyMatch(p -> p.row == to.row && p.col == to.col);

        if (!allowed) {
            throw new RuntimeException("Illegal move");
        }

        Move move = new Move(from, to, piece);
        move.captured = board.get(to);

        board.set(to, piece);
        board.set(from, null);

        history.add(move);

        switchTurn();
    }

    private void switchTurn() {
        current = (current == white) ? black : white;
    }
}
```

---

# 7. Complexity Analysis

- Move validation → O(M) (M = possible moves for piece)
    
- Board access → O(1)
    
- Game loop → O(T)
    

---

# 8. Design Patterns Used

- Strategy Pattern → Piece movement logic
    
- Command Pattern → Move encapsulation
    
- Factory Pattern → Piece creation (extendable)
    

---

# 9. Key Design Decisions

- Each piece encapsulates its movement logic
    
- Board uses 2D array for constant access
    
- Move stored for history/undo
    
- Turn-based control enforced in Game
    
- Separation of concerns across entities
    

---

# 10. Edge Cases Handled

- Invalid moves
    
- Moving opponent’s piece
    
- Capturing pieces
    
- Out-of-bound positions
    

---

# 11. Possible Extensions

- Add all piece types (Queen, Bishop, Knight, Pawn)
    
- Implement check/checkmate detection
    
- Add castling, en passant, pawn promotion
    
- Add undo/redo functionality
    
- Add AI opponent
    
- Add timers (blitz mode)
    
- Add online multiplayer
    
- Add board visualization