共享内存
==========

分配共享内存
--------------

可以通过调用 ``odp_shm_reserve()`` API创建共享内存块。
该API调用需要传入共享内存块名称、块大小、对齐要求和可选标志参数。它返回一个 `` odp_shm_t`` 结构体。
块大小和对齐以字节为单位。

.. note::  

    提供的名称不一定是唯一的，即，在保留不同的块时，可以使用相同的名称。


创建一个共享内存块代码如下：

.. code-block:: c

	#define ALIGNMENT 128
	#define BLKNAME "shared_items"

	odp_shm_t shm;
	uint32_t shm_flags = 0;

	typedef struct {
	...
	} shared_data_t;

	shm = odp_shm_reserve(BLKNAME, sizeof(shared_data_t), ALIGNMENT, shm_flags);


获取共享内存块地址
-------------------

上述API调用返回的 ``odp_shm_t`` 句柄检索指定创建的共享内存块的地址（在调用者的ODP线程虚拟地址空间中）。

获取贡献内存块的代码如下：

.. code-block:: c

	shared_data_t *shared_data;
	shared_data = odp_shm_addr(shm);

接口 ``odp_shm_addr()`` 返回的地址仅在ODP线程空间中有效。虽然 ``odp_shm_t`` 句柄可以在ODP线程之间共享，并且在任何线程中保持有效，
但 ``odp_shm_addr()`` 返回的地址可能根据线程空间而不同（对于同一个shm块），因此不应该在ODP线程之间共享。
举个例子，在两个ODP线程之间通过IPC传送shm句柄是正确的，并且让这些线程都执行各自的 ``odp_shm_addr()`` 来获取共享内存块的地址。
但是如果直接 ``odp_shm_addr()`` 返回的地址从一个ODP线程发送到另一个ODP线程则可能会失败（该地址在接收端的地址空间中可能没意义）。

即使调用 ``odp_shm_addr()`` 的线程与原来调用 ``odp_shm_reserve()`` 的线程不一致， ``odp_shm_addr()`` 返回的地址仍然保证根据在块创建时提供的对齐要求对齐。

所有共享内存块在任何ODP线程寻址空间中都是连续的，address~address+size，其中size是共享内存块的大小，这个空间是可读写的，映射了整个共享内存块。

内存行为
----------

默认情况下，ODP线程被假定为缓存一致性系统：对共享内存块执行的任何更改都将保证最终对共享此内存块的其他ODP线程可见。
然而，ODP中并没有共享内存上任何操作相关联的隐式内存屏障，也就是说当ODP线程执行的改变对另一个ODP线程是不可见的，
故使用共享内存块的程序需要执行ODP提供的一些内存屏障来保证ODP线程之间共享数据的一致性。

如果ODP线程具有单独的虚拟空间（ODP线程被实现为进程），则给定的共享内存块映射到不同的ODP线程虚拟地址空间上各个ODP线程各不相同。
但是， ``ODP_SHM_SINGLE_VA`` 标志可以在 ``odp_shm_reserve()`` 调用时使用，以保证所有ODP线程的地址唯一性，无论其实现或创建的时间如何。

根据名称查找
-------------

上面讲过，共享内存块指针可以通过IPC机制在ODP线程之间传递，然后执行API来获取当前ODP线的地址。
获取已创建的共享内存块的指针的更简单的方法是通过接口 `` odp_shm_lookup()`` 调用来实现。
但是，这需要ODP线程来提高共享内存块的名称，假如没有找到对应名称的内存块，则返回 ``ODP_SHM_INVALID`` 。
当使用相同名称保存多个共享内存块时，查找接口将返回任意这些块指针中的一个。

根据名称查找块指针和地址：

.. code-block:: c

	#define BLKNAME "shared_items"

	odp_shm_t shm;
	shared_data_t *shared_data;

	shm = odp_shm_lookup(BLKNAME);
	if (shm != ODP_SHM_INVALID) {
		shared_data = odp_shm_addr(shm);
		...
	}

