---
title: Integration testing TUI applications in Rust
date: 2024-01-21
description:
categories:
- Programming
tags:
- integration-testing
- rust

---
In building games with any language, there will be a loop to handle the key events. In case of [crossterm](https://github.com/crossterm-rs/crossterm), it's [event::read](https://docs.rs/crossterm/latest/crossterm/event/fn.read.html):


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
```

To make this code testable, we can add a [Trait](https://doc.rust-lang.org/book/ch10-02-traits.html):

```rust
pub trait Terminal {
    fn poll_event(&self, duration: Duration) -> Result<bool>;
    fn read_event(&self) -> Result<Event>;
}
```

and implement it for the real terminal:

```rust
pub struct RealTerminal;

impl Terminal for RealTerminal {
    fn poll_event(&self, duration: Duration) -> Result<bool> {
        Ok(poll(duration)?)
    }

    fn read_event(&self) -> Result<Event> {
        Ok(read()?)
    }
}
```

Now the `Game` struct can be modified to:

```rust
pub struct Game {
    terminal: Box<dyn Terminal>,
}
```

```rust
if self.terminal.poll_event(Duration::from_millis(10))? {
    if let Ok(event) = self.terminal.read_event() {
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
```

When writing integration test, we can create `MockTerminal` struct with a field to mock the key code:

```rust
struct MockTerminal {
    mock_key_code: Option<Receiver<KeyCode>>,
}

impl Terminal for MockTerminal {
    fn poll_event(&self, duration: Duration) -> Result<bool> {
        thread::sleep(duration);
        Ok(true)
    }

    fn read_event(&self) -> Result<Event> {
        if let Some(mock_key_code) = &self.mock_key_code {
            if let Ok(code) = mock_key_code.recv() {
                println!("Received: {:?}", code);
                return Ok(Event::Key(KeyEvent {
                    code,
                    modifiers: KeyModifiers::empty(),
                    kind: KeyEventKind::Press,
                    state: KeyEventState::empty(),
                }));
            }
        }

        Ok(Event::Key(KeyEvent {
            code: KeyCode::Null,
            modifiers: KeyModifiers::empty(),
            kind: KeyEventKind::em
            state: KeyEventState::empty(),
        }))
    }
```

Here, we use [mpsc::channel](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html) to drive the main thread. On the receiver side, we wait for a value and return the corresponding [KeyEvent](https://docs.rs/crossterm/latest/crossterm/event/struct.KeyEvent.html).

```rust
#[test]
fn clear_lines() -> Result<()> {
    let (tx, rx): (Sender<KeyCode>, Receiver<KeyCode>) = channel();
    let mut game = Game::new(
        Box::new(MockTerminal::new(Some(rx))),
        tetromino_spawner,
        sqlite_highscore_repository,
        40,
        20,
        0,
        0,
        None,
        None,
        Some(play_grid_tx),
    )?;

    let receiver = thread::spawn(move || {
        game.start().unwrap();
    });

    let previous_col = game.current_tetromino.position.col;
    tx.send(KeyCode::Char('h')).unwrap();
    assert_eq!(game.current_tetromino.position.col, previous_col - 1);
```

Upon running `cargo test`, I encountered an error:

```
error[E0382]: borrow of moved value: `game`
   --> tests/integration_test.rs:146:24
    |
129 |     let mut game = Game::new(
    |         -------- move occurs because `game` has type `Game`, which does not implement the `Copy` trait
...
142 |     let receiver = thread::spawn(move || {
    |                                  ------- value moved into closure here
143 |         game.start().unwrap();
    |         ---- variable moved due to use in closure
...
146 |     let previous_col = game.current_tetromino.position.col;
    |                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ value borrowed here after move
```

As indicated by the compiler, since Game is started in a separate thread, its value cannot be borrowed after move.
To address this, I tried using `Arc<Mutex<T>>` to share Game state between multiple threads:

```rust
let game = Arc::new(Mutex::new(Game::new(
    Box::new(MockTerminal::new(Some(rx))),
    tetromino_spawner,
    sqlite_highscore_repository,
    40,
    20,
    0,
    0,
    None,
    None,
    None,
)?));

let receiver = thread::spawn({
    let game = Arc::clone(&game);
    move || {
        let mut game_lock = game.lock().unwrap();
        game_lock.start().unwrap();
    }
});

let game_lock = game.lock().unwrap();
let previous_col = game_lock.current_tetromino.position.col;
tx.send(KeyCode::Char('h')).unwrap();
assert_eq!(game_lock.current_tetromino.position.col, previous_col - 1);

receiver.join().unwrap();

Ok(())
```

Upon re-running `cargo test -- --nocapture`, I got another error:

```
Running tests/integration_test.rs (target/debug/deps/integration_test-81597c4542f1a01f)

running 1 test
thread 'clear_lines' panicked at tests/integration_test.rs:155:5:
assertion `left == right` failed
  left: 3
 right: 2
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
thread '<unnamed>' panicked at tests/integration_test.rs:147:45:
called `Result::unwrap()` on an `Err` value: PoisonError { .. }
test clear_lines ... FAILED

failures:

failures:
    clear_lines

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

However, I realized that the assertion is performed before the game thread is completed, resulting in the tetromino position column not being updated. Furthermore, if I send `q` (quit), then `y` (confirm) to terminate the game thread, then there is... nothing to assert. This situation has left me in a dilemma.

After [seeking advice on the Rust forum](https://users.rust-lang.org/t/simulate-key-presses-to-test-the-game-loop/105428) and receiving guidance from [parasyte](https://users.rust-lang.org/t/simulate-key-presses-to-test-the-game-loop/105428/10):

> What I meant with the "return channel" was literally passing the state back through the channel for assertions. It's very easy to do if you don't mind cloning the state for your tests.

I decided to pass the play grid state back through the channel for assertions after the tetromino reaches the bottom:

```rust
pub struct Game {
    terminal: Box<dyn Terminal + Send>,
    ...
    // This is only used for integration testing purposes
    state_sender: Option<Sender<Vec<Vec<Cell>>>>,
}

impl Game {
    fn lock_and_move_to_next(
        &mut self,
        tetromino: &Tetromino,
        stdout: &mut io::Stdout,
    ) -> Result<()> {
        self.lock_tetromino(tetromino)?;

        // When performing integration testing, Game instance is started in a spawned thread
        // This sends the play grid state to the main thread, so it can be asserted.
        if let Some(state_sender) = &self.state_sender {
            state_sender.send(self.play_grid.clone())?;
        }

        self.move_to_next()?;

        if self.is_game_over() {
            self.handle_game_over(stdout)?;
        }

        Ok(())
    }
```

```rust
#[test]
fn clear_lines() -> Result<()> {
    let tetromino_spawner = Box::new(ITetromino);
    let conn = Connection::open_in_memory()?;
    let sqlite_highscore_repository = Box::new(HighScoreRepo { conn });

    let (tx, rx): (Sender<KeyCode>, Receiver<KeyCode>) = channel();
    let (play_grid_tx, play_grid_rx): (Sender<Vec<Vec<Cell>>>, Receiver<Vec<Vec<Cell>>>) =
        channel();
    let mut game = Game::new(
        Box::new(MockTerminal::new(Some(rx))),
        tetromino_spawner,
        sqlite_highscore_repository,
        40,
        20,
        0,
        0,
        None,
        None,
        Some(play_grid_tx),
    )?;

    let receiver = thread::spawn(move || {
        game.start().unwrap();
    });

    // Clear a line by placing 4 I tetrominoes like this ____||____
    // Move the first I tetromino to the left border
    tx.send(KeyCode::Char('h')).unwrap();
    tx.send(KeyCode::Char('h')).unwrap();
    tx.send(KeyCode::Char('h')).unwrap();
    tx.send(KeyCode::Char('j')).unwrap();
    if let Ok(play_grid) = play_grid_rx.recv() {
        for col in 0..4 {
            assert_eq!(play_grid[19][col], I_CELL);
        }
    }

    // // Move the 2nd I tetromino to the right border
    tx.send(KeyCode::Char('l')).unwrap();
    tx.send(KeyCode::Char('l')).unwrap();
    tx.send(KeyCode::Char('l')).unwrap();
    tx.send(KeyCode::Char('j')).unwrap();
    if let Ok(play_grid) = play_grid_rx.recv() {
        for col in 6..10 {
            assert_eq!(play_grid[19][col], I_CELL);
        }
    }

    // Rotate the 3rd I tetromino, move left one column, then hard drop
    tx.send(KeyCode::Char(' ')).unwrap();
    tx.send(KeyCode::Char('h')).unwrap();
    tx.send(KeyCode::Char('j')).unwrap();
    if let Ok(play_grid) = play_grid_rx.recv() {
        for row in 16..20 {
            assert_eq!(play_grid[row][4], I_CELL);
        }
    }

    // Rotate the 4th I tetromino, then hard drop to fill a line
    tx.send(KeyCode::Char(' ')).unwrap();
    tx.send(KeyCode::Char('j')).unwrap();
    if let Ok(play_grid) = play_grid_rx.recv() {
        for col in 0..4 {
            assert_eq!(play_grid[19][col], EMPTY_CELL);
        }
        for col in 6..10 {
            assert_eq!(play_grid[19][col], EMPTY_CELL);
        }
        for row in 18..20 {
            for col in 4..6 {
                assert_eq!(play_grid[row][col], I_CELL);
            }
        }
    }

    tx.send(KeyCode::Char('q')).unwrap();
    tx.send(KeyCode::Char('y')).unwrap();

    receiver.join().unwrap();

    Ok(())
}
```

and `cargo test -- --nocapture` worked like a charm:

```
     Running tests/integration_test.rs (target/debug/deps/integration_test-81597c4542f1a01f)

running 1 test
Received: Char('h')
Received: Char('h')
Received: Char('h')
Received: Char('j')
Received: Char('l')
Received: Char('l')
Received: Char('l')
Received: Char('j')
Received: Char(' ')
Received: Char('h')
Received: Char('j')
Received: Char(' ')
Received: Char('j')
Received: Char('q')
Received: Char('y')
```