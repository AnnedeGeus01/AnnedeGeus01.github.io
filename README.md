# AnnedeGeus01.github.io

## How to set up and use Jolt in an entity component system (c++) fo beginners.


## steps to take:
1. Look at the [API of Jolt Physics](https://jrouwe.github.io/JoltPhysics/) and look at their hello world example in the JoltPhysics-Master that you can download on [their github.](https://www.google.com)
2. Including the library in your project.
3. Setting up the needed classes.
4. Initializing the Physics world.
5. Creating bodies.
6. What to put in the update loop.

### 2. Including the library in your project:
#### To include the Jolt library into your project you need to take a couple of steps:
- Firstly download the JoltPhysics-Master from their github, and copy the Jolt folder and put it in your project in an appropriate place, like in a folder called external etc.
- Then in the properties settings of your project you want to go C/C++-> General-> Additional Include Directories and add **$(ProjectDir)external\joltPhysics** in both debug and release.
- In the JoltPhysics-Master files you want to go JoltPhysics-Master -> UnitTest and get the Layers.h file, and add it to your project as well. I recommend putting this files with your own made classes since you will need to change something in the class. You need to include **#include <Jolt/Jolt.h>** at the top of the other includes. With this done, and if you have looked at the hello world example, this layers.h file sets up all of the layers for you, that in the hello world were made in the main itself, this file gives for cleaner and simpler code.
Now you are all set up for the Jolt Physics library, and can use it in your project!

### Setting up the needed classes:
ECS, or Entity-Component-System, is a design pattern where entities are composed of components, and systems process entities based on their components. In this case, the `PhysicsSetup` class represents the system responsible for handling physics, and the `JoltBody` class is a component representing a physics body. I put them under the same namingspace, so it is clear to see it belongs togeher, but this is up for personal preference.

### 4. Initializing the Physics world, in the PhysicsSetup class.
```c++
class PhysicsSetup : public System
{
public:
    PhysicsSetup();
    ~PhysicsSetup();

    void Initialize(bool gravity);
    void Update(float deltaTime);
    void createBox(entity entityID, vec3 pos, vec3 size, bool isDynamic);
    void createSphere(eentity entityID, vec3 pos, float radius, bool isDynamic);
    void createConvexHull(entity entityID, vec3 pos, vec3 size, const bee::MeshRenderer& meshRenderer, bool isDynamic);

    BodyInterface* mBodyInterface = nullptr;                           // The physics system that simulates the world

private:

    // Jolt settings
    int mMaxConcurrentJobs = thread::hardware_concurrency();           // How many jobs to run in parallel
    float mUpdateFrequency = 60.0f;                                    // Physics update frequency
    int mCollisionSteps = 1;                                           // How many collision detection steps per physics update
    TempAllocator* mTempAllocator = nullptr;                           // Allocator for temporary allocations
    JobSystem* mJobSystem = nullptr;                                   // The job system that runs physics jobs
    BPLayerInterfaceImpl mBroadPhaseLayerInterface;                    // The broadphase layer interface that maps object layers to broadphase layers
    ObjectVsBroadPhaseLayerFilterImpl mObjectVsBroadPhaseLayerFilter;  // Class that filters object vs broadphase layers
    ObjectLayerPairFilterImpl mObjectVsObjectLayerFilter;              // Class that filters object vs object layers
    PhysicsSystem* mPhysicsSystem = nullptr;                           // The physics system that simulates the world
};
```
- Represents the system responsible for handling physics. Inherits from *System*, in my case, but depending on what ECS you are working with, you need to derive from the appropriate system.
- The *Initialize* function is responsible for setting up the Jolt physics system. It takes a Boolean parameter *gravity* to determine whether gravity is enabled. In the function it sets up things like the gravity, if it is on, and it sets up limitations, like the temporary allocator, the maximum amount of bodies, how many mutexes (a mutex is like a lock used in computer programming to make sure that only one part of the program can access or modify specific data at a time, preventing conflicts when multiple things try to use that data simultaneously), the maximum amount of body pairs that can be queued at a time and the maximum size of the constraint buffer. It then creates a physics system and initializes it, and lastly it creates a body interface which is the main way of interacting with bodies in the physics system. While profiling i tested if it makes a big difference in performance if you already allocate a lot for the tempAllocator, and the max amount of bodies etc. and it does not make a difference so i hardcoded it to already have a big allocation, if it would have made a big difference, I would have made a parameter in the initialisation function so you can set it there depending on what your project needs. If you do go over this limit, in initializing before running it, it will break at an error code Jolt made tell you that too many bodies have been created. On the other hand, if you create too many shapes while already running, it seems that jolt takes care of it by just removing some shapes, you can see this by them falling throuhg the floor:
![allocatingTooMany](https://github.com/AnnedeGeus01/AnnedeGeus01.github.io/assets/144111374/9e945638-527b-4f4b-8009-7f35e1aec124)
- The *Update* function is made to be called each frame to update the physics simulation. It iterates over entities that have both a *Transform* and a *JoltBody* component, updating their positions and rotations based on the physics simulation, this allows for a nice synchronization between the ECS and the physics simulation.
- Functions like *createBox*, *createSphere*, and *createConvexHull* make the process of creating entities with physics bodies easier. They handle the creation of both the *Transform* and *joltPhysics::JoltBody* components for the specified shapes. For me I use a mesh renderer to create my physics body, but all you need is the set of points that the mesh has, and pass it into the function and that way you can get the shape, I just chose to do this since this saves me space and extra work in the main, and it is just done each time in the create function itself.

### 5. Creating bodies, in the JoltBody class.
```c++
class JoltBody
{
    public:
        JoltBody(PhysicsSetup& physicsSetup, BodyCreationSettings settings, ShapeType shapeType);

        ~JoltBody();

        void debugDrawBody();
        void setVelocity(const vec3& velocity);
        vec3 getPosition() const;
        vec3 getVelocity() const;
        quat getRotation() const;

    private:
        Body* mBody = nullptr;
        ShapeType mShapeType; // Used for debug drawing
};
```
- Represents the physics component attached to entities. Contains a Jolt Body instance *mBody* and the type of shape *mShapeType* associated with the physics body.
- The *JoltBody* class encapsulates the interaction with the Jolt physics library. It hides the Jolt's body creation and provides functions such as *setVelocity*, *getPosition*, *getVelocity*, and *getRotation* for easy manipulation of physics-related objects. JoltBody() is something you call in the end of makin your shapes in the PhysicsSetup class.
- The *debugDrawBody* function, conditioned by the DEBUG macro (so I dont have any problems running it in release), gives visual debugging information by drawing the shapes of physics bodies using my engines way of drawing lines, this might be different depending on what entity component system you use. This is all based on shapes, so spheres actually have a circle, squares have a square and convex shapes have the shape of the actual convex shape. This is also already being called in the update, it will automatically show the debug lines. Tip: for boxes you can just use the *getLocalBounds()* function to draw the box, for circles I recommend to get the center of mass and the radius with the functions from Jolt, and for the convex shape I would find the function for getting the face vertices, so you can draw the actual triangles.

### 6. What to put in the update loop.
The example below shows how easy it is to create a physics entity with just a few lines of code. This simplicity is good for anyone who may not be familiar with the Jolt physics library. You don't need to know anything about jolt to be able to use these classes. 

```c++
// In the main:
// system set up example:
bool gravity = true;
auto& physics = ecs.CreateSystem<joltPhysics::PhysicsSetup>();
physics.Initialize(gravity);

// Body creation example:
const auto floorEntity = ecs.CreateEntity();
physics.createBox(floorEntity, vec3{0.0f, 0.0f, 0.0f}, vec3(100.0f * 0.5f, 100.0f * 0.5f, 1.f), false);

// Convx shape example:
const auto convexEntity = ecs.CreateEntity();
auto& meshRenderer = Engine.ECS().CreateComponent<MeshRenderer>(convexEntity, model->CreateMeshRendererFromNode("Suzanne"));
physics.createConvexHull(convexEntity, vec3(0.5, 0, 5), vec3(1, 1, 1), meshRenderer, true);

// Body with control example:
const auto playerEntity = ecs.CreateEntity();
physics.createSphere(playerEntity, vec3(7.0, 0.0, 5.0), 1.f, true);
Engine.ECS().CreateComponent<DirectionalControl>(playerEntity, 0, 6.0f); // Note that Jolt does NOT give a way t check for input, this should be something in the engine or self made.

// If you want to use input to move a body you can do something like this in the update:
for (const auto& [entity, control, physicsBody] : ecs.Registry.view<DirectionalControl, joltPhysics::JoltBody>().each())
{
    control.ComputeInputVelocity();
    // Update the position based on the input velocity
    vec3 velocity = vec3(control.m_inputVelocity.x, control.m_inputVelocity.y, physicsBody.getVelocity().z);
    physicsBody.setVelocity(velocity); // Just set the velocity, and the update function in PhysicsSetup will take care of the rest.
}
```

## Final output:
- Final output with Debug Drawing:
![DebugDrawing](https://github.com/AnnedeGeus01/AnnedeGeus01.github.io/assets/144111374/3b35e010-ddfb-4c85-a4db-ea31046aa297)

- Final output run in Release:
![Release](https://github.com/AnnedeGeus01/AnnedeGeus01.github.io/assets/144111374/464b5631-81a1-4a07-823f-36af10f2a09c)




