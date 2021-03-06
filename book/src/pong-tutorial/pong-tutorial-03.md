# Moving the paddles

In the previous chapter, we learned about the relationship between entities and
components, and how they represent the "things" in our games. This chapter
introduces Systems - the S in "ECS". Systems are objects that represent
operations over entities, or more specifically, combinations of components.
Let's add a system that moves the paddles based on user input.

A system is nothing more than a function that runs once each frame and
potentially makes some changes to components. If you've used other game
engines, this probably sounds familiar: Unity engine calls these objects
`MonoBehaviour`s and Unreal engine calls them `Actor`s, but these all represent
the same basic idea.

Systems in specs/Amethyst are slightly different. Rather than describe the
behavior of a single instance (eg, a single enemy in your game), they describe
the behavior of all components of a specific type (all enemies). This makes
your code more modular, easier to test, and makes it run faster.

Let's get started.

## Capturing user input

To capture user input, we'll need to introduce a few more files to our game.
Let's start by creating a resource file under the `resources` directory of our
project, called `bindings_config.ron`:

```ron,ignore
(
  axes: {
    "left_paddle": Emulated(pos: Key(W), neg: Key(S)),
    "right_paddle": Emulated(pos: Key(Up), neg: Key(Down)),
  },
  actions: {},
)
```

In Amethyst, inputs can be either scalar inputs (a button that is either
pressed or not), or axes (a range that represents an analog controller stick or
relates two buttons as opposite ends of a range).
In this file, we're creating two axes: W and S will move the
left paddle up and down, and the Up and Down arrow keys will move the right
paddle up and down.

Next, we'll add an input bundle to the game's `Application` object, that
contains an input handler system which captures inputs and maps them to the
axes we defined. Let's make the following changes to `main.rs`.

```rust,no_run,noplaypen
# extern crate amethyst;
# use amethyst::prelude::*;
# use amethyst::core::transform::TransformBundle;
# use amethyst::utils::application_root_dir;
# use amethyst::renderer::{DisplayConfig, DrawFlat, Event, Pipeline,
#                        PosTex, RenderBundle, Stage, VirtualKeyCode};
# macro_rules! env { ($x:expr) => ("") }
# fn main() -> amethyst::Result<()> {
use amethyst::input::InputBundle;

let binding_path = format!(
    "{}/resources/bindings_config.ron",
    application_root_dir()
);

let input_bundle = InputBundle::<String, String>::new()
    .with_bindings_from_file(binding_path)?;

# let path = "./resources/display_config.ron";
# let config = DisplayConfig::load(&path);
# let pipe = Pipeline::build().with_stage(Stage::with_backbuffer()
#       .clear_target([0.0, 0.0, 0.0, 1.0], 1.0)
#       .with_pass(DrawFlat::<PosTex>::new()),
# );
# struct Pong;
# impl<'a, 'b> SimpleState<'a, 'b> for Pong { }
let game_data = GameDataBuilder::default()
    .with_bundle(RenderBundle::new(pipe, Some(config)).with_sprite_sheet_processor())?
    .with_bundle(TransformBundle::new())?
    .with_bundle(input_bundle)?;
let mut game = Application::new("./", Pong, game_data)?;
game.run();
# Ok(())
# }
```

At this point, we're ready to write a system that reads input from the
`InputHandler`, and moves the paddles accordingly. First, we'll create a
directory called `systems` under `src` to hold all our systems. We'll use a
module to collect and export each of our systems to the rest of the
application. Here's our `mod.rs` for `src/systems`:

```rust,ignore
mod paddle;

pub use self::paddle::PaddleSystem;
```

We're finally ready to implement the `PaddleSystem` in `systems/paddle.rs`:

```rust,no_run,noplaypen
# extern crate amethyst;
#
# mod pong {
#     use amethyst::ecs::prelude::*;
#
#     pub enum Side {
#       Left,
#       Right,
#     }
#     pub struct Paddle {
#       pub side: Side,
#     }
#     impl Component for Paddle {
#       type Storage = VecStorage<Self>;
#     }
#
#     pub const ARENA_HEIGHT: f32 = 100.0;
#     pub const PADDLE_HEIGHT: f32 = 16.0;
# }
#
use amethyst::core::Transform;
use amethyst::ecs::{Join, Read, ReadStorage, System, WriteStorage};
use amethyst::input::InputHandler;

// You'll have to mark PADDLE_HEIGHT as public in pong.rs
use pong::{Paddle, Side, ARENA_HEIGHT, PADDLE_HEIGHT};

pub struct PaddleSystem;

impl<'s> System<'s> for PaddleSystem {
  type SystemData = (
    WriteStorage<'s, Transform>,
    ReadStorage<'s, Paddle>,
    Read<'s, InputHandler<String, String>>,
  );

  fn run(&mut self, (mut transforms, paddles, input): Self::SystemData) {
    for (paddle, transform) in (&paddles, &mut transforms).join() {
      let movement = match paddle.side {
        Side::Left => input.axis_value("left_paddle"),
        Side::Right => input.axis_value("right_paddle"),
      };
      if let Some(mv_amount) = movement {
        if mv_amount != 0.0 {
          let side_name = match paddle.side {
            Side::Left => "left",
            Side::Right => "right",
          };
          println!("Side {:?} moving {}", side_name, mv_amount);
        }
      }
    }
  }
}
#
# fn main() {}
```

