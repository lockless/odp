ODP 应用程序编码结构
======================

头文件结构
------------

应用程序编程时仅需要包含 `` include/odp_api.h`` 文件，这个文件包含了 ``platform/<implementation name>/include/odp/api`` 文件，
以在该平台上提供API的完整定义。
定义API行为的doxygen文档都包含在公共APi文件中，接口实现在每个平台目录中。
如果 `` #define`` 不直接对用户可见的话，通常可以适当的访问函数来替代。

**Users include structure**

.. code-block:: shell

    ./
    ├── include/
    │   ├── odp/
    │   │   └── api/
    │   │       └── spec/
    │   │           └── The Public API and the documentation.
    │   │
    │   │
    │   ├── odp_api.h   This file should be the only file included by the
    │   │               application.


Initialization
-----------------

ODP应用程序需要优雅退出，所有程序必须在关闭ingress，情况所有队列等情况下才能执行终结函数退出程序。


Startup
----------

ODP应用程序必须调用的第一个API是 ``odp_init_global()`` 。
这需要两个指针。
第一个是 ``odp_init_t`` ，包含平台独立且可移植的ODP初始化数据，
第二个是 ``odp_platform_init_t`` ，用于平台特定的数据。

调用 ``odp_init_global()`` 建立ODP API框架，这个操作需要在调用其他API之前执行。
每个应用程序只调用一次。
全局初始化完成之后，每个线程依次调用``odp_init_local()`` 。
这为该线程建立本地ODP线程上下文，这个操作必须在该线程调用其他ODP API前执行。
线程类型是 ``ODP_THREAD_WORKER`` 或 ``ODP_THREAD_CONTROL`` 。


Shutdown
----------

关闭操作是初始化过程的反响逻辑，在调用 ``odp_term_global()`` 之前，每个线程调用 ``odp_term_local`` 来终止ODP。

应用程序初始化和终止时序
--------------------------

ODP应用程序遵循以下一般结构流程：

.. _odp-app-flow:

.. figure:: img/odp-app-flow.*

    ODP Application Structure Flow Diagram
