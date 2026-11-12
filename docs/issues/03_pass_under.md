# 机の下などを「くぐれるか」も見てみて

STVL はボクセル（3D）で障害物を持っているので、高さ方向の情報がある。
TB3 burger の背が低いことを活かして、机やベンチの下をくぐれる判定になっているか、実際に試してみてほしい。

見るとよさそうなところ：
- RViz で `/local_costmap/voxel_grid` を表示して、机の天板がどう見えているか
- `nav2_param_scan6.yaml` の stvl_layer の `max_obstacle_height`（いま 2.0m）。ロボットの高さ＋余裕くらいにすると、それより上の天板は無視されてくぐれるようになるはず
- `min_obstacle_height: 0.1`（床の誤検知よけ）とのバランス
- RTAB-Map 側の `Grid/RangeMax: 4.0` など、障害物点群の出し方
