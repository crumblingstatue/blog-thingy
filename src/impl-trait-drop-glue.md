# 🦀 The impl trait drop glue effect 🦀

Turning a concrete return type into `impl Trait` might have unintended consequences.

To see how, let's create a music player!

```Rust
/// A nice little music player
#[derive(Default)]
struct Player {
    playlist: Playlist,
}

/// A list of music we can choose from
#[derive(Default)]
struct Playlist {
    items: Vec<String>,
}
```

The player has a method to play a song with the provided playlist index.

```Rust
impl Player {
    /// Play a song with the provided playlist index
    fn play(&mut self, _idx: usize) {
        /* Imagine an implementation here */
    }
}
```

So far, so good. As a celebration, let's play the [🦀 Crab Rave 🦀](<https://www.youtube.com/watch?v=LDU_Txk06tM>)!

```Rust
pub fn play_the_crab_rave() {
    let mut player = Player::default();
    if let Some(pos) = player
        .playlist
        .iter()
        .position(|item| item == "crab-rave.mp3")
    {
        player.play(pos);
    }
}
```

The last piece of the puzzle is implementing `Playlist::iter()`.

```Rust
impl Playlist {
    fn iter(&self) -> std::slice::Iter<'_, String> {
        self.items.iter()
    }
}
```

Perfection. Everything compiles as Ferris intended.

...

But let's say I don't wanna promise that the playlist items are stored as a slice.
There is a common solution to this problem, in the form of `impl Trait`.

Let's redefine our function as

```Rust
fn iter(&self) -> impl Iterator<Item = &'_ String> {
    self.items.iter()
}
```

Now I'm sure everything w-

```Rust
error[E0502]: cannot borrow `player` as mutable because it is also borrowed as immutable
  --> src/lib.rs:27:9
   |
22 |        if let Some(pos) = player
   |   ________________________-
   |  |________________________|
23 | ||         .playlist
   | ||_________________- immutable borrow occurs here
24 | |          .iter()
   | |________________- a temporary with access to the immutable borrow is created here ...
...
27 |            player.play(pos);
   |            ^^^^^^^^^^^^^^^^ mutable borrow occurs here
28 |        }
   |        - ... and the immutable borrow might be used here, when that temporary is dropped and runs the destructor for type `impl Iterator<Item = String>`
```

Oh.

Turns out, by simply returning `impl Iterator`, we lose the information that our returned type does not have [drop glue](#drop-glue),
and as we can see, [drop glue](#drop-glue) interacts with the borrow checker, breaking our code.

Now my question is, is there a way to encode the existence or lack thereof of drop glue with `impl Trait`?

And if there isn't, should there be?

Thanks for reading! 🦀

-----

P.S.: There is a simple way to make the `impl Trait` version compile. We just need to wrap the
expression returning the iterator in a block.

```Rust
if let Some(pos) = {
    /* Hi, this is a block */
    player
        .playlist
        .iter()
        .position(|item| item == "crab-rave.mp3")
} {
    player.play(pos);
}
```

🦀🦀

# Drop glue

For anyone who is confused what drop glue means, it's an automatically generated bit of code that
calls the `Drop` implementation of any field you might have.
You can read more about it at <https://doc.rust-lang.org/std/ops/trait.Drop.html>.

The phase of the borrow checker that deals with drop glue is called
[drop check](<https://doc.rust-lang.org/std/ops/trait.Drop.html#drop-check>).

🦀🦀🦀