Now lets add this system to our `GameDataBuilder` in `main.rs`:

```rust,no_run,noplaypen
# extern crate amethyst;
# use amethyst::prelude::*;
# use amethyst::core::transform::TransformBundle;
# use amethyst::renderer::{DisplayConfig, DrawFlat, Pipeline,
#                        PosTex, RenderBundle, Stage};
# fn main() -> amethyst::Result<()> {
# let path = "./resources/display_config.ron";
# let config = DisplayConfig::load(&path);
# let pipe = Pipeline::build().with_stage(Stage::with_backbuffer()
#       .clear_target([0.0, 0.0, 0.0, 1.0], 1.0)
#       .with_pass(DrawFlat::<PosTex>::new()),
# );
# mod systems {
# use amethyst;
# pub struct PaddleSystem;
# impl<'a> amethyst::ecs::System<'a> for PaddleSystem {
# type SystemData = ();
# fn run(&mut self, _: Self::SystemData) { }
# }
# }
# let input_bundle = amethyst::input::InputBundle::<String, String>::new();
// in the run() function
let game_data = GameDataBuilder::default()
    .with_bundle(RenderBundle::new(pipe, Some(config)).with_sprite_sheet_processor())?
    .with_bundle(TransformBundle::new())?
    .with_bundle(input_bundle)?
    .with(systems::PaddleSystem, "paddle_system", &["input_system"]); // Add this line
# Ok(())
# }
```

Don't forget to also add at the top of `main.rs`:

```rust,ignore
mod systems;
```

Take a look at the `with` method call. Here, we're not adding a bundle, we're adding
a system alone. We provide an instance of the system, a string representing its name
and a list of dependencies. The dependencies are the names of the systems that
must be ran before our newly added system. Here, we require the `input_system` to be
ran as we will use the user's input to move the paddles, so we need to have this
data be prepared. The `input_system` key itself is defined in the standard InputBundle.

Back in `paddle.rs`, let's review what our system does, because there's quite a bit there.

We create a unit struct, called `PaddleSystem`, and implement the `System`
trait for it. The trait specifies the lifetime of the components on which it
operates. Inside the implementation, we define the `SystemData` the system
operates on, a tuple of `WriteStorage`, `ReadStorage`, and `Read`. More
specifically, the generic types we've used here tell us that the `PaddleSystem`
mutates `Transform` components, `WriteStorage<'s, Transform>`, it
reads `Paddle` components, `ReadStorage<'s, Paddle>`, and also accesses the
`InputHandler<String, String>` resource we created earlier, using the `Read`
structure.

Then, now that we have access to the storages of the components we want, we can
iterate over them. We perform a join operation between the `Transform` and `Paddle`
storages. This will iterate over all entities that have both a `Paddle` and `Transform`
attached to them, and give us access to the actual components, immutably for the
`Paddle` and mutably for the `Transform`.

> There are many other ways to use storages. For example, you can use them to get
> a reference to the component of a specific type held by an entity, or simply
> iterate over them without joining. However in practice, your most common use will
> be to join over multiple storages as it is rare to have a system affect
> only one specific component.

> Please also note that it is possible to join over storages using multiple threads
> by using `par_join` instead of `join`, but here the overhead introduced is not
> worth the gain offered by parallelism.

## Modifying the transform

If we run the game now, we'll see the console print our keypresses. Let's
make it update the position of the paddle. To do this, we'll modify the y
component of the transform's translation.

```rust,no_run,noplaypen
# extern crate amethyst;
# use amethyst::core::Transform;
# use amethyst::ecs::{Join, Read, ReadStorage, System, WriteStorage};
# use amethyst::input::InputHandler;
# enum Side {
#   Left,
#   Right,
# }
# pub struct Paddle {
#   side: Side,
# }
# impl amethyst::ecs::Component for Paddle {
#   type Storage = amethyst::ecs::VecStorage<Paddle>;
# }
# pub struct PaddleSystem;
# impl<'s> System<'s> for PaddleSystem {
#  type SystemData = (
#    WriteStorage<'s, Transform>,
#    ReadStorage<'s, Paddle>,
#    Read<'s, InputHandler<String, String>>,
#  );
  fn run(&mut self, (mut transforms, paddles, input): Self::SystemData) {
    for (paddle, mut transform) in (&paddles, &mut transforms).join() {
      let movement = match paddle.side {
        Side::Left => input.axis_value("left_paddle"),
        Side::Right => input.axis_value("right_paddle"),
      };
      if let Some(mv_amount) = movement {
        let scaled_amount = 1.2 * mv_amount as f32;
        transform.translate_y(scaled_amount);
      }
    }
  }
# }
```

