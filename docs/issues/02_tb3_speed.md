# MPPI の速度上限が TB3 burger の性能を超えている

`nav2_param_scan6.yaml` で `vx_max: 0.5`、`wz_max: 1.9`、velocity_smoother の `max_velocity: [0.5, 0.0, 2.0]` になっている。
TB3 burger の最大は 約 0.22 m/s / 2.84 rad/s なので、`vx_max: 0.22` くらいに合わせると挙動が安定するかも。
