# New Threading POC

This is a POC implementation of a new threading approach. As such, the implementation is very rough (leaky, incomplete) and I've only tested it against PHP 7.2. Everything and anything is subject to change still.

The basic idea is that `Threaded` objects are abstracted away behind references to such objects. This means that when we would like to create a new thread, we now do something such as:
```php
<?php

class SomeClass implements Threaded
{
    public function run() {}
}

// ThreadRef::__construct(string $className, mixed ...$constructorArgs)
$tr = new ThreadRef(SomeClass::class);
```

By taking this approach, `Threaded` objects themselves no longer require their properties to be serialised. This is because we no longer need to pass around `Threaded` objects anymore (instead, their references can be safely passed around), and so `Threaded` objects no longer need to be able to operate in multiple execution contexts.

This introduces a new problem: how can inter-thread communication be performed? To solve this problem, I decided to make threads "process-like" in terms of their communication between each another. The inter-process communication (IPC) technique I have decided to implement so far is message queues. Message queues are mutex-controlled queues that can be passed around between threads, enabling for a producer-consumer communication style.

In code, this looks like the following:
```php
<?php

class T implements Threaded
{
    private $mq;

    public function __construct(MessageQueue $mq)
    {
        $this->mq = $mq;
    }

    public function run()
    {
        $this->mq->push(1);
        $this->mq->finish();
    }
}

$mq = new MessageQueue();

$tr = new ThreadRef(T::class, $mq);

$tr->start();

while (!$mq->isFinished() || $mq->hasMessages()) {
    if ($mq->pop($message)) {
        var_dump($message);
    }
}

$tr->join();
```

The `MessageQueue` class looks as follows:
```php
class MessageQueue
{
    /*
     * Set the message queue to a finished state.
     *
     * This notifies threads that are using it that no new messages will be.
     * pushed to the queue
     */
    public function finish(void) : void;

    /*
     * Checks to see if the queue has a finished state.
     */
    public function isFinished(void) : bool;

    /*
     * Pushes a message to the queue.
     */
    public function push(mixed $message) : void;

    /*
     * Pops a value from the queue.
     *
     * By using a boolean return value to denote whether a value was popped from
     * the queue or not, with a reference-based parameter to update on success,
     * we avoid having to expose a mutex lock to PHP developers. Otherwise, the
     * following would need to be performed: acquire a lock, check for presence
     * of messages, pop a message if present, release lock. So this approach
     * hides away the need for mutexes.
     */
    public function pop(mixed &$message) : bool;

    /*
     * Checks to see if any messages are left in the queue
     *
     * This is used to continue fetching messages from a queue even when it has
     * finished.
     */
    public function hasMessages(void) : bool;
}
```

In future, I may implement other techniques, too (such as message passing). This will likely require `Threaded` objects to extend the `Threaded` class, rather than simply implementing it as an interface.

With the above in mind, the serialisation points are:
 - The arguments to the `ThreadRef` constructor
 - The values pushed to the message queue
