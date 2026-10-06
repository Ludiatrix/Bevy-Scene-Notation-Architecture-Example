# Adding More Configuration to Props

Last page, we defined a `Prop` and built out it's core structure.

Now, we will add configuration data to the `PropConfig` in order to get actual objects into your scene.

## Creating Injectable Configuration Data

For our game, we want to be able to dynamically fill in data about a prop to generate scenes with `.scene` JSON files. To do this, we will need to create different data structures that own pieces of the prop configuration that can be deserialized later.

The following structs implement different configs that I will be using in future examples. It should be noted not every game is the same and you may not need some of this. Additionally, some data may be able to be swapped to Bevy-native components. This can be done by making sure you add `.register_type::<BevyTypeHere>()` to some part of your initial Bevy construction, such as in `App::new()`.

### Transform
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(default)]
pub struct TransformConfig {
    pub translation: [f32; 3],
    pub rotation_degrees: [f32; 3],
    pub scale: [f32; 3],
}

impl Default for TransformConfig {
    fn default() -> Self {
        Self {
            translation: [0.0, 0.0, 0.0],
            rotation_degrees: [0.0, 0.0, 0.0],
            scale: [1.0, 1.0, 1.0],
        }
    }
}

impl TransformConfig {
    pub fn build(&self) -> Transform {
        let [x, y, z] = self.rotation_degrees.map(f32::to_radians);

        Transform {
            translation: Vec3::from_array(self.translation),
            rotation: Quat::from_euler(EulerRot::YXZ, y, x, z),
            scale: Vec3::from_array(self.scale),
        }
    }
}
```

### Physics and PhysicsBody
```rust
#[derive(Debug, Clone, Copy, Serialize, Deserialize, Default)]
#[serde(rename_all = "snake_case")]
pub enum PhysicsBodyConfig {
    Static,
    #[default]
    Dynamic,
    Kinematic,
}

impl PhysicsBodyConfig {
    fn build(self) -> RigidBody {
        match self {
            Self::Static => RigidBody::Static,
            Self::Dynamic => RigidBody::Dynamic,
            Self::Kinematic => RigidBody::Kinematic,
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(default)]
pub struct PhysicsConfig {
    pub body: PhysicsBodyConfig,
    pub friction: f32,
    pub mass: f32,
}

impl Default for PhysicsConfig {
    fn default() -> Self {
        Self {
            body: PhysicsBodyConfig::Dynamic,
            friction: 0.8,
            mass: 1.0,
        }
    }
}
```

> [!NOTE] 
> The reason we set up the Rigidbody type here is because without it, we'd have to use ``template_value()`` to define the Rigidbody later on instead of here, which is not conducive to our use case.
### Collider
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "shape", rename_all = "snake_case")]
pub enum ColliderConfig {
    Cuboid { size: [f32; 3] },
    Sphere { radius: f32 },
    Capsule { radius: f32, height: f32 },
}

impl Default for ColliderConfig {
    fn default() -> Self {
        Self::Cuboid {
            size: [1.0, 1.0, 1.0],
        }
    }
}

impl ColliderConfig {
    fn build(&self) -> Collider {
        match self {
            Self::Cuboid { size } => {
                let [x, y, z] = *size;
                Collider::cuboid(x, y, z)
            }
            Self::Sphere { radius } => Collider::sphere(*radius),
            Self::Capsule { radius, height } => Collider::capsule(*radius, *height),
        }
    }
}
```

### Rendering and Meterials

```rust
#[derive(Debug, Clone, Copy, Serialize, Deserialize, Default)]
#[serde(rename_all = "snake_case")]
pub enum AlphaModeConfig {
    #[default]
    Opaque,
    Blend,
    Mask,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(default)]
pub struct PbrMaterialConfig {
    pub base_color: ColorConfig,
    pub metallic: f32,
    pub roughness: f32,
    pub albedo_map: Option<String>,
    pub normal_map: Option<String>,
    pub occlusion_map: Option<String>,
    pub height_map: Option<String>,
    pub emissive_map: Option<String>,
    pub emissive_color: ColorConfig,
    pub emissive_intensity: f32,
    pub unlit: bool,
    pub double_sided: bool,
    pub alpha_mode: AlphaModeConfig,
    pub alpha_cutoff: f32,
}

impl Default for PbrMaterialConfig {
    fn default() -> Self {
        Self {
            base_color: ColorConfig::WHITE,
            metallic: 0.0,
            roughness: 0.8,
            albedo_map: None,
            normal_map: None,
            occlusion_map: None,
            height_map: None,
            emissive_map: None,
            emissive_color: ColorConfig::BLACK,
            emissive_intensity: 0.0,
            unlit: false,
            double_sided: false,
            alpha_mode: AlphaModeConfig::Opaque,
            alpha_cutoff: 0.5,
        }
    }
}

impl PbrMaterialConfig {
    fn build(&self) -> StandardMaterial {
        let alpha_mode = match self.alpha_mode {
            AlphaModeConfig::Opaque => AlphaMode::Opaque,
            AlphaModeConfig::Blend => AlphaMode::Blend,
            AlphaModeConfig::Mask => AlphaMode::Mask(self.alpha_cutoff),
        };

        let mut material = StandardMaterial {
            base_color: self.base_color.build(),
            metallic: self.metallic,
            perceptual_roughness: self.roughness,
            emissive: self.emissive_color.build().to_linear() * self.emissive_intensity,
            unlit: self.unlit,
            alpha_mode,
            ..default()
        };

        if self.double_sided {
            material.cull_mode = None;
        }

        material
    }
}
```

With these added to your `props.rs` file, we can now implement these configs to our `PropConfig` and `Prop`.
```rust
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
#[serde(default)]
pub struct PropConfig {
    pub id: String,
    pub name: String,
    pub transform: TransformConfig,
    pub mesh: String,
    pub collider: ColliderConfig,
    pub physics: PhysicsConfig,
    pub material: PbrMaterialConfig,
}

impl Prop {
    fn scene(props: PropSceneProps) -> impl Scene {
        let config = props.config;
        let id = config.id;
        let name = config.name;
        let mesh = config.mesh;
        let transform = config.transform.build();
        let rigid_body = config.physics.body.build();
        let collider = config.collider.build();
        let friction = Friction::new(config.physics.friction);
        let mass = Mass(config.physics.mass.max(f32::EPSILON));
        let material = config.material.build();

        bsn! {
            Prop { id: {id} }

            Name::new(name)
            template_value(transform)
            template_value(rigid_body)
            template_value(collider)
            template_value(friction)
            template_value(mass)

            Mesh3d({mesh})

            MeshMaterial3d<StandardMaterial>(
                asset_value(material)
            )
        }
    }
}
```
Please feel free to take time now and go through each config struct in order to make sure it fits your specific needs.