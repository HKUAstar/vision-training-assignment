# Vision training assignments

## Detection

Run a supplied armor detector on competition video, solve PnP, and export detections and
3D corners as JSON Lines. Use any language; Python, C++, and Rust are recommended.

Start with the [detection assignment](detection/README.md). The directory includes the model,
two short videos, and camera calibration.

## Aiming

Predict and intercept armor plates on a translating, spinning target using Rust.
The simulator provides delayed target observations, a 3D viewer, and MCAP recording/replay.

Start with the [aiming assignment](aiming/README.md). Implement `aim` in
[aiming/src/lib.rs](aiming/src/lib.rs).

```sh
cargo run --manifest-path aiming/Cargo.toml --locked
```
