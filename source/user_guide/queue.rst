队列
=====

队列是ODP提供的基本事件排序机制，所有的ODP程序都显式或隐式地使用队列。
队列是通过API ``odp_queue_create()`` 来创建的，该API返回一个类型为 ``odp_queue_t`` 的指针，该指针用于所有使用该队列的API调用。
每个Queue具有两种ODP类型中的一个，POLL和SCHED，用于指示如何使用它们。
POLL队列由ODP应用程序直接管理，而SCHED队列使用ODP调度器提供自动可扩展调度和同步服务。

POLL queues 操作

.. code-block:: c

    odp_queue_t poll_q1 = odp_queue_create("poll queue 1", ODP_QUEUE_TYPE_POLL, NULL);
    odp_queue_t poll_q2 = odp_queue_create("poll queue 2", ODP_QUEUE_TYPE_POLL, NULL);
    ...
    odp_event_t ev = odp_queue_deq(poll_q1);
    ...do something
    int rc = odp_queue_enq(poll_q2, ev);

关键区别是，在POLL队列中，事件出队操作是应用程序负责的，而SCHED队列中的事件出队则是ODP调度程序完成的。

SCHED queues 操作

.. code-block:: c

    odp_queue_param_t qp;
    odp_queue_param_init(&qp);
    odp_schedule_prio_t prio = ...;
    odp_schedule_group_t sched_group = ...;
    qp.sched.prio = prio;
    qp.sched.sync = ODP_SCHED_SYNC_[NONE|ATOMIC|ORDERED];
    qp.sched.group = sched_group;
    qp.lock_count = n; /* Only relevant for ordered queues */
    odp_queue_t sched_q1 = odp_queue_create("sched queue 1", ODP_QUEUE_TYPE_SCHED, &qp);

    ...thread init processing

    while (1) {
            odp_event_t ev;
            odp_queue_t which_q;
            ev = odp_schedule(&which_q, <wait option>);
            ...process the event
    }
    
当使用sched queue时，发送者选择一个目标队列，将事件发送到队列中。
发送者不知道哪个ODP线程（哪个core）或硬件加速器将处理这个事件，但是队列中的所有事件最终将被调度和处理。

可以看出，可以在SCHED队列创建时指定队列的其他属性，它们控制调度程序如何处理其中包含的事件，这些属性包括组，优先级和同步类。


组
---

调度器的工作是从最高优先级的SCHED queue中返回下一个事件给调用者。SCHED queue必须是调用者有资格接收的队列。
这个是由队列创建时设置的队列调度器组及调用者的调度器组掩码来指定的。
调度器组由 ``odp_scheduler_group_t`` 类型的指针表示，并由 ``odp_scheduler_group_create()`` 接口创建。
ODP预定义了多个调度器组，包括 ``ODP_SCHED_GROUP_ALL`` ``ODP_SCHED_GROUP_WORKER`` 及 ``ODP_SCHED_GROUP_CONTROL`` 。
应用程序可以自由创建其他调度器组，线程可以使用 ``odp_scheduler_group_join()`` 和 ``odp_scheduler_group_leave()`` 加入或离开调度器组。


优先级
--------

数据结构 ``odp_queue_param_t`` 的prio字段指定队列调度的优先级，即如何选择符合条件的调度器组中的队列进行调度。
队列的默认调度优先级是 ``NORMAL`` 但可以根据需求设置成 ``HIGHEST`` 或 ``LOWEST`` 。


同步
-----

除了在多核心环境中为ODP应用提供自动可扩展性的调度功能之外，调度程序的另一个主要功能是提供事件同步服务，大大简化并行处理环境中的应用程序编程。
队列的SYNC模式决定调度程序如何处理来自同一队列的多个事件的同步处理。
ODP支持三种类型的队列调度器同步区：并行，原子和有序。

并行队列
---------

指定 ``ODP_SCHED_SYNC_NONE`` 模式的SCHED队列在处理事件的方式上是不受限制的。

.. _odp-paralles-queue:

.. figure:: img/odp-paralles-queue.*

   Parallel Queue Scheduling
   
在并行队列中保存的所有事件都有资格同时进行调度，并且它们之间的所有必需的同步都是应用程序负责的。
源自并行队列的事件也因此具有最高的吞吐率，但是它们也可能涉及大量的应用程序工作。
在上图中，四个线程正在调用 ``odp_schedule()`` 来获取要处理的事件。
调度程序已经将来自第一个队列的三个事件并行分配给三个线程。
第四个线程正在处理来自第三个队列的单个事件。
第二个队列可能是空的，优先级较低，或者不是在与调度程序服务的任何线程匹配的调度程序组中。


原子队列
---------

