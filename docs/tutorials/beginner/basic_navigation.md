# From Move Base to Move Base Flex
## Run tutorial

Clone [mbf_tutorials](https://github.com/uos/mbf_tutorials) 
```bash
git clone git@github.com:uos/mbf_tutorials.git ~/catkin_ws/src/mbf_tutorials
```
Install dependencies with rosdep
```bash
rosdep install turtlebot3_mbf
```
From source of workspace: 
```bash
cd ~/catkin_ws/src
catkin_make -j4 && source devel/setup.bash`
```
Define the Turlebot model you want to use:
```bash
export TURTLEBOT3_MODEL=burger
```
Launch the appropriate gazebo world
```bash
roslaunch turtlebot3_gazebo turtlebot3_world.launch
```
Start localization (AMCL)
```bash
roslaunch turtlebot3_mbf amcl_demo_relay_subscriber.launch
```

Launch Rviz
```bash
roslaunch turtlebot3_mbf rviz.launch
```

In RViz

* Set an Initial Pose estimate with the `2D Pose Estimate` Pose
* Finally set your Navigation Goal with the `2D Nav Goal` Pose

You will now be able to navigate in a similar fashion to this:

![](../../img/demo.gif)


<br>

## What is happening here? 

We used Move Base Flex by relaying `mb_msgs/MoveBaseAction` to `mbf_msgs/MoveBaseAction`. 

### Code

```python
import actionlib
import rospy
import nav_msgs.srv as nav_srvs
import mbf_msgs.msg as mbf_msgs
import move_base_msgs.msg as mb_msgs
from geometry_msgs.msg import PoseStamped

def simple_goal_cb(msg):
    mbf_mb_ac.send_goal(mbf_msgs.MoveBaseGoal(target_pose=msg))
    rospy.logdebug("Relaying move_base_simple/goal pose to mbf")

    mbf_mb_ac.wait_for_result()

    status = mbf_mb_ac.get_state()
    result = mbf_mb_ac.get_result()

    rospy.logdebug("MBF execution completed with result [%d]: %s", result.outcome, result.message)

if __name__ == '__main__':
    rospy.init_node("move_base_relay")

    # move base flex ation client relays incoming mb goals to mbf
    mbf_mb_ac = actionlib.SimpleActionClient("move_base_flex/move_base", mbf_msgs.MoveBaseAction)
    mbf_mb_ac.wait_for_server(rospy.Duration(20))

    # move_base simple topic and action server
    mb_sg = rospy.Subscriber('move_base_simple/goal', PoseStamped, simple_goal_cb)

    rospy.spin()
```


### The Code Explained
    

MoveBase subscriber to handle goal events

```python
mb_sg = rospy.Subscriber('move_base_simple/goal', PoseStamped, simple_goal_cb)
```

MoveBase expects goal Messages (`geometry_msgs/Pose`) on topic `move_base_simple/goal`. The subscriber callback `simple_goal_cb` handles the `mbf_msgs.MoveBaseAction` ROS Action Client. The Move Base Flex SimpleActionServer is launched from within Move Base Flex.

```python
mbf_mb_ac = actionlib.SimpleActionClient("move_base_flex/move_base", mbf_msgs.MoveBaseAction)
...

def simple_goal_cb(msg):
    mbf_mb_ac.send_goal(mbf_msgs.MoveBaseGoal(target_pose=msg))
```
and relays the MoveBaseAction to the Move Base Flex action client!

At this stage, we are using the global planner and local planner defined in [move_base.yml](https://github.com/uos/mbf_tutorials/blob/master/beginner/param/move_base.yaml).


## A Relay with more control

While the first example allows you to relay messages to Move Base Flex, the only way to reach goals is by setting a 2D Navigation Goal via RViz, which can be limiting. This examples allows you to send goals directly from a ROS node.

### Code

=== "Server"

    ```python
    import actionlib
    import rospy
    import mbf_msgs.msg as mbf_msgs
    import move_base_msgs.msg as mb_msgs

    def mb_execute_cb(msg):
        mbf_mb_ac.send_goal(mbf_msgs.MoveBaseGoal(target_pose=msg.target_pose),
                            feedback_cb=mbf_feedback_cb)

        rospy.logdebug("Relaying move_base goal to mbf")
        mbf_mb_ac.wait_for_result()

        status = mbf_mb_ac.get_state()
        result = mbf_mb_ac.get_result()

        rospy.logdebug("MBF execution completed with result [%d]: %s", result.outcome, result.message)
        if result.outcome == mbf_msgs.MoveBaseResult.SUCCESS:
            mb_as.set_succeeded(mb_msgs.MoveBaseResult(), "Goal reached.")
        else:
            mb_as.set_aborted(mb_msgs.MoveBaseResult(), result.message)

    def mbf_feedback_cb(feedback):
        mb_as.publish_feedback(mb_msgs.MoveBaseFeedback(base_position=feedback.current_pose))

    if __name__ == '__main__':
        rospy.init_node("move_base")

        # move_base_flex get_path and move_base action clients
        mbf_mb_ac = actionlib.SimpleActionClient("move_base_flex/move_base", mbf_msgs.MoveBaseAction)
        mbf_mb_ac.wait_for_server(rospy.Duration(10))

        mb_as = actionlib.SimpleActionServer('move_base', mb_msgs.MoveBaseAction, mb_execute_cb, auto_start=False)
        mb_as.start()

        rospy.spin()
    ```

=== "Client"

    ```python
    import rospy
    import actionlib
    import mbf_msgs.msg as mbf_msgs
    import move_base_msgs.msg as mb_msgs

    def mb_relay_client():
        client = actionlib.SimpleActionClient('move_base', mb_msgs.MoveBaseAction)
        client.wait_for_server(rospy.Duration(10))

        rospy.loginfo("Connected to SimpleActionServer 'move_base'")

        goal = mb_msgs.MoveBaseGoal()
        goal.target_pose.header.frame_id = "map"
        goal.target_pose.header.stamp = rospy.Time.now()
        goal.target_pose.pose.position.x = -1.990
        goal.target_pose.pose.position.y = -0.508
        goal.target_pose.pose.orientation.z = -0.112
        goal.target_pose.pose.orientation.w = 1.0

        client.send_goal(goal)
        client.wait_for_result()

        return client.get_result() 

    if __name__ == '__main__':
        try:
            rospy.init_node('mb_relay_client')
            result = mb_relay_client()
            rospy.loginfo("MBF get_path execution completed with result [%d]: %s", result.outcome, result.message)

        except rospy.ROSInterruptException:
            rospy.logerror("program interrupted before completion")
    ```

### The Code Explained

On the server side, we start a standard Move Base Action Server, and connect a Move Base Flex Action Client to the default Move Base Flex Action Server.

```python
mbf_mb_ac = actionlib.SimpleActionClient("move_base_flex/move_base", mbf_msgs.MoveBaseAction)
mbf_mb_ac.wait_for_server(rospy.Duration(10))

mb_as = actionlib.SimpleActionServer('move_base', mb_msgs.MoveBaseAction, mb_execute_cb, auto_start=False)
```

We then relay the goal in the callback of the Move Base Action Server, like in the first subriber callback example

```python
def mb_execute_cb(msg):
    mbf_mb_ac.send_goal(mbf_msgs.MoveBaseGoal(target_pose=msg.target_pose),
                        feedback_cb=mbf_feedback_cb)
```

On the client side, we simply connect to the Move Base Action Server, and send a goal, which is then relayed in the above function.

```python        
client = actionlib.SimpleActionClient('move_base', mb_msgs.MoveBaseAction)
client.wait_for_server(rospy.Duration(10))

rospy.loginfo("Connected to SimpleActionServer 'move_base'")

goal = mb_msgs.MoveBaseGoal()
goal.target_pose.header.frame_id = "map"
goal.target_pose.header.stamp = rospy.Time.now()
goal.target_pose.pose.position.x = -1.990
goal.target_pose.pose.position.y = -0.508
goal.target_pose.pose.orientation.z = -0.112
goal.target_pose.pose.orientation.w = 1.0

```

The full source code can be found [here](https://github.com/uos/mbf_tutorials/tree/master/beginner).
