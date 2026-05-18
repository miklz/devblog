+++
title = 'Chess Development: Testing Foundation'
date = 2025-09-28T18:07:18-03:00
draft = false
tags = [
	"Continuous Integration",
	"Transposition Table",
]
image = "banner.png"
+++

In this post, I detail how I automated my engine testing and why my first attempt at Transposition Tables actually made my engine 29x slower.
## Why Testing Matters

We need a way to ensure we are constantly improving the engine's strength (its ELO), guaranteeing that any code change actually makes the engine better, not worse.

I already started with a few basic testing like unit test and perft test that validates if moves are being generated correctly. Beyound that, we can benchmark our code to check if it's getting faster or slower. In Rust, there are several crates for benchmarking. I chose **Divan** to profile the code.

The FENs below test the move generator across different game phases: the initial setup, a complex middlegame, and a simplified endgame.

```Rust
#[divan::bench(
	args = [
		("rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1"),
		("rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR b KQkq - 0 1"),
		("r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1"),
		("r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R b KQkq - 0 1"),
		("8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 w - - 0 1"),
		("8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 b - - 0 1"),
		("rnbqkbnr/pp1ppppp/8/2p5/4P3/8/PPPP1PPP/RNBQKBNR w KQkq c6 0 1"),
		("rnbqkbnr/pp1ppppp/8/2p5/4P3/8/PPPP1PPP/RNBQKBNR b KQkq c6 0 1")
	],
)]

fn generate_moves(bencher: Bencher, fen: &str) {
	let mut game = GameState::default();
	game.set_fen_position(fen);
	
	bencher.bench_local(|| {
		game.generate_moves();
	});
}

```

This test times the engine move generator across different positions. Here's the output — our benchmark baseline:

| position                                                             | fastest  | slowest  | median   | mean     | samples | iter |
| -------------------------------------------------------------------- | -------- | -------- | -------- | -------- | ------- | ---- |
| 8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 w - - 0 1                            | 1.613 ms | 1.717 ms | 1.635 ms | 1.641 ms | 100     | 100  |
| r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1 | 239.6 ms | 243.3 ms | 241.9 ms | 241.9 ms | 100     | 100  |
| rnbqkbnr/pp1ppppp/8/2p5/4P3/8/PPPP1PPP/RNBQKBNR w KQkq c6 0 1        | 12.01 ms | 12.6 ms  | 12.09 ms | 12.11 ms | 100     | 100  |
| rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1             | 6.359 ms | 7.409 ms | 6.42 ms  | 6.44 ms  | 100     | 100  |

## Sequential Probability Ratio Test

To really measure if the engines ELO is improving, a different kind of test is necessary: the **Sequential Probability Ratio Test (SPRT)**. The idea is to match two versions of the engine against each other — the old version and the new candidate. If the new version wins more matches, the change had a positive effect on ELO. If it loses more, the change had a negative effect.

There are a few tools that can run an SPRT, `cutechess` and `fastchess` are the most known. Here's how you run an SPRT with `cutechess`:

```
cutechess-cli \
-engine cmd=/path/to/old_engine name="Old Engine" \
-engine cmd=./path/to/new_engine name="New Engine" \
option.Threads=1 -each proto=uci tc=40/60 -rounds 20000 \
-sprt elo0=0 elo1=10 alpha=0.05 beta=0.05 \
-pgnout ./sprt.pgn -ratinginterval 50 -concurrency 4
```

This command test if the new engine is at least 10 ELO stronger, with a 5% chance of false positive or false negative and the max number of games at 20000, but the test ends early if a conclusive result is reached before then.

This seems easy, but it's tedious — we have to run this test every time we implement a new feature or optimization to ensure the change had the desired effect. And soon enough, wrong code will slip through because of the manual labor involved. That's why automated testing is needed.

> "You do not rise to the level of your goals. You fall to the level of your systems" - James Clear

## The Automation Challenge

So, a simple Continuous Integration pipeline would need to:

1. Download cutechess
2. Get the current version of the engine
3. Get the proposed version of the engine
4. Run SPRT test
5. Check if the modification improved, worsened, or kept the engine's ELO the same.

I could run this automation directly on GitHub CI, but the tests take a long time to produce results, and GitHub has monthly limits on CI minutes. So I needed another way to automate these longer tests.

Looking around, I found infrastructures that already implement this kind of testing: OpenBench and FishTest, to name a few. They run matches in a decentralized manner — meaning multiple machines can test the engine at different times, and the tests can run on different machines than the one I use for development. In the future, I could dedicate a Raspberry Pi or another device to test my engine.

 To make use OpenBench at the current stage of my engine, I have to implement the basic requirements:

