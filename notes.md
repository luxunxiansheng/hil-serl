# Franka Server Architecture

```mermaid

graph TB
    subgraph Main Process
        A[Flask Web Server]
        B[franka_control_api Node]
        C[Gripper Server]
        A --- B
        B --- C
    end
    
    subgraph ROS Core Process
        D[ROS Master]
    end
    
    subgraph Controller Process
        E[cartesian_impedance_controller Node]
        F[franka_state_controller Node]
    end
    
    subgraph Gripper Process
        G[Robotiq Node] --> |If Robotiq|C
        H[Franka Gripper Node] --> |If Franka|C
    end
    
    D --- B
    D --- E
    D --- F
    D --- G
    D --- H

```

## The relationship between gripper server and Franka gripper node

```mermaid
graph TD

    subgraph Impedance Launch Process
        C[franka_gripper_node]
        D[franka_control_node]
    end

    subgraph Main Process
        A[franka_control_api Node]
        B[FrankaGripperServer]
        A --> B
    end
    
    B -->|publishes| E[/franka_gripper/move/goal/]
    B -->|publishes| F[/franka_gripper/grasp/goal/]
    C -->|provides| E
    C -->|provides| F
    C -->|publishes| G[/joint_states/]
    B -->|subscribes| G
    
    style B fill:#e8f5e9,stroke:#1b5e20
    style C fill:#e8f5e9,stroke:#1b5e20

```
------
```mermaid

graph TD
    
    subgraph Main Process ["Main Process (franka_control_api node)"]
        A[Flask Web Server]
        B[FrankaGripperServer<br/><i>client interface</i>]
    end
    

    subgraph Impedance Launch Process
        C[franka_gripper_node<br/><i>hardware control</i>]
    end
    
    B -->|publishes| D[/franka_gripper/move/goal/]
    B -->|publishes| E[/franka_gripper/grasp/goal/]
    B -->|subscribes| F[/joint_states/]
    C -->|provides| D
    C -->|provides| E
    C -->|publishes| F

```










## Example: Moving the robot

### Scenario

```mermaid
sequenceDiagram
    Client->>Flask Server: POST /pose {pose data}
    Flask Server->>FrankaServer: move(pose)
    FrankaServer->>ROS Master: Publish /cartesian_impedance_controller/equilibrium_pose
    ROS Master->>Franka Controller: /cartesian_impedance_controller/equilibrium_pose
    Franka Controller->>Franka Robot: Move to pose
    Franka Robot->>Franka Controller: State update
    Franka Controller->>ROS Master: Publish /franka_state_controller/franka_states
    ROS Master->>FrankaServer: /franka_state_controller/franka_states
    FrankaServer->>Flask Server: Update internal state
    Flask Server->>Client: Response: "Moved"
```


### Nuances and Considerations

1.  **ROS Services**: While topics are suitable for continuous data streams (like state updates), ROS services might be a better choice for specific, request-response interactions, such as setting robot parameters or triggering one-time actions.

2.  **Actionlib**: For long-running or complex actions (like a complete pick-and-place task), ROS's `actionlib` is often preferred. It provides a standardized way to manage goals, provide feedback, and handle cancellation.

3.  **Direct Communication (Sometimes)**: In some cases, direct communication between nodes (without going through the ROS Master for every message) can be more efficient. However, this requires more complex setup and is less common for basic control tasks.

4.  **Error Handling**: The diagram doesn't explicitly show error handling. In a real-world system, it's crucial to include mechanisms for detecting and responding to errors at each stage of the process.

5.  **Security**: Depending on the application, security considerations (authentication, authorization, encryption) might be necessary, especially if the ROS network is exposed to external systems.

### Improved Diagram (Illustrative)

```mermaid
sequenceDiagram
    Client->>Flask Server: POST /pose {pose data}
    Flask Server->>FrankaServer: move(pose)
    FrankaServer->>ROS Master: Publish /cartesian_impedance_controller/equilibrium_pose
    alt Command Accepted
        ROS Master->>Franka Controller: /cartesian_impedance_controller/equilibrium_pose
        Franka Controller->>Franka Robot: Move to pose
        Franka Robot->>Franka Controller: State update
        Franka Controller->>ROS Master: Publish /franka_state_controller/franka_states
        ROS Master->>FrankaServer: /franka_state_controller/franka_states
        FrankaServer->>Flask Server: Update internal state
        Flask Server->>Client: Response: "Moved"
    else Command Rejected
        FrankaServer->>Flask Server: Error: Invalid Pose
        Flask Server->>Client: Error: Invalid Pose
    end
```