释放共享内存块
----------------

使用 ``odp_shm_free()`` API调用执行释放共享内存块。
该接口需要共享内存块指针作为参数。
允许任何ODP线程在共享内存块上执行 ``odp_shm_free()`` ，即申请和释放共享内存块的线程可能不同。
共享内存块应该只被释放一次，一旦释放，共享内存块就不应该被任何ODP线程再次引用。

释放共享内存块：

.. code-block:: c

	if (odp_shm_free(shm) != 0) {
		...//handle error
	}

与外部程序共享内存
-------------------

ODP提供了与ODP实例外部的实体共享内存的方法：

与外部非ODP线程共享内存块是通过在调用 ``odp_shm_reserve()`` 时设置 `` ODP_SHM_PROC`` 标志来实现的。
这些非ODP线程如何检索共享内存块依赖于具体的实现和操作系统。

与外部ODP实例（运行于同一个操作系统）共享内存块是通过调用 ``odp_shm_reserve()`` 时设置 `` ODP_SHM_EXPORT `` 标志来实现的。
在ODP实例A中使用此标志创建的内存块可以通过在ODP实例B上使用 ``odp_shm_import()`` 接口映射到远程ODP实例B上（在相同操作系统中）。

ODP实例间共享内存: instance A

.. code-block:: c

	odp_shm_t shmA;
	shmA = odp_shm_reserve("memoryA", size, 0, ODP_SHM_EXPORT);
	
ODP实例间共享内存: instance B

.. code-block:: c

	odp_shm_t shmB;
	odp_instance_t odpA;

	/* get ODP A instance handle by some OS method */
	odpA = ...

	/* get the shared memory exported by A:
	shmB = odp_shm_import("memoryA", odpA, "memoryB", 0, 0);
	
.. note::

    每个ODP实例的范围限制shmA和shmB（您不能在其所属的ODP实例之外使用它们）。
    另请注意，两个ODP实例必须在完成后调用odp_shm_free（）。
	
共享内存创建标志
-------------------

``odp_shm_reserve()`` API的最后一个参数是一组ORed标志。当前支持如下几种标志：

ODP_SHM_PROC
~~~~~~~~~~~~~~

当给出此标志时，分配的共享内存将在ODP之外变得可见。
非ODP线程（例如通常的linux进程或linux线程）将能够使用本机（非ODP）OS调用（如shm_open（）和mmap（对于linux））来访问内存。
每个ODP实施应提供关于在该特定平台上如何完成此映射的描述。

ODP_SHM_EXPORT
~~~~~~~~~~~~~~

当给出此标志时，分配的共享内存将在同一个操作系统上运行的其他ODP实例变得可见。
想要看到此导出内存的其他ODP实例应使用 ``odp_shm_import()`` ODP函数。

ODP_SHM_SW_ONLY
~~~~~~~~~~~~~~~~

该标志指示ODP共享内存将仅由ODP应用软件使用：没有硬件（如DMA或其他加速器）访问内存。
这个内存不会涉及其他的ODP调用（ODP调用可能隐含地涉及到HW，这取决于ODP的实现），除了 ``odp_shm_lookup()`` 和 ``odp_shm_free()`` 。
ODP实现可以使用该标志作为性能优化的提示，或者也可以忽略该标志。

ODP_SHM_SINGLE_VA
~~~~~~~~~~~~~~~~~~

该标志用于保证共享内存被映射的地址的唯一性：没有该标志，给定的内存块可能会被不同的ODP线程映射到不同的虚拟地址（假设目标具有虚拟地址）。
这意味着 ``odp_shm_addr()`` 返回的值在不同的线程中是不同的。
设置此标志保证共享此内存块的所有ODP线程将在在所有ODP线程上调用 ``odp_shm_addr()`` 返回相同的值。
注意，ODP实现可能会限制可以分配这个标志的内存数目。
