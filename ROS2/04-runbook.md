# GNOME Terminal 启动与操作

每次运行建立独立目录，避免 mav.tlog、mav.parm、terrain 和 SITL 数据散落到家目录。

## Iris

~~~bash
run_dir="$HOME/sim_runs/iris/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$run_dir"
gnome-terminal --title="ROS2 Gazebo ArduPilot - Iris"   --working-directory="$run_dir" -- bash -lc '
    source /opt/ros/jazzy/setup.bash
    source /home/xanter/ros2_ws/install/setup.bash
    export RCUTILS_COLORIZED_OUTPUT=1
    ros2 launch ardupilot_gz_bringup iris_runway.launch.py       rviz:=true use_gz_tf:=true use_instance_dir:=True       2>&1 | tee launch.log
    exec bash
  '
~~~

## Alti Transition 固定翼

~~~bash
run_dir="$HOME/sim_runs/alti/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$run_dir"
gnome-terminal --title="ROS2 Gazebo ArduPilot - Alti Transition"   --working-directory="$run_dir" -- bash -lc '
    source /opt/ros/jazzy/setup.bash
    source /home/xanter/ros2_ws/install/setup.bash
    export RCUTILS_COLORIZED_OUTPUT=1
    ros2 launch ardupilot_gz_bringup       alti_transition_runway.launch.py       rviz:=true use_instance_dir:=True       2>&1 | tee launch.log
    exec bash
  '
~~~

等到模型创建成功、ArduPilotPlugin 接通 JSON、Ready to FLY 和 DDS: Initialization passed 后再控制。

## ROS 检查

~~~bash
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
ros2 topic echo --once /clock
ros2 topic echo --once /ap/status
ros2 topic echo --once /ap/pose/filtered
ros2 service list -t
~~~

## Alti 经 ROS 2 解锁和起飞

Plane 的 GUIDED 模式号为 15。每步确认响应成功：

~~~bash
ros2 service list -t | grep '^/ap/'
ros2 service call /ap/prearm_check std_srvs/srv/Trigger
ros2 service call /ap/mode_switch   ardupilot_msgs/srv/ModeSwitch "{mode: 15}"
ros2 service call /ap/arm_motors   ardupilot_msgs/srv/ArmMotors "{arm: true}"
ros2 service call /ap/experimental/takeoff   ardupilot_msgs/srv/Takeoff "{alt: 20.0}"
~~~

模式号随载具类型变化，15 只指这里的 ArduPlane GUIDED。

## Alti 经 MAVProxy 解锁和起飞

集成启动将 MAVLink 转发到 127.0.0.1:14551：

~~~bash
control_dir="$HOME/sim_runs/mavproxy-control/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$control_dir"
gnome-terminal --title="MAVProxy - Alti Control"   --working-directory="$control_dir" -- bash -lc '
    source /home/xanter/venv-ardupilot/bin/activate
    mavproxy.py --master=udp:127.0.0.1:14551
    exec bash
  '
~~~

MAVProxy 中输入：

~~~text
mode GUIDED
arm throttle
takeoff 20
~~~

这是本机实际验证成功的 SITL 流程。解锁前确认连接的是仿真实例。

## 停止

在 launch 的 GNOME Terminal 中按 Ctrl+C，让信号发给前台进程组。等待 Gazebo、SITL、Agent、bridge、RViz 退出，然后检查：

~~~bash
ps -eo pid,comm,args |   grep -E 'gz sim|arducopter|arduplane|micro_ros_agent|parameter_bridge|rviz2' |   grep -v grep
~~~

MAVProxy 会在当前目录写 mav.tlog、mav.tlog.raw、mav.parm；use_instance_dir:=True 会将 eeprom.bin、BIN 日志和 terrain 放入实例目录。