> - **UCI option for `Hash` and for `Threads`**:  If your engine does not support multi-threading, you may report `option name Threads type spin default 1 min 1 max 1` in your UCI startup. The same could be done for `Hash`.
> - **All engines must be buildable by executing a makefile**: Engines written in languages other than C/C++ are still built by executing a make command, although the makefile might be as simple as executing `cargo`, and passing along an option or two.
> - **All engine makefiles must support passing EXE= to control the output binary name**: OpenBench will execute `make EXE=Engine-ABCDEFGH`. It is critical that the output file is indeed named `Engine-ABCDEFGH`, or `Engine-ABCDEFGH.exe` in the same directory as the makefile
> - **All engines must support being "benched" from the command line**: This must report a final node count, and a final nodes-per second count, and then exit. A simple, acceptable format is `4712710 nodes 1323423 nps`. Refer to `Client/bench.py`'s `parse_bench_output()` function for specifics.

After adding the requirements to support my engine in OpenBench, here's how the integration would ideally would work:
1. Open a Pull Request (PR) in GitHub
2. The Continuous Integration (CI) pipeline creates a new test in OpenBench
3. PR is marked as "pending"
4. When the test ends, OpenBench signals GitHub, and the PR status updates to "success" or "failure"

Unfortunately, OpenBench doesn't have webhooks or APIs that integrate directly with GitHub repository or CI systems. So I had to put in extra work to get this CI working.
## Implementing the Test CI

First, I had to replicate what the OpenBench web page does when creating a new test. I wrote a Python script for this — using the browser's console debugger to capture the correct HTTP requests. The CI invokes this script to create a new test with the baseline branch and the proposed branch.

Since I run my OpenBench instance locally, I used a GitHub self-hosted runner to create tests on my machine. If my OpenBench instance were on the web, this wouldn't be necessary — the GitHub CI could create the test directly. When a test is created successfully, the commit hash and test ID are written to the CI logs, and the PR is marked as pending.

So instead of a human checking the test results, a bot does it. The bot periodically checks for open PRs on GitHub. When it finds one, it looks at the CI logs for a test ID, then checks if that specific test has concluded. If the test has ended, the bot posts a comment with the results and approves or rejects the PR.

To give the bot access to my PRs and CI logs, I had to create a GitHub App with specific permissions. With this App ID, I wrote a Python script that:

- Checks for PRs marked as "pending" for an OpenBench test
- Reads the CI run logs to find if a test was created for that PR
- Polls OpenBench until the test terminates
- Updates the commit status to "success" or "failure" based on the test result

It's a lot of moving pieces. Here's a diagram showing the steps:

![GitHub and OpenBench Integration](GitHub-and-OpenBench-Integration.svg)

Now, I needed a new candidate engine feature to test the CI. I decided to start by implementing transposition tables and move ordering.
## Transposition Tables

Transposition tables are a way to remember what was previously searched, it acts as a cache, storing previously searched board positions to save time. We encode the position as a hash using the Zobrist hashing scheme, then use that hash as an index into the table to check if we have already looked at this position before.

### Zobrist Hashing

In theory, we could use any hashing algorithm to encode a position. However, the algorithm must be extremely fast to avoid adding significant overhead during the tree search. The hashing algorithm tailored for the chess engines is the Zobrist hashing. It works by generating random values for every piece type and color on every square, plus side to move, castling rights,  anden passant files — basically information that uniquely identifies a position. Then it XORs them all together.

We initialize the board by generating random values for:
1. Each piece type and color at each square (64 squares × 12 piece types)
2. Castling rights (white/black kingside/queenside — 4 total)
3. Side to move (white or black)
4. En passant files (8 possible files)

This was not easy for me to understand when I implemented it, so here's a code sample that initializes those values:
``` rust
impl Zobrist {
    /// Generates a new Zobrist structure with cryptographically secure random numbers.
    ///
    /// This should be called once and shared across all board instances for consistency.
    /// Uses `rand::rng()` for random number generation.
    ///
    /// # Performance
    /// - Initialization is O(64×12) = 768 random number generations
    /// - Should be done once at program start
    pub fn new() -> Self {
        let mut rng = rand::rng();
        let mut zobrist = Zobrist {
            pieces: [[0; 12]; 64],
            side_to_move: rng.random(),
            castling_rights: [rng.random(), rng.random(), rng.random(), rng.random()],
            en_passant: [
                rng.random(),
                rng.random(),
                rng.random(),
                rng.random(),
                rng.random(),
                rng.random(),
                rng.random(),
                rng.random(),
            ],
        };

        for square in 0..64 {
            for piece in 0..12 {
                zobrist.pieces[square][piece] = rng.random();
            }
        }
        zobrist
    }
}
```