原子队列简化事件同步，因为每一次只有一个线程可以处理给定原子队列中的事件。
由于锁定是由调度程序隐式完成的，因此原子队列调度的事件可以被锁定。
请注意，如果使用 ``odp_schedule_multi()`` ，调用者可能会从同一个原子队列接收一个或多个事件。
在这种情况下，这些多个事件都共享相同的原子调度上下文。

.. _odp-atomic-queue:

.. figure:: img/odp-atomic-queue.*

   Atomic Queue Scheduling
   
在这个例子中，无论在一个原子队列中可能有多少个事件，每次只有一个调用线程可以接收它的调度事件。
这里两个线程处理来自两个不同原子队列的事件。
请注意，不同的原子队列之间不存在来自同一原子队列的事件之间的同步。
与原子队列相关联的队列上下文将持续到下次调用调度程序或直到应用程序通过调用 ``odp_schedule_release_atomic()`` 显式释放它。
请注意，虽然原子队列简化了编程，但原子队列的串行性质可能会削弱扩展性。

有序队列
---------

有序队列同时提供了并行队列的可扩展性和原子队列的易同步性。

.. _odp-order-queue:

.. figure:: img/odp-order-queue.*

   Ordered Queue Scheduling
   
当从有序队列调度事件时，调度程序从队列中并行分配多个事件到不同的线程，
并且调度程序还可以确保输出队列上的这些事件的相对顺序与其起始有序队列的序列相同。

与原子队列一样，与有序队列相关联的排序保证是指源自同一队列的事件，而不是源自不同队列的事件。
因此，在该图中，三个线程分别来自第一有序队列的处理事件5,3和4。
不管这些线程如何完成处理，这些事件将以它们的输出队列的原始相对顺序显示。

有序保证
~~~~~~~~~

无论事件是否被发送到不同的输出队列，相对顺序都将被保留。
例如，如果某些事件被发送到输出队列A，而其他事件被发送到输出队列B，则这些输出队列上的事件将仍然与它们的发起队列具有相同的相对顺序。
类似地，如果处理消耗事件，使得对于它们中的一些（例如，作为IP片段重新组合处理的一部分）不发出输出，其他事件仍然将相对于这些序列间隙被正确地排序。
最后，如果针对给定的顺序排入多个事件（例如，作为MTU注意事项的分组分割处理的一部分），则这些事件中的每一个将占据目标输出队列中的发起者的序列。
在这种情况下，这些事件的相对顺序将按照线程 ``odp_queue_enq()`` 调用它们的顺序。

与有序队列调度事件相关联的有序上下文将持续到下一个调度程序调用，或者直到调用 ``odp_schedule_release_ordered()`` 的显式释放。
该调用可以用作性能咨询，线程不再需要为当前上下文订购保证。
因此，当前调度程序上下文中的任何后续排队将被视为线程在并行队列上下文中运行。


顺序锁定
~~~~~~~~~

调度器处理有序队列的另一个强大功能是顺序锁。
与每个有序队列相关联的多个顺序锁是在队列创建时由lock_count参数指定的。
顺序锁提供了有效的方式来在顺序上下文中执行顺序处理。
例如，相关顺序5,6和7的假定事件由三个不同的线程并行执行。
顺序锁将使这些线程能够同步，以便它们可以在其起始队列顺序中执行一些关键部分。
每个有序队列支持的顺序锁的数量取决于具体实现（可通过 ``odp_config_max_ordered_locks_per_queue()`` API查询）。
如果实现支持多个有序锁，则这些锁可用于保护给定有序上下文中的不同有序临界区。

小结：有序队列
~~~~~~~~~~~~~~~

要了解这些注意事项如何组合在一起，请考虑以下代码：
Processing with Ordered Queues

.. code-block:: c

    void worker_thread()
            odp_init_local();
            ...other initialization processing

            while (1) {
                    ev = odp_schedule(&which_q, ODP_SCHED_WAIT);
                    ...process events in parallel
                    odp_schedule_order_lock(0);
                    ...critical section processed in order
                    odp_schedule_order_unlock(0);
                    ...continue processing in parallel
                    odp_queue_enq(dest_q, ev);
            }
    }
    
这段代码表示了在有序队列上运行的典型工作线程的简化结构。
并行处理多个事件，并使用有序队列确保它们以与发起的顺序相同的顺序放置在dest_q上。
在并行处理的同时，使用有序锁可以使关键部分在整个并行流程中按顺序进行处理。
当一个线程到达 ``odp_schedule_order_lock()`` 调用时，它等待直到所有先前事件的锁定的锁定顺序已经解决，然后进入临界区。
``odp_schedule_order_unlock()`` 调用释放关键部分，并允许下一个订单输入。


队列调度小结
-------------

有序和并行队列由于并行事件处理而提高了原子队列的吞吐量，但要求应用程序采取措施确保上下文数据同步（如果需要）。