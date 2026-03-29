
# 1. Functional Requirements

- Two players play alternately
    
- 3x3 board
    
- Players place symbols (X / O)
    
- Validate moves
    
- Detect winner
    
- Detect draw
    
- Display board
    

---

# 2. Non-Functional Requirements

- Low latency
    
- Simple and extensible
    
- Thread-safe (optional multiplayer)
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Game|Controls game flow|
|Board|Maintains grid|
|Player|Player info|
|Move|Represents a move|
|GameStatus|Tracks game state|

---

# 4. Design Approach

- Use **Strategy Pattern** for winning logic (extensible for NxN)
    
- Use **Factory Pattern** for player creation
    

---

# 5. Flows

---

## Game Start

1. Initialize board
    
2. Add players
    
3. Set current turn
    

---

## Move Flow

1. Player selects cell
    
2. Validate move
    
3. Place symbol
    
4. Check winner
    
5. Switch turn
    

---

# 6. Code

```java
import java.util.*;

// ENUMS
enum Symbol {
    X, O
}

enum GameStatus {
    IN_PROGRESS, WIN, DRAW
}

// MOVE
class Move {
    int row, col;

    public Move(int r, int c) {
        this.row = r;
        this.col = c;
    }
}

// PLAYER
class Player {
    String name;
    Symbol symbol;

    public Player(String name, Symbol symbol) {
        this.name = name;
        this.symbol = symbol;
    }
}

// BOARD
class Board {
    private int size;
    private Symbol[][] grid;

    public Board(int n) {
        this.size = n;
        grid = new Symbol[n][n];
    }

    public boolean isValidMove(int r, int c) {
        return r >= 0 && c >= 0 && r < size && c < size && grid[r][c] == null;
    }

    public void placeMove(int r, int c, Symbol symbol) {
        grid[r][c] = symbol;
    }

    public Symbol get(int r, int c) {
        return grid[r][c];
    }

    public int getSize() {
        return size;
    }
}

// STRATEGY: WIN CHECK
interface WinningStrategy {
    boolean checkWinner(Board board, Move move, Symbol symbol);
}

// SIMPLE STRATEGY
class DefaultWinningStrategy implements WinningStrategy {

    public boolean checkWinner(Board board, Move move, Symbol symbol) {
        int n = board.getSize();
        boolean rowWin = true, colWin = true, diag = true, antiDiag = true;

        for (int i = 0; i < n; i++) {
            if (board.get(move.row, i) != symbol) rowWin = false;
            if (board.get(i, move.col) != symbol) colWin = false;
            if (board.get(i, i) != symbol) diag = false;
            if (board.get(i, n - i - 1) != symbol) antiDiag = false;
        }

        return rowWin || colWin || diag || antiDiag;
    }
}

// GAME
class Game {
    private Board board;
    private Player p1, p2;
    private Player current;
    private WinningStrategy strategy;
    private GameStatus status;

    public Game(int size, Player p1, Player p2) {
        this.board = new Board(size);
        this.p1 = p1;
        this.p2 = p2;
        this.current = p1;
        this.strategy = new DefaultWinningStrategy();
        this.status = GameStatus.IN_PROGRESS;
    }

    public void makeMove(int r, int c) {
        if (!board.isValidMove(r, c)) {
            throw new RuntimeException("Invalid move");
        }

        board.placeMove(r, c, current.symbol);

        Move move = new Move(r, c);

        if (strategy.checkWinner(board, move, current.symbol)) {
            status = GameStatus.WIN;
            System.out.println(current.name + " wins!");
            return;
        }

        if (isDraw()) {
            status = GameStatus.DRAW;
            System.out.println("Game is a draw");
            return;
        }

        switchTurn();
    }

    private boolean isDraw() {
        int n = board.getSize();
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                if (board.get(i, j) == null) return false;
            }
        }
        return true;
    }

    private void switchTurn() {
        current = (current == p1) ? p2 : p1;
    }
}
```

---

# 7. Complexity Analysis

- Move → O(N) (row/col/diag check)
    
- Board access → O(1)
    
- Game → O(N²) total
    

---

# 8. Design Patterns Used

- Strategy Pattern → Winning logic
    
- Factory Pattern → Player creation (extendable)
    

---

# 9. Key Design Decisions

- Board generalized to NxN
    
- Winning logic abstracted via strategy
    
- Minimal state stored for simplicity
    
- Turn switching handled centrally
    
- Validation before execution
    

---

# 10. Edge Cases Handled

- Invalid moves
    
- Duplicate cell selection
    
- Early win detection
    
- Full board draw
    

---

# 11. Possible Extensions

- O(1) winner check using counters
    
- Multiplayer online mode
    
- AI opponent (minimax)
    
- Undo/redo moves
    
- Time-based turns
    
- Score tracking across games
    
- GUI implementation