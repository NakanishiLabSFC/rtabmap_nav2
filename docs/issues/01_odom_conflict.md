# オドメトリが二重に出ている（TB3 の odom と RTAB-Map の visual odom）

## いまの状態
README の手順4（RTAB-Map）の当時のコマンドでは `rgbd_odometry:=false` を付けていたけど、これは `rtabmap.launch.py` に**存在しない引数なので無視されている**。
そのため RTAB-Map の `rgbd_odometry` も起動して、`odom -> base_footprint` の TF を出している（起動ログ `docs/logs/` の `[rgbd_odometry-1] publish_tf = true`）。
TB3 bringup も同じ TF を出すので、**同じ TF を出しているノードが2つある**状態。

## やるとよさそうなこと
TB3 の車輪オドメトリだけを使うなら：
```bash
visual_odometry:=false odom_topic:=/odom
```
を手順4のコマンドに足す。逆に visual odom を使いたいなら、TB3 側の odom TF 配信を止める。
どっちが精度いいか比べてみると面白いかも。
