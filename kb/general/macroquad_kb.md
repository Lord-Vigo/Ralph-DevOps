---
# macroquad KB

## Overview
Notes on using macroquad 0.4 for 2D/3D rendering, including the main loop pattern, 3D drawing primitives, 2D overlay, Linux system library requirements, and headless testing strategy.

## Key Information

- **macroquad version:** 0.4 (specified in `Cargo.toml` as `macroquad = "0.4"`)

- **Main loop pattern** — the entry point is an async function decorated with the macroquad attribute:
  ```rust
  use macroquad::prelude::*;

  #[macroquad::main("MyApp")]
  async fn main() {
      loop {
          clear_background(BLACK);
          // draw 3D, then 2D overlay
          next_frame().await;
      }
  }
  ```

- **3D drawing** — set a 3D camera before issuing 3D draw calls, then switch back to 2D for the overlay:
  ```rust
  // Set 3D camera
  set_camera(&Camera3D {
      position: vec3(0.0, 0.0, -100.0),
      target:   vec3(0.0, 0.0, 0.0),
      up:       vec3(0.0, 1.0, 0.0),
      ..Default::default()
  });

  // 3D primitives
  draw_sphere(vec3(x, y, z), radius, None, color);
  draw_line_3d(vec3(x1, y1, z1), vec3(x2, y2, z2), color);

  // Return to 2D for overlay
  set_default_camera();
  ```

- **2D overlay** — standard 2D drawing functions work after `set_default_camera()`:
  ```rust
  draw_rectangle(10.0, 10.0, 200.0, 40.0, Color::new(0.0, 0.0, 0.0, 0.7));
  draw_text("Hello", 15.0, 35.0, 20.0, WHITE);
  ```

- **Linux system library requirements** — macroquad requires the following native libraries to link and run:
  - `libGL` / `libEGL` (Mesa or proprietary)
  - `libX11`, `libXi`, `libXcursor`, `libXrandr` (X11 display)
  - `libxkbcommon`

  Install on Debian/Ubuntu:
  ```bash
  sudo apt install libgl1-mesa-dev libx11-dev libxi-dev \
                   libxcursor-dev libxrandr-dev libxkbcommon-dev
  ```

- **Headless / CI testing** — macroquad cannot render without a display. Isolate rendering logic and test via unit tests that don't invoke draw calls. Use `cargo test --lib` to compile and run only the library crate, avoiding the binary target and linking requirements.

## Commands

```bash
# Build the full binary (requires X11/GL headers)
cargo build

# Run tests WITHOUT linking macroquad (safe for headless environments)
cargo test --lib

# Run the application (requires a display)
./target/release/<app-name>
```

## Coordinate System

macroquad uses a right-handed coordinate system. Common mapping patterns:

```
# For 3D clustering effect:
x = (octet_a - 127.5) * scale
y = (octet_b - 127.5) * scale
z = (octet_c - 127.5) * scale
```

Values in the same /16 or /24 subnet will cluster spatially.

## Common Pitfalls

- **`Camera3D.fovy` is in DEGREES, not radians.** Passing `.to_radians()` converts 60° to ~1.047, which macroquad interprets as ~1 degree — causing extreme zoom. Fix: pass degrees directly.
  ```rust
  // WRONG:
  fovy: self.fov_degrees.to_radians(),
  // CORRECT:
  fovy: self.fov_degrees,  // macroquad Camera3D.fovy is in degrees
  ```

- **Always verify macroquad `Camera3D` defaults** — `Camera3D::default()` uses `fovy: 45.0` (degrees), `position: vec3(0., 10., 0.)`, target at origin, up = Z axis. Ensure your camera constructor sets appropriate values for your coordinate system.

## Color Constants

```rust
use macroquad::prelude::*;

// Named colors
WHITE, BLACK, RED, GREEN, BLUE, YELLOW

// Custom color with alpha
Color::new(r, g, b, a)  // components in range 0.0–1.0
```