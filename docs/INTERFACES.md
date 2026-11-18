# Interface contract (define and implement together)

Observation: RGB image(s), robot joint state, timestamp, camera intrinsics/extrinsics when available.
Target: object id, confidence, pose/position in named coordinate frame, desired placement pose.
Controller input: target, robot state, timestep; output: simulator-specific action with bounds.
Environment: reset(seed), observe(), step(action), success(), close().
Record units (m, rad, s), coordinate transforms, action convention and failure conditions before integration.
