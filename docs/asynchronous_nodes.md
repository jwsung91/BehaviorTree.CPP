# Understand Asynchrous Nodes, Concurrency and Parallelism

When designing reactive Behavior Trees, it is important to understand 2 main concepts:

- what we mean by **"Asynchronous"** Actions VS **"Synchronous"** ones.
- The difference between **Concurrency** and **Parallelism** in general and in the context of BT.CPP.

## Concurrency vs Parallelism

If you Google those words, you will read many good articles about this topic.

BT.CPP executes all the nodes **Concurrently**, in other words:

- The Tree execution engine itself is single-threaded.
- all the `tick()` methods are **always** executed sequentially.
- if any `tick()` method is blocking, the entire execution is blocked.

We achieve reactive behaviors through "concurrency" and asynchronous execution.

In other words, an Action that takes a long time to execute should, instead,
return as soon as possible the state RUNNING to notify that the action was started,
and only when ticked again check if the action is completed or not.

An Asynchronous node may delegate this long execution either to another process,
another server or simply another thread.

## Asynchronous vs Synchronous

In general, an Asynchronous Action (or TreeNode) is simply one that:

- May return RUNNING instead of SUCCESS or FAILURE, when ticked.
- Can be stopped as fast as possible when the method `halt()` (to be implemented by the developer) is invoked.

When your Tree ends up executing an Asynchronous action that returns running, that RUNNING state is usually propagated backbard and the entire Tree is itself in the RUNNING state.

In the example below, "ActionE" is asynchronous and the RUNNING; when
a node is RUNNING, usually its parent returns RUNNING too.

![tree in running state](images/RunningTree.svg)

Let's consider a simple "SleepNode". A good template to get started is the StatefullAction

```c++
// Example os Asynchronous node that use StatefulActionNode as base class
class SleepNode : public BT::StatefulActionNode
{
  public:
    SleepNode(const std::string& name, const BT::NodeConfiguration& config)
      : BT::StatefulActionNode(name, config)
    {}

    static BT::PortsList providedPorts()
    {
        // amount of milliseconds that we want to sleep
        return{ BT::InputPort<int>("msec") };
    }

    NodeStatus onStart() override
    {
        int msec = 0;
        getInput("msec", msec);
        if( msec <= 0 )
        {
            // No need to go into the RUNNING state
            return NodeStatus::SUCCESS;
        }
        else {
            using namespace std::chrono;
            // once the deadline is reached, we will return SUCCESS.
            deadline_ = system_clock::now() + milliseconds(msec);
            return NodeStatus::RUNNING;
        }
    }

    /// method invoked by an action in the RUNNING state.
    NodeStatus onRunning() override
    {
        if ( std::chrono::system_clock::now() >= deadline_ )
        {
            return NodeStatus::SUCCESS;
        }
        else {
            return NodeStatus::RUNNING;
        }
    }

    void onHalted() override
    {
        // nothing to do here...
        std::cout << "SleepNode interrupted" << std::endl;
    }

  private:
    std::chrono::system_clock::time_point deadline_;
};
```

In the code above:

1. When the SleedNode is ticked the first time, the `onStart()` method is executed.
This may return SUCCESS immediately if the sleep time is 0 or will return RUNNING otherwise.
2. We should continue ticking the tree in a loop. This will invoke the method
`onRunning()` that may return RUNNING again or, eventually, SUCCESS.
3. Another node might trigger a `halt()` signal. In this case, the `onHalted()` method is invoked. We can take the opportunity to clean up our internal state.

## Avoid blocking the execution of the tree

A **wrong** way to implement the `SleepNode` would be this one:

```c++
// This is the synchronous version of the Node. probably not what we want.
class BadSleepNode : public BT::ActionNodeBase
{
  public:
    BadSleepNode(const std::string& name, const BT::NodeConfiguration& config)
      : BT::ActionNodeBase(name, config)
    {}

    static BT::PortsList providedPorts()
    {
        return{ BT::InputPort<int>("msec") };
    }

    NodeStatus tick() override
    {  
        int msec = 0;
        getInput("msec", msec);
        // This blocking function will FREEZE the entire tree :(
        std::this_thread::sleep_for( std::chrono::milliseconds(msec) );
        return NodeStatus::SUCCESS;
     }

    void halt() override
    {
        // No one can invoke this method, because I freezed the tree.
        // Even if this method COULD be executed, there is no way I can
        // interrupt std::this_thread::sleep_for()
    }
};
```

## The problem with multi-threading

In the early days of this library (version 1.x), spawning a new thread
looked as a good solution to build asynchronous Actions.

That was a bad idea, for multiple reasons:

- Accessing the blackboard in a thread-safe way is harder (more about this later).
- You probably don't need to.
- People think that this will magically make the Action "asynchronous", but they forget that it is still **their responsibility** to stop that thread "somehow" when the `halt()`method is invoked.

For this reason, user a usually discouraged from using `BT::AsyncActionNode` as a
base class. Let's have a look again at the SleepNode.

```c++
// This will spawn its own thread. But it still have problems when halted
class BadSleepNode : public BT::AsyncActionNode
{
  public:
    BadSleepNode(const std::string& name, const BT::NodeConfiguration& config)
      : BT::ActionNodeBase(name, config)
    {}

    static BT::PortsList providedPorts()
    {
        return{ BT::InputPort<int>("msec") };
    }

    NodeStatus tick() override
    {  
        // This code run in its own thread, therefore the Tree is still running.
        // Think looks good but the thread can't be aborted
        int msec = 0;
        getInput("msec", msec);
        std::this_thread::sleep_for( std::chrono::milliseconds(msec) );
        return NodeStatus::SUCCESS;
     }

    // The halt() method can not kill the spawned thread :()
    // void halt(); 
    }
};
```

A "correct" (but over-engineered) version of it would be:

```c++
// I will create my own thread here, for no good reason
class ThreadedSleepNode : public BT::AsyncActionNode
{
  public:
    ThreadedSleepNode(const std::string& name, const BT::NodeConfiguration& config)
      : BT::ActionNodeBase(name, config)
    {}

    static BT::PortsList providedPorts()
    {
        return{ BT::InputPort<int>("msec") };
    }

    NodeStatus tick() override
    {  
        // This code run in its own thread, therefore the Tree is still running.
        int msec = 0;
        getInput("msec", msec);

        using namespace std::chrono;
        auto deadline = system_clock::now() + milliseconds(msec);

        // periodically check isHaltRequested() 
        // and sleep for a small amount of time only (1 millisecond)
        while( !isHaltRequested() && system_clock::now() < deadline )
        {
            std::this_thread::sleep_for( std::chrono::milliseconds(1) );
        }
        return NodeStatus::SUCCESS;
     }

    // The halt() method can not kill the spawned thread :()
    // void halt(); 
    }
};
```

As you can see, this looks more complicated than the version we implemented
first, using `BT::StatefulActionNode`.
This pattern can still be useful in some case, but you must remember that introducing multi-threading make things more complicated and **should be avoided by default**. 