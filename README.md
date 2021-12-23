# Functional Games

_El jardín de senderos que se bifurcan_ - Jorge Luis Borges 1941
_When you come to a fork in the road, take it_ - often ascribed to Yogi Berra

## Motivation

Functional languages are about immutable state, pure functions and non-strict execution. Immutable state and pure functions with no side effects mean that nothing changes. A function with a given set of arguments must always return the same result and can never have any effect other than to return that result. Non-strict execution means that it is not possible to tell, except by second-guessing the compiler, exactly when any computation will happen.

All these things make functional languages look like the exact opposite of what you want in a computer game. Games are interactive, dynamic and unpredictable. State changes all the time, and the players of the game needs to be in complete control of what happens when.

There is an interesting parallel with the universe we live in. The Newtonian view of a clockwork deterministic universe mirrors the functional model of calculation. Once you know the inputs, the progress of objects is fixed. In fact, from an exact knowledge of the state at any one time, you could calculate the state at any other time. The inputs determine the outputs. There are no side-effects and the order in which you do the calculation is irrelevant.

The way in which the Newtonian view fails is exactly the same as the way a functional computer game would fail -- there is no space for uncertainty, human input, free will. In a sense there is no need for side-effects either. The Newtonian universe has no external observer (except maybe God), so there is no user interface, just inputs leading to outputs.

How would we go about augmenting Newtonian reality to make it reflect a real world containing randomless and people who make independent decisions? Is it possible to do this while maintaining the purity of a mechanistic system that can exactly predict the movements of stars, planets and moons thousands of years into the future?

In the same way, how would we go about designing a computer language that is purely functional, yet allows multiple users with independent inputs, firt-person views onto what is happening, and randomness?

In physics, it is quantum mechanics that destroys the simplistic linear view of time and the block universe where there is only one past and one future and no special time called the present. Quantum mechanics says that anything that can happen will happen; everything that could have happened did happen. In quantum mechanics, we no longer think of a four-dimensional block universe, with a single dimension of time alongside the three dimensions of space. One way to see it is as multiple time dimensions. All of the possible (or at least the probable) paths through space-time satisfy the simple block-universe rules of gravity and mechanics, so the motions of planets are the same in every path. However, when you get to smaller objects, there is an infinity of possible paths. The decay of radioactive atoms or the decisions made in brains allow many possibilities and they all happen. There are many futures and many pasts. A garden of forked paths.

## What is a functional language

Examples in this document are in Haskell, which is a pure non-strict functional language that is designed to look similar to mathematical notation of functions.

## How can a functional language handle user choice?

You can represent a choice of different outcomes or precursors as a list. For example, I can represent a maze node as a list of tunnels emerging from that node. A simple maze game like Zork becomes a list of possible outcomes; an infinite list because there is nothing to stop you from looping, apart from your torch running out of battery so you are eaten by a grue. Infinite lists are okay in functional languages, so long as they are non-strict, but they have to be countably infinite.

When the game involves continuous inputs, for example cursor keys and the spacebar in a game of Asteroids, it is no longer possible to represent the outcomes or precursors as lists, not even infinite lists. Here I assume that the inputs are genuinely continuous. (On a real-world computer, the inputs are actually a choice between discrete sequences of keyboard interrupts, but the number of possibilities is so huge that it is easier to pretend they are continuous.)

We can represent a mapping from a keyboard input to a game outcome as a function. Fortunately for us, functional languages are extremely good at managing functions.

Consider a very simple game with just one rotary controller, so the input to the game at any point is a floating point number. The input is a function of time, so the overall input to the game could be seen as a function of time that returns a float. A game would therefore a function that takes this function of time and returns a gamestate that is also a function of time.

The problem with this approach is that it requires independent synchnonisation of the user input and the gamestate. Clearly it is not possible for the user to appropriately waggle their joystick or turn their rotary controller unless they can see the gamestate, or at least their view of it, at a point synchronised with the time their input is needed.

This technique, of using different functions parameterised by time, where side-effects of different functions are synchronised, is anathema to functional programming. Even in declarative languages, it is unlikely to give the level of synchronisation needed for top-level game play.

We therefore need a completely different paradigm. The idea of user input should be a function that takes a gamestate and returns a user response. Even this is not really sufficient. A game player normally watches the game evolve over time and tailors their response to everything that has happened up to that time, rather than responding only to a snapshot. It is strongly reminiscent of a non-strict evaluation of a list. We get close to what is needed with:

```Haskell
userResponse :: User -> [Gamestate] -> [Response]
```

Note that we are supplying the entire history of gamestate, so the gamestate can be represented by a sequence of deltas. We only need to tell the user about things that are changing. What we mean by change depends on how sophisticated the user is. It might be a list of all sprites that have changed position, or we might only need to show sprites which have changed speed or direction, or even those that have changed for reasons other than standard physics.

One possible issue here would be if the userResponse were defined only by the list of gamestate. This means that an optimised Haskell implementation could use beta reduction to avoid talking to the user at all. If the user tried to play the game a second time, they might find that the gameplay is forced to be exactly the same as they played the first time -- this is after all a function, which must always give the same result from the same input. We avoid this by adding a parameter to represent the user.

## How games usually work

Pretty much all computer games work around what is called _the game loop_. For example, in C++ this might look like:

```C++
while (!WindowShouldClose())    // Detect window close button or ESC key
{
    RespondToPlayerKeys();  // joysticks, mouse presses, keyboard: change global velocities etc
    MoveAllSprites();       // basic physics engine, moving characters, bullets, explosions etc.
    CheckForCollisions();   // Also respond, for example start explosion if bullet hit
    ClearScreen();
    RedrawSprites();        // Cheap and cheerful -- fancier methods exist such as double buffering
}
```

This does not look remotely like functional programming. For example, all communication is via mutable global variables and side effects, so the functions have no arguments and no return values. Even the concept of a loop is alien to functional programming. There is no _while_ or _for_ keyword in Haskell.

## How a functional game could work

```Haskell

-- Given a user and an initial game state, keep running until the game ends, then return the score
gameLoop :: User -> Gamestate -> Int
gameLoop user gs = let
        gameState = gs:(nextState gs response)
        response = userResponse user gameState
        finalState = find endOfGame gamestate
    in
        score finalState

-- Given a gamestate and the most recent user response, calculate the next game state
nextState :: Gamestate -> Response -> Gamestate
nextState gs r = let
        gs1 = respondToPlayerKeys gs r
        gs2 = moveAllSprites gs1
        gs3 = checkForCollisions gs2
    in
        gs3
```

Comparing this with the C++ code above, we can see that the actual game logic is similar, though made more explicit. (To be fair, we could have done this in C++ as well.)

One difference is that in the Haskell version, the code to clear the screen and redraw the sprites is wrapped up in the same userResponse call that fetches the players keystrokes etc.

The other obvious difference is that the Haskell version has no loop. Instead, Haskell has one function, userResponse, which takes a list and returns a list. This function is non-strict, so it can start returning the results before it has seen the whole of the list. Indeed, this is essential to the operation, because the list of gamestates itself depends on the previous response. This is mutual recursion through non-strict list evaluation. It is a standard technique in Haskell.