This is our first attempt at moving the paddles: we take the movement, and
scale it by some factor to make the motion seem smooth. In a real game, we
would use the time elapsed between frames to determine how far to move the
paddle, so that the behavior of the game would not be tied to the game's
framerate. Amethyst provides you with [`amethyst::core::timing::Time`][doc_time]
for that purpose, but for now current approach should suffice.
If you run the game now, you'll notice the paddles are able to "fall" off the edges of the game area.

To fix this, we'll make sure the paddle's anchor point never gets out of the
arena. But as the anchor point is in the middle of the sprite, we also need
to add a margin for the paddle to not go halfway out of the screen.
Therefore, we will border the y value of the transform from
`ARENA_HEIGHT - PADDLE_HEIGHT * 0.5` (the top of the screen but a little bit
lower) to `PADDLE_HEIGHT * 0.5` (the bottom of the screen but a little bit higher).


Our run function should now look something like this:

```rust,no_run,noplaypen
# extern crate amethyst;
# use amethyst::core::Transform;
# use amethyst::ecs::{Join, Read, ReadStorage, System, WriteStorage};
# use amethyst::input::InputHandler;
# const PADDLE_HEIGHT: f32 = 16.0;
# const PADDLE_WIDTH: f32 = 4.0;
# const ARENA_HEIGHT: f32 = 100.0;
# const ARENA_WIDTH: f32 = 100.0;
# enum Side {
#   Left,
#   Right,
# }
# pub struct Paddle {
#   side: Side,
# }
# impl amethyst::ecs::Component for Paddle {
#   type Storage = amethyst::ecs::VecStorage<Paddle>;
# }
# pub struct PaddleSystem;
# impl<'s> System<'s> for PaddleSystem {
#  type SystemData = (
#    WriteStorage<'s, Transform>,
#    ReadStorage<'s, Paddle>,
#    Read<'s, InputHandler<String, String>>,
#  );
  fn run(&mut self, (mut transforms, paddles, input): Self::SystemData) {
    for (paddle, mut transform) in (&paddles, &mut transforms).join() {
      let movement = match paddle.side {
        Side::Left => input.axis_value("left_paddle"),
        Side::Right => input.axis_value("right_paddle"),
      };
      if let Some(mv_amount) = movement {
        let scaled_amount = 1.2 * mv_amount as f32;
        let paddle_y = transform.translation().y;
        transform.set_y(
            (paddle_y + scaled_amount)
                .min(ARENA_HEIGHT - PADDLE_HEIGHT * 0.5)
                .max(PADDLE_HEIGHT * 0.5),
        );
      }
    }
  }
# }
```

## Automatic set up of resources by system.

You might remember, that we had troubles because
amethyst required to set up storage for `Paddle` before
we could use it.

Now that we have a system in place that uses `Paddle` component
we no longer need to manually register it with the world.
Instead, as the `Paddle` is used by our system, the storage will
be set up by it.

```rust,no_run,noplaypen
# extern crate amethyst;
# use amethyst::prelude::*;
# use amethyst::renderer::{TextureHandle, SpriteSheetHandle};
# use amethyst::ecs::World;
# struct Paddle;
# impl amethyst::ecs::Component for Paddle {
#   type Storage = amethyst::ecs::VecStorage<Paddle>;
# }
# fn initialise_paddles(world: &mut World, spritesheet: SpriteSheetHandle) { }
# fn initialise_camera(world: &mut World) { }
# fn load_sprite_sheet(world: &mut World) -> SpriteSheetHandle { unimplemented!() }
# struct MyState;
# impl<'a, 'b> SimpleState<'a, 'b> for MyState {
fn on_start(&mut self, data: StateData<GameData>) {
    let world = data.world;

    // Load the spritesheet necessary to render the graphics.
    let sprite_sheet_handle = load_sprite_sheet(world);

    world.register::<Paddle>(); // <<-- No longer needed

    initialise_paddles(world, sprite_sheet_handle);
    initialise_camera(world);
}
# }
```

## Summary
In this chapter, we added an input handler to our game, so that we
could capture keypresses. We then created a system that would interpret these
keypresses, and move our game's paddles accordingly. In the next chapter, we'll
explore another key concept in real-time games: time. We'll make our game aware
of time, and add a ball for our paddles to bounce back and forth.

[doc_time]: https://www.amethyst.rs/doc/master/doc/amethyst_core/timing/struct.Time.html
