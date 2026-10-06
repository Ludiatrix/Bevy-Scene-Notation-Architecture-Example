# Creating Global Props

In our project, we have a number of Props that do not fit neatly into our predefined categories. These Props are often used across multiple scenes and have unique configurations. I tend to call these **Global Props**.

For our game, we are going to create two props that we want to have stand out: a directional light called `Sun` to light the scene and a cube that  will be called `Floor`. In your `props` folder, create a new file called `sun.rs` and `floor.rs`. For the sake of demonstration, I will write these in a similar way to `prop.rs` even though we could've probably reused code, for the sake of learning.

In `floor.rs`, add the following:
```rust
use avian3d::prelude::*;
use bevy::prelude::*;
use serde::{Deserialize, Serialize};

use crate::features::props::components::ColorConfig;

#[derive(Default)]
pub struct FloorSceneProps {
    pub config: FloorConfig,
}

#[derive(SceneComponent, Default, Clone)]
#[scene(FloorSceneProps)]
pub struct Floor;

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(default)]
pub struct FloorConfig {
    pub size: [f32; 2],
    pub thickness: f32,
    pub position: [f32; 3],
    pub color: ColorConfig,
    pub friction: f32,
}

impl Default for FloorConfig {
    fn default() -> Self {
        Self {
            size: [40.0, 40.0],
            thickness: 0.2,
            position: [0.0, 0.0, 0.0],
            color: ColorConfig([0.18, 0.42, 0.22, 1.0]),
            friction: 0.8,
        }
    }
}

impl Floor {
    fn scene(props: FloorSceneProps) -> impl Scene {
        let config = props.config;
        let size = Vec3::new(config.size[0], config.thickness, config.size[1]);
        let center = Vec3::from_array(config.position) - Vec3::Y * config.thickness * 0.5;
        let collider = Collider::cuboid(size.x, size.y, size.z);
        let material = StandardMaterial {
            base_color: config.color.build(),
            perceptual_roughness: 0.9,
            ..default()
        };

        bsn! {
            #Floor

            Name::new("Floor")

            Mesh3d(
                asset_value(Cuboid::from_size(size))
            )

            MeshMaterial3d<StandardMaterial>(
                asset_value(material)
            )

            template_value(Transform::from_translation(center))
            template_value(RigidBody::Static)
            template_value(collider)
            template_value(Friction::new(config.friction))
        }
    }
}

```