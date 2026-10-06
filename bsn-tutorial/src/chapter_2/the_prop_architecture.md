# The Prop Architecture

_Note: For those of you who come from another game engine such as Unity or Unreal, it will be helpful to compare this approach to prefabs or actor blueprints._

A prop can be defined as an entity that contains an empty shell of pre-defined configurations. In the following examples, we will assume that we are making a game that creates a distinction between static **environment props** and **dynamic props** that can be picked up, dropped, and thrown.

In your editor, create a new folder called `features` as well as a `features.rs` file next to it. Then create a `props` folder inside of it with a `props.rs` file next to it. This is going to be our container for the core structure of this component.

## Define the Basics

Open `props.rs` and begin by defining a `SceneComponent` struct for the runtime entity of the prop.

```rust
#[derive(SceneComponent, Default, Clone)]
pub struct Prop {
    pub id: String,
}
```

By defining this, we can now use `@Prop` when calling a `bsn!` macro. This alone won't be enough for our architecture to be useful though. Now, we need to be able to inject this prop with configuration data. Define a new struct and call this `PropConfig`.

```rust
#[derive(Debug, Clone, Default)]
pub struct PropConfig {
    pub id: String,
    pub name: String,
}
```

Going forward, each new configuration we will create for props is going to be contained in this struct and then loaded into the prop for generation. This configuration data is useful, however it is currently not tied to a Prop, let's go ahead and create the glue. I am going to call this `PropSceneConfig`

```rust
#[derive(Default)]
pub struct PropSceneProperties {
    pub config: PropConfig,
}
```
This glue struct will bind our `Prop` to its `PropConfig`, but we need to make it so `Prop` uses `PropSceneProperties` when it is being constructed in the scene. We do this by adding the glue struct to our runtime entity so it gets used when the `scene()` function is called

```rust
#[derive(SceneComponent, Default, Clone)]
#[scene(PropSceneProperties)]
pub struct Prop {
    pub id: String,
}
```

Finally, we need to implement `Prop` so it will actually build itself in the scene.

```rust
impl Prop {
    fn scene(properties: PropSceneProperties) -> impl Scene {
        let config = properties.config;
        let id = config.id;
        let name = config.name;

        bsn! {
            Prop { id: {id} }
            Name::new(name)
        }
    }
}
```

And that's it! We have now created a Prop that can be configured with a `PropConfig` and built in the scene using the `PropSceneProperties` glue struct.

Next, we will work on adding configuration data to the `PropConfig` in order to give it more functionality.