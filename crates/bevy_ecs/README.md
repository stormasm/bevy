# Bevy ECS

[![License](https://img.shields.io/badge/license-MIT%2FApache-blue.svg)](https://github.com/bevyengine/bevy#license)
[![Crates.io](https://img.shields.io/crates/v/bevy_ecs.svg)](https://crates.io/crates/bevy_ecs)
[![Downloads](https://img.shields.io/crates/d/bevy_ecs.svg)](https://crates.io/crates/bevy_ecs)
[![Docs](https://docs.rs/bevy_ecs/badge.svg)](https://docs.rs/bevy_ecs/latest/bevy_ecs/)
[![Discord](https://img.shields.io/discord/691052431525675048.svg?label=&logo=discord&logoColor=ffffff&color=7389D8&labelColor=6A7EC2)](https://discord.gg/bevy)

## What is Bevy ECS?

Bevy ECS is an Entity Component System custom-built for the [Bevy][bevy] game engine.
It aims to be simple to use, ergonomic, fast, massively parallel, opinionated, and featureful.
It was created specifically for Bevy's needs, but it can easily be used as a standalone crate in other projects.

## ECS

All app logic in Bevy uses the Entity Component System paradigm, which is often shortened to ECS. ECS is a software pattern that involves breaking your program up into Entities, Components, and Systems. Entities are unique "things" that are assigned groups of Components, which are then processed using Systems.

For example, one entity might have a `Position` and `Velocity` component, whereas another entity might have a `Position` and `UI` component. You might have a movement system that runs on all entities with a Position and Velocity component.

The ECS pattern encourages clean, decoupled designs by forcing you to break up your app data and logic into its core components. It also helps make your code faster by optimizing memory access patterns and making parallelism easier.

## Concepts

Bevy ECS is Bevy's implementation of the ECS pattern. Unlike other Rust ECS implementations, which often require complex lifetimes, traits, builder patterns, or macros, Bevy ECS uses normal Rust data types for all of these concepts:

### Components

Components are normal Rust structs. They are data stored in a `World` and specific instances of Components correlate to Entities.

```rust
use bevy_ecs::prelude::*;

#[derive(Component)]
struct Position { x: f32, y: f32 }
```

### Worlds

Entities, Components, and Resources are stored in a `World`. Worlds, much like `std::collections`'s `HashSet` and `Vec`, expose operations to insert, read, write, and remove the data they store.

```rust
use bevy_ecs::world::World;

let world = World::default();
```

### Entities

Entities are unique identifiers that correlate to zero or more Components.

```rust
use bevy_ecs::prelude::*;

#[derive(Component)]
struct Position { x: f32, y: f32 }
#[derive(Component)]
struct Velocity { x: f32, y: f32 }

let mut world = World::new();

let entity = world
    .spawn((Position { x: 0.0, y: 0.0 }, Velocity { x: 1.0, y: 0.0 }))
    .id();

let entity_ref = world.entity(entity);
let position = entity_ref.get::<Position>().unwrap();
let velocity = entity_ref.get::<Velocity>().unwrap();
```

### Systems

Systems are normal Rust functions. Thanks to the Rust type system, Bevy ECS can use function parameter types to determine what data needs to be sent to the system. It also uses this "data access" information to determine what Systems can run in parallel with each other.

```rust
use bevy_ecs::prelude::*;

#[derive(Component)]
struct Position { x: f32, y: f32 }

fn print_position(query: Query<(Entity, &Position)>) {
    for (entity, position) in &query {
        println!("Entity {} is at position: x {}, y {}", entity, position.x, position.y);
    }
}
```

### Resources

Apps often require unique resources, such as asset collections, renderers, audio servers, time, etc. Bevy ECS makes this pattern a first class citizen. `Resource` is a special kind of component that does not belong to any entity. Instead, it is identified uniquely by its type:

```rust
use bevy_ecs::prelude::*;

#[derive(Resource, Default)]
struct Time {
    seconds: f32,
}

let mut world = World::new();

world.insert_resource(Time::default());

let time = world.get_resource::<Time>().unwrap();

// You can also access resources from Systems
fn print_time(time: Res<Time>) {
    println!("{}", time.seconds);
}
```

### Schedules

Schedules run a set of Systems according to some execution strategy.
Systems can be added to any number of System Sets, which are used to control their scheduling metadata.

The built in "parallel executor" considers dependencies between systems and (by default) run as many of them in parallel as possible. This maximizes performance, while keeping the system execution safe. To control the system ordering, define explicit dependencies between systems and their sets.

## Using Bevy ECS

Bevy ECS should feel very natural for those familiar with Rust syntax:

```rust
use bevy_ecs::prelude::*;

#[derive(Component)]
struct Position { x: f32, y: f32 }
#[derive(Component)]
struct Velocity { x: f32, y: f32 }

// This system moves each entity with a Position and Velocity component
fn movement(mut query: Query<(&mut Position, &Velocity)>) {
    for (mut position, velocity) in &mut query {
        position.x += velocity.x;
        position.y += velocity.y;
    }
}

fn main() {
    // Create a new empty World to hold our Entities and Components
    let mut world = World::new();

    // Spawn an entity with Position and Velocity components
    world.spawn((
        Position { x: 0.0, y: 0.0 },
        Velocity { x: 1.0, y: 0.0 },
    ));

    // Create a new Schedule, which defines an execution strategy for Systems
    let mut schedule = Schedule::default();

    // Add our system to the schedule
    schedule.add_systems(movement);

    // Run the schedule once. If your app has a "loop", you would run this once per loop
    schedule.run(&mut world);
}
```

## Features

### Query Filters

```rust
use bevy_ecs::prelude::*;

#[derive(Component)]
struct Position { x: f32, y: f32 }
#[derive(Component)]
struct Player;
#[derive(Component)]
struct Alive;

// Gets the Position component of all Entities with Player component and without the Alive
// component.
fn system(query: Query<&Position, (With<Player>, Without<Alive>)>) {
    for position in &query {
    }
}
```

### Change Detection

Bevy ECS tracks _all_ changes to Components and Resources.

Queries can filter for changed Components:

```rust
use bevy_ecs::prelude::*;

#[derive(Component)]
struct Position { x: f32, y: f32 }
#[derive(Component)]
struct Velocity { x: f32, y: f32 }

// Gets the Position component of all Entities whose Velocity has changed since the last run of the System
fn system_changed(query: Query<&Position, Changed<Velocity>>) {
    for position in &query {
    }
}

// Gets the Position component of all Entities that had a Velocity component added since the last run of the System
fn system_added(query: Query<&Position, Added<Velocity>>) {
    for position in &query {
    }
}
```

Resources also expose change state:

```rust
use bevy_ecs::prelude::*;

#[derive(Resource)]
struct Time(f32);

// Prints "time changed!" if the Time resource has changed since the last run of the System
fn system(time: Res<Time>) {
    if time.is_changed() {
        println!("time changed!");
    }
}
```

### Component Storage

Bevy ECS supports multiple component storage types.

Components can be stored in:

* **Tables**: Fast and cache friendly iteration, but slower adding and removing of components. This is the default storage type.
* **Sparse Sets**: Fast adding and removing of components, but slower iteration.

Component storage types are configurable, and they default to table storage if the storage is not manually defined.

```rust
use bevy_ecs::prelude::*;

#[derive(Component)]
struct TableStoredComponent;

#[derive(Component)]
#[component(storage = "SparseSet")]
struct SparseStoredComponent;
```

### Component Bundles

Define sets of Components that should be added together.

```rust
use bevy_ecs::prelude::*;

#[derive(Default, Component)]
struct Player;
#[derive(Default, Component)]
struct Position { x: f32, y: f32 }
#[derive(Default, Component)]
struct Velocity { x: f32, y: f32 }

#[derive(Bundle, Default)]
struct PlayerBundle {
    player: Player,
    position: Position,
    velocity: Velocity,
}

let mut world = World::new();

// Spawn a new entity and insert the default PlayerBundle
world.spawn(PlayerBundle::default());

// Bundles play well with Rust's struct update syntax
world.spawn(PlayerBundle {
    position: Position { x: 1.0, y: 1.0 },
    ..Default::default()
});
```

### Buffered Events

Buffered events offer a communication channel between one or more systems.
They can be sent using the `EventWriter` system parameter and received with `EventReader`.

```rust
use bevy_ecs::prelude::*;

#[derive(BufferedEvent)]
struct Message(String);

fn writer(mut writer: EventWriter<Message>) {
    writer.write(Message("Hello!".to_string()));
}

fn reader(mut reader: EventReader<Message>) {
    for Message(message) in reader.read() {
        println!("{}", message);
    }
}
```

### Observers

Observers are systems that listen for a "trigger" of a specific `Event`:

```rust
use bevy_ecs::prelude::*;

#[derive(Event)]
struct Speak {
    message: String
}

let mut world = World::new();

world.add_observer(|trigger: On<Speak>| {
    println!("{}", trigger.message);
});

world.flush();

world.trigger(Speak {
    message: "Hello!".to_string(),
});
```

These differ from `EventReader` and `EventWriter` in that they are "reactive".
Rather than happening at a specific point in a schedule, they happen _immediately_ whenever a trigger happens.
Triggers can trigger other triggers, and they all will be evaluated at the same time!

If the event is an `EntityEvent`, it can also be triggered to target specific entities:

```rust
use bevy_ecs::prelude::*;

#[derive(EntityEvent)]
struct Explode;

let mut world = World::new();
let entity = world.spawn_empty().id();

world.add_observer(|trigger: On<Explode>, mut commands: Commands| {
    println!("Entity {} goes BOOM!", trigger.target());
    commands.entity(trigger.target()).despawn();
});

world.flush();

world.trigger_targets(Explode, entity);
```

[bevy]: https://bevy.org/
