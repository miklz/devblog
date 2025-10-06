Here's the output of my engine when I start a new game with the command "ucinewgame", the "print" command is not from the UCI specification, I just added for debugging:
```
ucinewgame
print

12x10 Chess Board:
==============================
10 │ X X X X X X X X X X │
09 │ X X X X X X X X X X │
08 │ X r n b q k b n r X │
07 │ X p p p p p p p p X │
06 │ X . . . . . . . . X │
05 │ X . . . . . . . . X │
04 │ X . . . . . . . . X │
03 │ X . . . . . . . . X │
02 │ X P P P P P P P P X │
01 │ X R N B Q K B N R X │
00 │ X X X X X X X X X X │
-1 │ X X X X X X X X X X │
   └─────────────────────
     z a b c d e f g h i
```


### Piece Sets
With piece sets, each square on the board has a 32 bit value this value represents which pieces are "attacking" that square. How does this works? The value has 16 bits for the white side plus 16 bits for the black side, since we have 32 pieces in a chess board, we can represent them all in this fashion. For example, lets say that only the white queen can go to the h1 square, the bits would be set 
``` mathematica
bit:    0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
White: [K  Q  R  R  B  B  N  N  P  P  P  P  P  P  P  P]
		0  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0

bit:   16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
Black: [K  Q  R  R  B  B  N  N  P  P  P  P  P  P  P  P]
		0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0
```
And the value in the h1 square would be 0b00000000000000000000000000000010 in binary or 0x00000002 in hexadecimal.
As you can see, half of the bits represent the black pieces and half the white pieces, and which bits are set tells us which pieces are attaking that square. The values on each square are also called bitlist, because we have a list of pieces in a bit manner.

## Bitboards



```
sudo sysctl -w kernel.kptr_restrict=0
```

```
sudo sysctl -w kernel.perf_event_paranoid=-1
```

```
cargo flamegraph --dev
```

```
cargo flamegraph --dev --test pawn_tests
```

```
cargo bench --features bench
```

1. Time management
If we want tools to help us catch errors we must be able to communicate with them, that's the reason why a standard communication protocol exists and we must comply with the protocol if we want to write a good chess engine.
## Algorithms


minimax algorithm to search and select the best possible move
alpha-beta pruning to select only the most sensible lines


"Someone has to do the work, the reader, or the writer, the knowledge must be well written and organized."