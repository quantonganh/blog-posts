---
title: "Learning Rust by building Tetris: my favorite childhood game"
date: 2023-10-03
description:
categories:
- Programming
tags:
- rust
- tetris
---
![Play tetris 2-player mode in the terminal](/2023/10/03/tetris-2-player.gif)

After completing the [Rust book](https://doc.rust-lang.org/book/) and working through [rustlings](https://github.com/rust-lang/rustlings), I found myself standing at the crossroads, wondering where to go next. It was then that I had an idea - a project that would allow me to apply my newfound knowledge and create something meaningful. My favorite childhood game, Tetris, became the inspiration for my next coding adventure.

Since I want it to be playable on Windows, I chose the TUI library [crossterm](https://github.com/crossterm-rs/crossterm).

1. [Drawing borders and the playfield](#drawing-borders-and-playfield)
2. [Drawing tetrominoes](#drawing-tetrominoes)
3. [Automatic and soft drop](#automatic-and-soft-drop)
4. [Lock tetromino and move to the next](#lock-tetromino-and-move-to-next)
5. [Handling key events](#handling-key-events)
6. [Clearing the filled lines](#clearing-filled-lines)
7. [Game over](#game-over)
8. [Pause and Quit](#pause-and-quit)
9. [Reset](#reset)
10. [Multiplayer mode](#multiplayer-mode)

#### 1. Drawing borders and playfield {#drawing-borders-and-playfield}

The initial step is to draw the borders. The standard Tetris playing field has 20 rows and 10 columns. I've designated the pipe operator (`|`) for the left and right borders and a dash (`-`) for the top and bottom borders:

The function takes input in the form of coordinates, width and height parameters representing the playing field:

```rust
fn render_frame(
    stdout: &mut io::Stdout,
    title: &str,
    start_x: usize,
    start_y: usize,
    width: usize,
    height: usize,
) -> Result<()> {
    execute!(
        stdout,
        SetForegroundColor(Color::White),
        SetBackgroundColor(Color::Black),
    )?;

    // Print the top border
    let left = (width - title.len() - 2) / 2;
    execute!(
        stdout,
        MoveTo(start_x as u16, start_y as u16),
        Print(format!(
            "|{} {} {}|",
            "-".repeat(left as usize),
            title,
            "-".repeat(width as usize - left as usize - title.len() - 2)
        )),
    )?;

    // Print the left and right borders
    for index in 1..height {
        execute!(
            stdout,
            MoveTo(start_x as u16, start_y as u16 + index as u16),
            Print("|"),
            MoveTo(
                start_x as u16 + width as u16 + 1,
                start_y as u16 + index as u16
            ),
            Print("|"),
        )?;
    }

    // Print the bottom border
    execute!(
        stdout,
        MoveTo(start_x as u16, start_y as u16 + height as u16),
        Print(format!("|{}|", ("-").repeat(width as usize))),
    )?;

    stdout.flush()?;

    Ok(())
}
```

the borders can be rendered as:

```rust
const PLAY_WIDTH: usize = 10;
const PLAY_HEIGHT: usize = 20;
const SQUARE_BRACKETS: &str = "[ ]";
```

```rust
render_frame(
    stdout,
    "Tetris",
    self.start_x,
    self.start_y,
    PLAY_WIDTH * 3,
    PLAY_HEIGHT + 1,
)?;
```

As I use square brackets to make up the tetromino, we need to multiply `PLAY_WIDTH` by 3:

```rust
fn create_grid(
    width: usize,
    height: usize,
) -> Vec<Vec<Cell>> {
    let mut grid = vec![vec![EMPTY_CELL; width]; height];

    grid
}
```

```rust
let play_grid = create_grid(PLAY_WIDTH, PLAY_HEIGHT);
```

#### 2. Drawing Tetrominoes {#drawing-tetrominoes}

A tetromino can be represented as:


```rust
struct Tetromino {
    states: Vec<Vec<Vec<Cell>>>,
    current_state: usize,
    position: Position,
}
```

```rust
const I_CELL: Cell = Cell {
    symbols: SQUARE_BRACKETS,
    color: Color::Cyan,
};
```

```rust
let i_tetromino_states: Vec<Vec<Vec<Cell>>> = vec![
    vec![
        vec![EMPTY_CELL, EMPTY_CELL, EMPTY_CELL, EMPTY_CELL],
        vec![I_CELL, I_CELL, I_CELL, I_CELL],
        vec![EMPTY_CELL, EMPTY_CELL, EMPTY_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, EMPTY_CELL, EMPTY_CELL, EMPTY_CELL],
    ],
    vec![
        vec![EMPTY_CELL, EMPTY_CELL, I_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, EMPTY_CELL, I_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, EMPTY_CELL, I_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, EMPTY_CELL, I_CELL, EMPTY_CELL],
    ],
    vec![
        vec![EMPTY_CELL, EMPTY_CELL, EMPTY_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, EMPTY_CELL, EMPTY_CELL, EMPTY_CELL],
        vec![I_CELL, I_CELL, I_CELL, I_CELL],
        vec![EMPTY_CELL, EMPTY_CELL, EMPTY_CELL, EMPTY_CELL],
    ],
    vec![
        vec![EMPTY_CELL, I_CELL, EMPTY_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, I_CELL, EMPTY_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, I_CELL, EMPTY_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, I_CELL, EMPTY_CELL, EMPTY_CELL],
    ],
];
```

To draw a tetromino, we just need to loop through the its current state, and draw each cell with corresponding color:


```rust
fn render_current_tetromino(&self, stdout: &mut std::io::Stdout) -> Result<()> {
    let current_tetromino = &self.current_tetromino;
    for (row_index, row) in current_tetromino.states[current_tetromino.current_state]
        .iter()
        .enumerate()
    {
        for (col_index, &ref cell) in row.iter().enumerate() {
            let grid_x = current_tetromino.position.col + col_index as isize;
            let grid_y = current_tetromino.position.row + row_index as isize;

            if cell.symbols != SPACE {
                if grid_x < PLAY_WIDTH as isize && grid_y < PLAY_HEIGHT as isize {
                    execute!(
                        stdout,
                        SavePosition,
                        MoveTo(
                            self.start_x as u16 + 1 + grid_x as u16 * CELL_WIDTH as u16,
                            self.start_y as u16 + 1 + grid_y as u16
                        ),
                        SetForegroundColor(cell.color),
                        SetBackgroundColor(Color::Black),
                        Print(cell.symbols),
                        ResetColor,
                        RestorePosition,
                    )?;
                }
            }
        }
    }

    Ok(())
}
```

#### 3. Automatic and soft drop {#automatic-and-soft-drop}

##### 3.1 Automatic drop

First, we need to write a function to detect the collision. To do this, we need to check if the new column / row (after moving) goes outside of the playfield, or if that cell is already occupied:

```rust
fn can_move(&mut self, tetromino: &Tetromino, new_row: i16, new_col: i16) -> bool {
    for (t_row, row) in tetromino.get_cells().iter().enumerate() {
        for (t_col, &ref cell) in row.iter().enumerate() {
            if cell.symbols == SQUARE_BRACKETS {
                let grid_x = new_col + t_col as i16;
                let grid_y = new_row + t_row as i16;

                if grid_x < 0
                    || grid_x >= PLAY_WIDTH as i16
                    || grid_y >= PLAY_HEIGHT as i16
                    || self.play_grid[grid_y as usize][grid_x as usize].symbols
                        == SQUARE_BRACKETS
                {
                    return false;
                }
            }
        }
    }

    true
}
```

At a regular interval, we will check if a tetromino can be moved down, and if so, we will increase the row by 1, clear the old tetromino and draw a new one:

```rust
let mut drop_timer = Instant::now();
if drop_timer.elapsed() >= Duration::from_millis(self.drop_interval) {
    let mut tetromino = self.current_tetromino.clone();
    let can_move_down = self.can_move(
        &tetromino,
        tetromino.position.row as i16 + 1,
        tetromino.position.col as i16,
    );

    if can_move_down {
        tetromino.move_down(self, stdout)?;
        self.current_tetromino = tetromino;
    } else {
        self.lock_and_move_to_next(&tetromino, stdout)?;
    }

    self.render_current_tetromino(stdout)?;

    drop_timer = Instant::now();
}
```

```rust
fn move_down(&mut self, game: &mut Game, stdout: &mut std::io::Stdout) -> Result<()> {
    if game.can_move(self, self.position.row as i16 + 1, self.position.col as i16) {
        game.clear_tetromino(stdout)?;
        self.position.row += 1;
    }

    Ok(())
}
```

To clear the old tetromino, we just need to draw an empty cell:

```rust
fn clear_tetromino(&mut self, stdout: &mut std::io::Stdout) -> Result<()> {
    let tetromino = &self.current_tetromino;
    for (row_index, row) in tetromino.states[tetromino.current_state].iter().enumerate() {
        for (col_index, &ref cell) in row.iter().enumerate() {
            let grid_x = tetromino.position.col + col_index as isize;
            let grid_y = tetromino.position.row + row_index as isize;

            if cell.symbols != SPACE {
                execute!(
                    stdout,
                    SetBackgroundColor(Color::Black),
                    SavePosition,
                    MoveTo(
                        self.start_x as u16 + 1 + grid_x as u16 * CELL_WIDTH as u16,
                        self.start_y as u16 + 1 + grid_y as u16,
                    ),
                    Print(SPACE),
                    ResetColor,
                    RestorePosition
                )?;
            }
        }
    }

    Ok(())
}
```

##### 3.2. Soft drop {#soft-drop}

At the starting levels, we may need a way to make the tetromino drop faster than the regular interval. That's when the soft drop comes into play.

```rust
if kind == KeyEventKind::Press {
    let mut tetromino = self.current_tetromino.clone();
    match code {
        KeyCode::Char('s') | KeyCode::Up => {
            if soft_drop_timer.elapsed()
                >= (Duration::from_millis(self.drop_interval / 8))
            {
                let mut tetromino = self.current_tetromino.clone();
                if self.can_move(
                    &tetromino,
                    tetromino.position.row as i16 + 1,
                    tetromino.position.col as i16,
                ) {
                    tetromino.move_down(self, stdout)?;
                    self.current_tetromino = tetromino;
                } else {
                    self.lock_and_move_to_next(&tetromino, stdout)?;
                }

                soft_drop_timer = Instant::now();
            }
        }
```

#### 4. Lock tetromino and move the next {#lock-tetromino-and-move-to-next}

When a tetromino reaches the bottom, we need to lock it at that position and move on to the next one:

```rust
fn lock_and_move_to_next(
    &mut self,
    tetromino: &Tetromino,
    stdout: &mut io::Stdout,
) -> Result<()> {
    self.lock_tetromino(tetromino, stdout)?;
    self.move_to_next(stdout)?;

    Ok(())
}
```

```rust
fn lock_tetromino(&mut self, tetromino: &Tetromino, stdout: &mut io::Stdout) -> Result<()> {
    for (ty, row) in tetromino.get_cells().iter().enumerate() {
        for (tx, &ref cell) in row.iter().enumerate() {
            if cell.symbols == SQUARE_BRACKETS {
                let grid_x = (tetromino.position.col as usize).wrapping_add(tx);
                let grid_y = (tetromino.position.row as usize).wrapping_add(ty);

                self.play_grid[grid_y][grid_x] = cell.clone();
            }
        }
    }

    self.clear_filled_rows(stdout)?;

    Ok(())
}

fn move_to_next(&mut self, stdout: &mut io::Stdout) -> Result<()> {
    self.current_tetromino = self.next_tetromino.clone();
    self.current_tetromino.position.row = 0;
    self.current_tetromino.position.col =
        (PLAY_WIDTH - tetromino_width(&self.current_tetromino.states[0])) as isize / 2;
    self.render_current_tetromino(stdout)?;

    self.next_tetromino = Tetromino::new(true);
    self.render_next_tetromino(stdout)?;

    Ok(())
}
```

The next tetromino can be rendered the same as the current one; we just need different coordinates.

#### 5. Handling key events {#handling-key-events}

Similar to the automatic drop, we need to handle the key events to move left, right, rotate and hard drop:

```rust
if poll(Duration::from_millis(10))? {
    let event = read()?;
    match event {
        Event::Key(KeyEvent {
            code,
            state: _,
            kind,
            modifiers: _,
        }) => {
            if kind == KeyEventKind::Press {
                let mut tetromino = self.current_tetromino.clone();
                match code {
                    KeyCode::Char('h') | KeyCode::Left => {
                        tetromino.move_left(self, stdout)?;
                        self.current_tetromino = tetromino;
                    }
                    KeyCode::Char('l') | KeyCode::Right => {
                        tetromino.move_right(self, stdout)?;
                        self.current_tetromino = tetromino;
                    }
                    KeyCode::Char(' ') => {
                        tetromino.rotate(self, stdout)?;
                        self.current_tetromino = tetromino;
                    }
                    KeyCode::Char('j') | KeyCode::Down => {
                        tetromino.hard_drop(self, stdout)?;
                        self.lock_and_move_to_next(&tetromino, stdout)?;
                    }
                    _ => {}
                }
            }
        }
        _ => {}
    }
    self.render_current_tetromino(stdout)?;
}
```

Moving left, right is straightforward, but what about the rotate? When considering the algorithm for rotating a tetromino, I discovered that we can simplify it by storing all states of a tetromino:

```rust
let t_tetromino_states: Vec<Vec<Vec<Cell>>> = vec![
    vec![
        vec![EMPTY_CELL, T_CELL, EMPTY_CELL],
        vec![T_CELL, T_CELL, T_CELL],
        vec![EMPTY_CELL, EMPTY_CELL, EMPTY_CELL],
    ],
    vec![
        vec![EMPTY_CELL, T_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, T_CELL, T_CELL],
        vec![EMPTY_CELL, T_CELL, EMPTY_CELL],
    ],
    vec![
        vec![EMPTY_CELL, EMPTY_CELL, EMPTY_CELL],
        vec![T_CELL, T_CELL, T_CELL],
        vec![EMPTY_CELL, T_CELL, EMPTY_CELL],
    ],
    vec![
        vec![EMPTY_CELL, T_CELL, EMPTY_CELL],
        vec![T_CELL, T_CELL, EMPTY_CELL],
        vec![EMPTY_CELL, T_CELL, EMPTY_CELL],
    ],
];
```

and switch to the next state when rotating:

```rust
fn rotate(&mut self, game: &mut Game, stdout: &mut std::io::Stdout) -> Result<()> {
    let next_state = (self.current_state + 1) % (self.states.len());

    let mut temp_tetromino = self.clone();
    temp_tetromino.current_state = next_state;

    if game.can_move(
        &temp_tetromino,
        self.position.row as i16,
        self.position.col as i16,
    ) {
        game.clear_tetromino(stdout)?;
        self.current_state = next_state;
    }

    Ok(())
}
```

The `hard_drop` function can be implemented by moving down infinitely until it reaches the bottom:

```rust
fn hard_drop(&mut self, game: &mut Game, stdout: &mut std::io::Stdout) -> Result<()> {
    while game.can_move(self, self.position.row as i16 + 1, self.position.col as i16) {
        game.clear_tetromino(stdout)?;
        self.position.row += 1;
    }

    Ok(())
}
```

#### 6. Clearing the filled lines {#clearing-filled-lines}

After a tetromino reaches the bottom, we need to check if there are any filled lines. If so, we need to clear them from the playfield and update the score/lines:

```rust
fn clear_filled_rows(&mut self, stdout: &mut io::Stdout) -> Result<()> {
    let mut filled_rows: Vec<usize> = Vec::new();

    for row_index in (0..PLAY_HEIGHT).rev() {
        if self.play_grid[row_index][0..PLAY_WIDTH]
            .iter()
            .all(|cell| cell.symbols == SQUARE_BRACKETS)
        {
            filled_rows.push(row_index);
        }
    }

    let new_row = vec![EMPTY_CELL; PLAY_WIDTH];
    for &row_index in filled_rows.iter().rev() {
        self.play_grid.remove(row_index);
        self.play_grid.insert(0, new_row.clone());

        self.lines += 1;
    }

    let num_filled_rows = filled_rows.len();
    match num_filled_rows {
        1 => {
            self.score += 100 * (self.level + 1);
        }
        2 => {
            self.score += 300 * (self.level + 1);
        }
        3 => {
            self.score += 500 * (self.level + 1);
        }
        4 => {
            self.score += 800 * (self.level + 1);
        }
        _ => (),
    }

    self.render_changed_portions(stdout)?;

    Ok(())
}
```

To minimize flickering, we only need to re-render the portions that have been changed:

```rust
fn render_changed_portions(&self, stdout: &mut std::io::Stdout) -> Result<()> {
    self.render_play_grid(stdout)?;

    let stats_start_x = self.start_x - STATS_WIDTH - DISTANCE - 1;
    execute!(
        stdout,
        SetForegroundColor(Color::White),
        SetBackgroundColor(Color::Black),
        SavePosition,
        MoveTo(
            stats_start_x as u16 + 2 + "Score: ".len() as u16,
            self.start_y as u16 + 2
        ),
        Print(self.score),
        MoveTo(
            stats_start_x as u16 + 2 + "Lines: ".len() as u16,
            self.start_y as u16 + 3
        ),
        Print(self.lines),
        MoveTo(
            stats_start_x as u16 + 2 + "Level: ".len() as u16,
            self.start_y as u16 + 4
        ),
        Print(self.level),
        ResetColor,
        RestorePosition,
    )?;

    Ok(())
}
```

#### 7. Game over {#game-over}

Whenever a new tetromino is spawned, if it cannot be placed in the playfield, it means it's game over:

```rust
fn lock_and_move_to_next(
    &mut self,
    tetromino: &Tetromino,
    stdout: &mut io::Stdout,
) -> Result<()> {
    self.lock_tetromino(tetromino, stdout)?;
    self.move_to_next(stdout)?;

    if self.is_game_over() {
        self.handle_game_over(stdout)?;
    }

    Ok(())
}

fn is_game_over(&mut self) -> bool {
    let tetromino = self.current_tetromino.clone();

    let next_state = (tetromino.current_state + 1) % (tetromino.states.len());
    let mut temp_tetromino = tetromino.clone();
    temp_tetromino.current_state = next_state;

    if !self.can_move(
        &tetromino,
        tetromino.position.row as i16,
        tetromino.position.col as i16 - 1,
    ) && !self.can_move(
        &tetromino,
        tetromino.position.row as i16,
        tetromino.position.col as i16 + 1,
    ) && !self.can_move(
        &tetromino,
        tetromino.position.row as i16 + 1,
        tetromino.position.col as i16,
    ) && !self.can_move(
        &temp_tetromino,
        tetromino.position.row as i16,
        tetromino.position.col as i16,
    ) {
        return true;
    }

    false
}
```

We can check if the player has achieved a new high score and allow them to enter their name:

```rust
fn handle_game_over(&mut self, stdout: &mut io::Stdout) -> Result<()> {
    if self.score == 0 {
        self.show_high_scores(stdout)?;
    } else {
        let count: i64 =
            self.conn
                .query_row("SELECT COUNT(*) FROM high_scores", params![], |row| {
                    row.get(0)
                })?;

        if count < 5 {
            self.new_high_score(stdout)?;
        } else {
            let player: Player = self.conn.query_row(
                "SELECT player_name, score FROM high_scores ORDER BY score DESC LIMIT 4,1",
                params![],
                |row| Ok(Player { score: row.get(1)? }),
            )?;

            if (self.score as u64) <= player.score {
                self.show_high_scores(stdout)?;
            } else {
                self.new_high_score(stdout)?;
            }
        }
    }

    Ok(())
}
```

```rust
fn new_high_score(&mut self, stdout: &mut std::io::Stdout) -> Result<()> {
    print_centered_messages(
        stdout,
        None,
        vec![
            "NEW HIGH SCORE!",
            &self.score.to_string(),
            "",
            &format!("{}{}", ENTER_YOUR_NAME_MESSAGE, " ".repeat(MAX_NAME_LENGTH)),
        ],
    )?;

    let mut name = String::new();
    let mut cursor_position: usize = 0;

    let (term_width, term_height) = terminal::size()?;
    stdout.execute(MoveTo(
        (term_width - ENTER_YOUR_NAME_MESSAGE.len() as u16 - MAX_NAME_LENGTH as u16) / 2
            + ENTER_YOUR_NAME_MESSAGE.len() as u16,
        term_height / 2 - 3 / 2 + 2,
    ))?;
    stdout.write(format!("{}", name).as_bytes())?;
    stdout.execute(cursor::Show)?;
    stdout.flush()?;

    loop {
        if poll(Duration::from_millis(10))? {
            let event = read()?;
            match event {
                Event::Key(KeyEvent {
                    code,
                    state: _,
                    kind,
                    modifiers: _,
                }) => {
                    if kind == KeyEventKind::Press {
                        match code {
                            KeyCode::Backspace => {
                                // Handle Backspace key to remove characters.
                                if !name.is_empty() && cursor_position > 0 {
                                    name.remove(cursor_position - 1);
                                    cursor_position -= 1;

                                    stdout.execute(MoveLeft(1))?;
                                    stdout.write(b" ")?;
                                    stdout.flush()?;
                                    print!("{}", &name[cursor_position..]);
                                    stdout.execute(MoveLeft(
                                        name.len() as u16 - cursor_position as u16 + 1,
                                    ))?;
                                    stdout.flush()?;
                                }
                            }
                            KeyCode::Enter => {
                                self.conn.execute(
                                    "INSERT INTO high_scores (player_name, score) VALUES (?1, ?2)",
                                    params![name, self.score],
                                )?;

                                execute!(stdout.lock(), cursor::Hide)?;
                                self.show_high_scores(stdout)?;
                            }
                            KeyCode::Left => {
                                // Move the cursor left.
                                if cursor_position > 0 {
                                    stdout.execute(MoveLeft(1))?;
                                    cursor_position -= 1;
                                }
                            }
                            KeyCode::Right => {
                                // Move the cursor right.
                                if cursor_position < name.len() {
                                    stdout.execute(MoveRight(1))?;
                                    cursor_position += 1;
                                }
                            }
                            KeyCode::Char(c) => {
                                if name.len() < MAX_NAME_LENGTH {
                                    name.insert(cursor_position, c);
                                    cursor_position += 1;
                                    print!("{}", &name[cursor_position - 1..]);
                                    stdout.flush()?;
                                    for _ in cursor_position..name.len() {
                                        stdout.execute(MoveLeft(1))?;
                                    }
                                    stdout.flush()?;
                                }
                            }
                            _ => {}
                        }
                    }
                }
                _ => {}
            }
        }
    }
}
```

#### 8. Pause and Quit {#pause-and-quit}

```rust
if poll(Duration::from_millis(10))? {
    let event = read()?;
    match event {
        Event::Key(KeyEvent {
            code,
            state: _,
            kind,
            modifiers: _,
        }) => {
            if kind == KeyEventKind::Press {
                let mut tetromino = self.current_tetromino.clone();
                match code {
                    KeyCode::Char('p') => {
                        self.paused = !self.paused;
                    }
                    KeyCode::Char('q') => {
                        self.handle_quit_event(stdout)?;
                    }
                    _ => {}
                }
            }
        }
        _ => {}
    }
    self.render_current_tetromino(stdout)?;
}
```

```rust
fn handle_pause_event(&mut self, stdout: &mut io::Stdout) -> Result<()> {
    print_centered_messages(stdout, None, vec!["PAUSED", "", "(C)ontinue | (Q)uit"])?;

    loop {
        if poll(Duration::from_millis(10))? {
            let event = read()?;
            match event {
                Event::Key(KeyEvent {
                    code,
                    modifiers: _,
                    kind,
                    state: _,
                }) => {
                    if kind == KeyEventKind::Press {
                        match code {
                            KeyCode::Enter | KeyCode::Char('c') => {
                                self.render_changed_portions(stdout)?;
                                self.paused = false;
                                break;
                            }
                            KeyCode::Char('q') => {
                                quit(stdout)?;
                            }
                            _ => {}
                        }
                    }
                }
                _ => {}
            }
        }
    }

    Ok(())
}
```

```rust
fn handle_quit_event(&mut self, stdout: &mut io::Stdout) -> Result<()> {
    print_centered_messages(stdout, None, vec!["QUIT?", "", "(Y)es | (N)o"])?;

    loop {
        if poll(Duration::from_millis(10))? {
            let event = read()?;
            match event {
                Event::Key(KeyEvent {
                    code,
                    modifiers: _,
                    kind,
                    state: _,
                }) => {
                    if kind == KeyEventKind::Press {
                        match code {
                            KeyCode::Enter | KeyCode::Char('y') => {
                                quit(stdout)?;
                            }
                            KeyCode::Esc | KeyCode::Char('n') => {
                                self.render_changed_portions(stdout)?;
                                self.paused = false;
                                break;
                            }
                            _ => {}
                        }
                    }
                }
                _ => {}
            }
        }
    }

    Ok(())
}
```

#### 9. Reset {#reset}

After the game is over, we can display the highscores table, and allow the player press `r` to restart:

```rust
fn reset_game(game: &mut Game, stdout: &mut io::Stdout) -> Result<()> {
    game.reset();
    game.render(stdout)?;

    game.handle_event(stdout)?;

    Ok(())
}
```

```rust
fn reset(&mut self) {
    // Reset play grid
    self.play_grid = create_grid(
        PLAY_WIDTH,
        PLAY_HEIGHT,
        self.start_with_number_of_filled_lines,
    );

    // Reset tetrominos
    self.current_tetromino = Tetromino::new(false);
    self.next_tetromino = Tetromino::new(true);

    // Reset game statistics
    self.lines = 0;
    self.level = self.start_at_level;
    self.score = 0;

    let mut drop_interval: u64 = DEFAULT_INTERVAL;
    for _i in 1..=self.start_at_level {
        drop_interval -= drop_interval / 10;
    }
    self.drop_interval = drop_interval;

    // Clear any existing messages in the receiver
    if let Some(ref mut receiver) = self.receiver {
        while let Ok(_) = receiver.try_recv() {}
    }

    // Resume the game
    self.paused = false;
}
```

#### 10. Multiplayer mode {#multiplayer-mode}

To make the game more interesting, I want to add the 2-player mode (so I can play with my son). Whenever a player clears some lines, the number of cleared lines will be sent to the competitor.

To do that, first, we need to open a TCP connection between 2 players and then spawn a new thread to receive messages from the competitor:

```
player 1 <--- TCP stream ---> [receive_message thread <--- mpsc::channel --> main thread] player 2
```

```rust
if args.multiplayer {
    if args.server_address == None {
        let listener = TcpListener::bind("0.0.0.0:8080")?;
        let my_local_ip = local_ip()?;
        println!(
            "Server started. Please invite your competitor to connect to {}.",
            format!("{}:8080", my_local_ip)
        );

        let (stream, _) = listener.accept()?;
        println!("Player 2 connected.");

        let mut stream_clone = stream.try_clone()?;
        let (sender, receiver): (Sender<MessageType>, Receiver<MessageType>) = channel();
        let mut game = Game::new(
            conn,
            Some(stream),
            Some(receiver),
            args.number_of_lines_already_filled,
            args.level,
        )?;

        thread::spawn(move || {
            receive_message(&mut stream_clone, sender);
        });

        game.start()?;
    } else {
        if let Some(server_address) = args.server_address {
            let stream = TcpStream::connect(server_address)?;

            let mut stream_clone = stream.try_clone()?;
            let (sender, receiver): (Sender<MessageType>, Receiver<MessageType>) = channel();
            let mut game = Game::new(
                conn,
                Some(stream),
                Some(receiver),
                number_of_lines_already_filled,
                start_at_level,
            )?;

            thread::spawn(move || {
                receive_message(&mut stream_clone, sender);
            });

            game.start()?;
        }
    }
}
```

When a player receives a message, the sender will forward it to the receiver via the channel:

```rust
fn receive_message(stream: &mut TcpStream, sender: Sender<MessageType>) {
    let mut buffer = [0u8; 256];
    loop {
        match stream.read(&mut buffer) {
            Ok(n) if n > 0 => {
                let msg = String::from_utf8_lossy(&buffer[0..n]);
                if msg.starts_with(PREFIX_CLEARED_ROWS) {
                    if let Ok(rows) = msg.trim_start_matches(PREFIX_CLEARED_ROWS).parse() {
                        if let Err(err) = sender.send(MessageType::ClearedRows(rows)) {
                            eprintln!("Error sending number of cleared rows: {}", err)
                        }
                    }
                } else if msg.starts_with(PREFIX_NOTIFICATION) {
                    let msg = msg.trim_start_matches(PREFIX_NOTIFICATION).to_string();
                    if let Err(err) = sender.send(MessageType::Notification(msg)) {
                        eprintln!("Error sending notification message: {}", err)
                    }
                }
            }
            Ok(_) | Err(_) => {
                break;
            }
        }
    }
}
```

On the receiver side, when a player receive a number of cleared lines, it will add the received number of lines (each with an empty cell in a random column) to the bottom of playfield:

```rust
if let Some(receiver) = &self.receiver {
    for message in receiver.try_iter() {
        match message {
            MessageType::ClearedRows(rows) => {
                let cells =
                    vec![I_CELL, O_CELL, T_CELL, S_CELL, Z_CELL, T_CELL, L_CELL];
                let mut rng = rand::thread_rng();
                let random_cell_index = rng.gen_range(0..cells.len());
                let random_cell = cells[random_cell_index].clone();

                let mut new_row = vec![random_cell; PLAY_WIDTH];
                let random_column = rng.gen_range(0..PLAY_WIDTH);
                new_row[random_column] = EMPTY_CELL;

                for _ in 0..rows {
                    self.play_grid.remove(0);
                    self.play_grid.insert(PLAY_HEIGHT - 1, new_row.clone());
                }

                self.render_play_grid(stdout)?;
            }
```

We also need another component to store the score when playing in 2-player mode:

```rust
struct MultiplayerScore {
    my_score: u8,
    competitor_score: u8,
}

fn handle_game_over(&mut self, stdout: &mut io::Stdout) -> Result<()> {
    if let Some(stream) = &mut self.stream {
        send_message(stream, MessageType::Notification("YOU WIN!".to_string()));
        self.multiplayer_score.competitor_score += 1;

        let stats_start_x = self.start_x - STATS_WIDTH - DISTANCE - 1;
        if let Some(_) = &self.stream {
            execute!(
                stdout,
                SetForegroundColor(Color::White),
                SetBackgroundColor(Color::Black),
                SavePosition,
                MoveTo(
                    stats_start_x as u16 + 2 + "Score: ".len() as u16,
                    self.start_y as u16 + 10
                ),
                Print(format!(
                    "{} - {}",
                    self.multiplayer_score.my_score, self.multiplayer_score.competitor_score
                )),
                ResetColor,
                RestorePosition,
            )?;
        }
    }
```

```rust
if let Some(receiver) = &self.receiver {
    for message in receiver.try_iter() {
        match message {
            MessageType::Notification(msg) => {
                self.paused = !self.paused;

                print_centered_messages(
                    stdout,
                    None,
                    vec![&msg, "", "(R)estart | (C)ontinue | (Q)uit"],
                )?;

                self.multiplayer_score.my_score += 1;

                let stats_start_x = self.start_x - STATS_WIDTH - DISTANCE - 1;
                if let Some(_) = &self.stream {
                    execute!(
                        stdout,
                        SetForegroundColor(Color::White),
                        SetBackgroundColor(Color::Black),
                        SavePosition,
                        MoveTo(
                            stats_start_x as u16 + 2 + "Score: ".len() as u16,
                            self.start_y as u16 + 10
                        ),
                        Print(format!(
                            "{} - {}",
                            self.multiplayer_score.my_score,
                            self.multiplayer_score.competitor_score
                        )),
                        ResetColor,
                        RestorePosition,
                    )?;
                }
```

Throughout [this project](https://github.com/quantonganh/tetris-tui/), I gained invaluable insights, including:

- Crafting a cross platform CLI application with [crossterm](https://docs.rs/crossterm/latest/crossterm/)
- Establishing communication between two machines through [TcpStream](https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html)
- Effectively passing data between threads utilizing [mpsc::channel](https://doc.rust-lang.org/stable/std/sync/mpsc/fn.channel.html)
- ...

PS: I am currently seeking a new remote software engineering opportunity with a focus on backend development. My flexibility extends accommodating time zones within a range of +/- 3 hours from ICT (Indochina Time). If you have any information regarding companies or positions that are actively hiring for such roles, I would greately approciate it if you can kindly leave a comment or get in touch. Your assistance is sincerely valued. Thank you.
