# Plan of Attack

Table of Contents:

  - [Tools](#Tools)
  - [Tasks](#Tasks)
    - [Basic Game Loop](#Basic-Game-Loop)
    - [REPL](#REPL)
    - [Web](#Web)

## Tools

You will need to have the following to start:

### Python 3

Any version of Python after 3.0 will work. We might as well use the latest, which is either 3.6 or 3.7.

Some more Python-specific tools will be added soon, such as a static type-checker called [Pyro](https://pyre-check.org), and an auto-formatter/linter such as [Black](https://github.com/python/black).

### Git

Git is a [version control system](https://git-scm.com) with a [fairly good tutorial](https://git-scm.com/docs/user-manual.html), though other excellent [tutorials exist](https://www.atlassian.com/git/tutorials/what-is-version-control). Regardless of which you follow, Git is mostly platform agnostic, though different hosting services provide different feature supersets. We'll stick to the basics on [GitHub](https://github.com/alexpopov/Wizard).

I've always used Git from the command-line, but there are also [GUI tools](https://www.sourcetreeapp.com) that many of my [friends used](https://desktop.github.com). I prefer the command-line because that's where I do all my work anyway, but GUI is fine too.

### TBD

Who knows what other tools we'll be using! I suspect at least Django, though that remains to be seen.

## Tasks

General tasks are as follows:

### Basic Game Loop

#### State

Since our game is stateful we need to define a `State` class. This `State` will hold the entirety of the game's state, such as the deck, the players, their hands, and the scores. Good design would mean that the `State` class is actually a relatively dumb container for numerous _other_ types of state, such as those just mentioned, which will each have their _own_ class. This neatly splits up logic and allows us to keep our code smaller.


Somewhat counter-intuitively, we actually want our `State` to be immutable. While what we're modelling is mutable, we will have significantly fewer headaches, particularly down the road, if we make sure nothing in the `State` (and its components) can be mutated. Some of the benefits of this will become clear later, especially once we start dealing with the web portion.

#### Game Logic

This object "runs" the game. Given a `State` and an `Action`, it produces a new `State` or an `Error`.

This leads to a clean little mathematical expression: `(S, A) -> Either[S, E]` which can be looped over infinitely: every time an action is taken we either update the current state with the new one, or we perform no change and report an error (e.g. if a user made an illegal move).

```python
result = run(state, action)
if result.error:
  report_error()
else:
  state = result.new_state
```

As you can see, the game logic is directly dependent on the `State`, but building out the game-logic ASAP also allows us to quickly test if the way we've defined `State` is conducive to our end-goals.
I recommend working on the logic concurrently with the `State`; it may lead to less refactoring down the line.

Of course, this also requires actions...

#### Actions

Actions, or messages, are a discrete set of possible inputs that determine what function(s) need to be applied to our state to produce a new one. They're a neat abstraction that allows us to coordinate and validate the actions being taken on our `State`, while also being able to bundle in a variable number of possible arguments and argument types.

One neat way to think about it is that each `Action` corresponds to a function. Let's say we have two actions: `PlayCard` and `ShuffleDeck`. Our model of `(State, Action) -> State` (omitting `Error` for brevity) is actually equivalent to `<action>(State, ActionArgs...) -> State` where `<action>` is just a function and `ActionArgs...` are a variable number of arguments that are passed with the `Action`.

So instead of calling:

```
run(state, PlayCard)
```

We could instead just call:

```
play_card(state, card_to_play)
```

These are equivalent computationally, but different architecturally.

In the former example, the _caller_ doesn't need to know anything except that this `Action` needs to take-place---the caller does not need to know how to handle the resulting state or any of the possible errors that may arise.
On the logic-side, if we decide that `PlayCard` isn't valid at this point in time, we can just ignore the message, passing the error directly to the user instead of the caller. The logic also doesn't need to know anything about the arguments bundled inside of `PlayCard`; that information is only relevant deeper in the logic.

In the latter code-snippet, we expose a lot more information: the caller now needs to be able to handle all errors and the resulting state (updating it in the main game-loop somehow) and our logic class has a ton of different functions which fundamentally just forward their invocation to a function somewhere else. It's messy.

The other difference is that `PlayCard` has an argument (or two; it's probably important to keep track of which player is playing the card), while `ShuffleDeck` does not; the hypothetical functions `play_card` and `shuffle_deck` have different signatures, whereas our `run` function always has the same signature. This allows us to cleanly have a single interface for many different types.

#### All together

These three major classes work together to run the game and I recommend building them together. Create a `State` which has one property: a `Deck` class. The `Deck` class has two properties: cards still in the deck and cards dealt. Cards are also a class with two properties: value and suit.

We define an abstract base class called `Action` or `GameAction`---it's entirely empty. The reason we do this is because with Pyro we can then statically-check that we're only passing in subclasses of `Action`.

Our first action will be very simple: `ShuffleDeck`. There are no parameters.

Our game logic will receive this action and forward it to something (TBD) which will shuffle the deck, producing a new `Deck` (because every part of `State` needs to be immutable), which will be put into a new `State`, which finally gets returned.

Once this is done, the exact same process is repeated for every single possible action. After all the complexity of building out this first example, the rest is just plugging in more actions and functions into the framework we've built.

### REPL

The game logic outlined above is agnostic to how it is display or how actions are given to it. It could drive an entirely 3D game in virtual reality, or a web interface with simple graphics. It can also drive a textual interface, much like the command-line.

Fundamentally, all of these are the same: a subset of the current state is displayed, there is a way to issue actions, and then the displayed state is updated.

Our first step to actual gameplay, will be a text-based "shell" which accepts user inputs. A REPL is a read-eval-print loop; it _reads_ user input, _evaluates_ it, and _prints_ out the results. For this we need a dead-simple parser, an object which sends parsed actions to the game loop, and a formatter which creates the display of what's going on in the game. Note that all of this is entirely independent of what goes on inside the game logic; the only things we need to know are what actions exist, what the `State` is, and what errors exist.

When this is complete, the game will probably be a "hand-off" style game, since it's limited to a single window on a single machine.

### Web

TBD. This will be built on top of all the other code; instead of the REPL it will create a web-page which essentially just emulates the REPL but has support for multiple players.