When making a move, we XOR the piece out of the previous square and XOR it into the new square. We also XOR the side to move — this makes the position unique to the color whose turn it is. To generate a new hash or undo one, we only need (in most cases) 3 XOR operations. That's extremely efficient.

XOR is perfect for this use case because it's reversible: XORing the same value twice returns the original value. That means the same function that hashes a move can also un-hash it when undoing the move.
### Table Entry

I initially tried using a struct to hold the evaluation data in the table, but I realized the size of the struct would limit how many entries the table could hold. So I switched to a 64-bit representation that packs all the evaluation information into a single integer:

```
MSB (Most Significant Bit)                            LSB (Least Significant Bit)
  63         50 49      42 41              26 25 24 23     16 15               0
 +-------------+----------+------------------+-----+---------+------------------+
 |   Reserved  |    Age   |     Best Move    |Type |  Depth  |       Score      |
 |   (14 bits) |  (8 bits)|     (16 bits)    |(2b) | (8 bits)|     (16 bits)    |
 +-------------+----------+------------------+-----+---------+------------------+
        |            |               |          |       |          |
        |            |               |          |       |          +--Score (i16)
        |            |               |          |       +-- Depth (u8)
        |            |               |          +-- Node Type (Exact/Upper/Lower)
        |            |               +-- Best Move (See breakdown below)
        |            +-- Entry Age (for replacement)
        +-- Unused Bits
```

Working with bits directly is error-prone and not ergonomic, so I wrote helper functions to get and set these values without dealing with bit manipulation in the main code:
``` rust
impl TranspositionEntry {
	fn score(data: u64) -> i16 {
		((data & 0xFFFF) as u16) as i16
	}
	
	fn depth(data: u64) -> u8 {
        ((data >> 16) & 0xFF) as u8
    }
```

Oh, and the best move was encoded in a 16-bit format:
```
15          12 11           6 5            0
 +-------------+---------------+--------------+
 |  Promotion  |   To Square   |  From Square |
 |   (4 bits)  |    (6 bits)   |   (6 bits)   |
 +-------------+---------------+--------------+
        |               |               |
        |               |               +-- 0-63 (e.g., E2)
        |               +-- 0-63 (e.g., E4)
        +-- Flags: Queen(0x1), Rook(0x2), Bishop(0x4), Knight(0x8)
```

With this encoding (8 bytes per entry), a 1 GB table can hold 134'217'728 positions. That may sound like a lot, but in the chess world, it's really not that much. Soon enough, the table will have hash collisions, so we need a replacement strategy to evict old entries and keep relevant ones — something to properly implement in the future. But for now, a more immediate problem is pressing.
### Data races

When you have a multi-threaded program reading and writing to the same table, you can encounter data races — one thread writing while another thread reads. This can lead to catastrophic errors (really bad or invalid moves). The most common solution is to use a mutex to prevent concurrent access. So my first attempt at implementing the Transposition Table (TT) used `Arc<RwLock<T>>` — an atomically reference-counted read-write lock — to share the table between multiple threads and prevent data races.

Benchmarking minimax with alpha-beta pruning plus transposition tables gave me this:

| position                                                             | fastest  | slowest  | median   | mean     | samples | iter |
| -------------------------------------------------------------------- | -------- | -------- | -------- | -------- | ------- | ---- |
| 8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 w - - 0 1                            | 178.1 ms | 196.4 ms | 179.7 ms | 180.8 ms | 100     | 100  |
| r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1 | 538.2 ms | 563.4 ms | 540.7 ms | 541.6 ms | 100     | 100  |
| rnbqkbnr/pp1ppppp/8/2p5/4P3/8/PPPP1PPP/RNBQKBNR w KQkq c6 0 1        | 194.1 ms | 199.2 ms | 195.8 ms | 195.8 ms | 100     | 100  |
| rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1             | 185.1 ms | 207.7 ms | 189 ms   | 190.4 ms | 100     | 100  |

**The result was a disaster.** The engine became 2x to 29x slower:

| **Position**   | **Baseline (No TT)** | **With TT (RwLock)** | **Result**      |
| -------------- | -------------------- | -------------------- | --------------- |
| **Startpos**   | 6.42 ms              | 189 ms               | **29x Slower**  |
| **Middlegame** | 241.9 ms             | 540.7 ms             | **2.2x Slower** |

Using **Flamegraphs**, I found the culprit: **Lock Contention.** The search algorithm was spending a significant amount of time just waiting for permission to read or write to the table. Even in a single-threaded environment, the overhead of managing the lock was crushing the performance:

![Transposition Table Flamegraphs](tt-flamegraph.png)

The issue is how read-write lock work. `RwLock` adds atomic operations on every probe, since the search tries to read the table at every position this overhead compounds and slows down the search time instead of helping it.

Since my chess engine only has one thread for search, I switch to a `RefCell`, which is cheaper than `RwLock`. But to build a top competitive engine, parallelism support is a must. There's no free lunch, but maybe I can get cheaper, with another approach.

### Lockless Table

What if we could allow concurrent reads and writes without using locks? The key is a verification step that detects if an entry was corrupted by a simultaneous write.

Here's how it works: When storing a position in the table, we don't just save the evaluation we also save a _verification hash_ that proves this entry belongs to this specific position. This verification hash is computed by XORing the 64-bit Zobrist hash of the position with the 64-bit entry data we're storing.

When reading from the table later, we reverse the process: we take the stored entry, XOR it with the current position's Zobrist hash, and see if the result matches the stored verification hash. If they match, the entry is valid and safe to use. If they don't match (because another thread overwrote part of it mid-read), we simply ignore the entry and recompute from scratch.

This approach turns a data race from a crash into a harmless cache miss. No locks, no waiting, just a cheap XOR operation on every table access. The table now stores two 64-bit values per entry: the evaluation data and the verification hash.

After switching to lockless implementation I got the following times:

| position                                                             | fastest   | slowest  | median   | mean     | samples | iter |
| -------------------------------------------------------------------- | --------- | -------- | -------- | -------- | ------- | ---- |
| 8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 w - - 0 1                            | 6.632 us  | 14.26 ms | 15.36 us | 157.8 us | 100     | 100  |
| r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1 | 14.66 us  | 245.4 ms | 17.1 us  | 2.472 ms | 100     | 100  |
| rnbqkbnr/pp1ppppp/8/2p5/4P3/8/PPPP1PPP/RNBQKBNR w KQkq c6 0 1        | 11.8 us   | 58.96 ms | 12.42 us | 602.1 us | 100     | 100  |
| rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1             | 9.497  us | 29.23 ms | 10.05 us | 302.4 us | 100     | 100  |

The fastest time improved dramatically — by one to two orders of magnitude. However, the slowest times got worse. This shows that when we evaluate a position for the first time, we add overhead by probing the table. But when we encounter the same position again, we speed up the evaluation massively.

| **Position**   | **Baseline** | **Lockless TT** |
| -------------- | ------------ | --------------- |
| **Startpos**   | 6.42 ms      | **10.05 us**    |
| **Middlegame** | 241.9 ms     | **17.1 us**     |
## Move Ordering

If we search the best moves first, we can prune (cut off) large parts of the search tree, speeding up the search massively. But how do we know which moves are more promising?

For now, I'm keeping it simple. My move ordering prioritizes:

1. **The best move from the transposition table** — if we found a winning move in this position before, let's try it first
2. **Captures** — capturing a piece is usually better than moving somewhere random

That's it. No killer heuristics, no history heuristic, no MVV-LVA (Most Valuable Victim - Least Valuable Aggressor) for capture ordering. Just TT move first, then all captures, then everything else.

This is far from optimal. A proper move ordering scheme would sort captures by the value of the captured piece relative to the moving piece, track killer moves that cause beta cutoffs, and maintain move history scores across the search. But implementing those is a project for another day.

For now, even this simple approach gives the search a fighting chance — and it's enough to test my CI pipeline. I'll return to move ordering improvements once the foundation is solid.

## Results

After implementing transposition tables and move ordering, the question is: did it actually get better? I let my bot answer that:

![Bot comment](bot-comment.png)

The engine won 78 games, lost 29, and draw 579 out of 686 total games. The Log-Likelihood Ratio (LLR) confirms that the new engine is in fact better than the previous version.

The engine is officially stronger. And more importantly, I now have the automated foundation to keep building with confidence.
## References

- [Engine Testing Guide](https://dannyhammer.github.io/engine-testing-guide/sprt.html)
- [Lockless Table](https://talkchess.com/viewtopic.php?t=76483)