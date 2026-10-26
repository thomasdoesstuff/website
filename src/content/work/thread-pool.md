---
title: "My thread pool implementation"
date: 2026-10-25
summary: "A thread pool class I wrote for my team"
---
## Thread pools
You're probably already aware of what a thread pool is, but just in case, a thread pool is a group of threads that manage tasks together. A well-written thread pool class organizes tasks to be performed by the program and creates a standard interface through which they can be executed, observed, and debugged. If basic building blocks like this are done well in your code, it sets the whole team up for easy success.

When my team needed a thread pool, this is some of the actual code I wrote for us. As a sidenote, this was a few years ago, before AI was what it is today. If anything I post on this website is not my own design, I will be clear and honest about it. But this one's all me.

**Threadpool.hpp**
```cpp
#include <condition_variable>
#include <functional>
#include <queue>
#include <thread>

template <size_t Size>
class Threadpool
{
  public:
    Threadpool() { }

    virtual ~Threadpool()
    {
      if (!_terminate)
      {
        waitForCompletion();
      }
      stop();
    }

    void start()
    {
      for (auto& thread : _threads)
      {
        thread = std::thread(&Threadpool::threadLoop, this);
      }
    }

    void stop()
    {
      std::unique_lock lock(_mutex);
      _terminate = true;
      lock.unlock();

      _cv.notify_all();
      for (auto& thread : _threads)
      {
        thread.join();
      }
    }

    void queueTask(const std::function<void()>& inTask)
    {
      std::unique_lock lock(_mutex);
      _tasks.push(inTask);
      lock.unlock();

      _cv.notify_one();
    }

    void waitForCompletion()
    {
      std::unique_lock lock(_mutex);
      _tasksCompleteCv.wait(lock, [this] {
        return _tasks.empty();
      });
    }

  private:
    void threadLoop()
    {
      std::unique_lock lock(_mutex, std::defer_lock);
      std::function<void()> task;

      while (true)
      {
        lock.lock();
        _cv.wait(lock, [this] {
          return !_tasks.empty() || _terminate;
        });
        if (_terminate) { return; }
        task = _tasks.front();
        _tasks.pop();
        lock.unlock();

        _tasksCompleteCv.notify_one();
        task();
      }
    }

    std::array<std::thread, Size> _threads;
    std::queue<std::function<void()>> _tasks;
    std::mutex _mutex;
    std::condition_variable _cv, _tasksCompleteCv;
    bool _terminate = false;
};
```

## std::condition_variable
One of the most criminally underutilized tools in C++ concurrency. I love these things. I'm using 2 of them here; `_cv` indicates when there is a task for the thread pool to complete, and `_tasksCompleteCv` is used to signal when the class is ready to run its destructor cleanly. Let's talk about this snippet from `threadLoop()`:
```cpp
      while (true)
      {
        lock.lock();
        _cv.wait(lock, [this] {
          return !_tasks.empty() || _terminate;
        });
        if (_terminate) { return; }
        task = _tasks.front();
        _tasks.pop();
        lock.unlock();

        _tasksCompleteCv.notify_one();
        task();
      }
```
When `lock` is passed to `_cv.wait()`, the lock is released until `_cv` is notified. When a task is added to the queue, that's done using `_cv.notify_one()`, which wakes up just one waiting thread. That thread checks the predicate in the given lambda expression and, if it returns true, exits the `_cv.wait()` call. Otherwise, it will continue waiting until `_cv` is notified again. In the case of the `stop()` function, `_cv.notify_all()` is used. This wakes all threads in the pool so they can exit the loop and be joined.

## The important little things
Let's focus in on that loop a bit more.
```cpp
        task = _tasks.front();
        _tasks.pop();
        lock.unlock();

        _tasksCompleteCv.notify_one();
        task();
```
These few lines are the most crucial for the performance of the class. Notice how the task does NOT run while the mutex is still locked. Instead, it gets copied out of the queue: `task = _tasks.front()`. The function wrapper type `std::function<void()>` is little more than a pointer, so this is a quick copy. The lock is released once the wrapper has been copied out of the queue and popped. This way the other threads in the pool spend less time waiting on the lock. Then, after notifying `_tasksCompleteCv` that a task has been removed from the queue, the task runs. The tasks all originate from outside of the Threadpool class, so the class' internal mutex is not relevant to them. The lock is only necessary to protect the queue itself. 

Another (frankly negligible) optimization:
```cpp
template <size_t Size>
class Threadpool

...

  private:
    std::array<std::thread, Size> _threads;
```
Because why allocate memory on the heap with a vector when you could allocate it on the stack with an array? Considering there's no real reason to be repeatedly constructing hundreds of thread pools, this makes little to no difference in this case. But in other cases this could be a big deal. It's important to know when templates can be leveraged to improve performance.

## Shutting down gracefully
One of the trickiest parts of concurrency is bringing all the threads back down properly at the end. It's not the fun part, but it's gotta get done.
```cpp
    virtual ~Threadpool()
    {
      if (!_terminate)
      {
        waitForCompletion();
      }
      stop();
    }
```
First off, the destructor is virtual. A thread pool is a class that makes sense to inherit from. I can't know what another developer might do with that, so I need to play it safe. The destructor should be virtual to ensure it will run. Anyway, what's happening here is `waitForCompletion()` uses `_taskCompleteCv.wait()` to block destruction until the threads are no longer busy. Then `stop()` sets `_terminate` to `true` and wakes all the threads to join them.
```cpp
    void stop()
    {
      std::unique_lock lock(_mutex);
      _terminate = true;
      lock.unlock();

      _cv.notify_all();
      for (auto& thread : _threads)
      {
        thread.join();
      }
    }
```
One thing to note is there are actually gaps in the destructor where the mutex is not locked, and something could technically call `queueTask()` after `waitForCompletion()` has finished. In our use case, we know that won't happen. But if it could happen, we move some locks around and set `_terminate` to `true` in `waitForCompletion()`, and have `queueTask()` check `_terminate` before it adds a task to the queue.
