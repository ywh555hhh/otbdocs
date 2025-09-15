# 线程池动态切换方案

##  需求背景

Per_thread采用每个用户连接对应一个handle_connection线程的方式处理用户请求。Thread_pool采用一定的工作线程池（线程池大小通常与CPU核数相同）来处理用户连接请求。Thread_pool模式与Per_thread模式相比，更适合处理大量用户的短查询请求，可以避免频繁创建和销毁线程，避免并发数较高的情况下的临界资源争抢。Thread_pool的不足在于当用户请求偏向于慢查询时，工作线程阻塞在高时延操作上，难以快速响应新的用户请求，导致系统吞吐量反而相较于Per_thread模式更低。

Per_thread模式与Thread_pool模式各有优缺点，系统需要根据用户的业务类型灵活地进行切换。遗憾的是，当前两种模式的切换必须重启服务器才能完成。通常而言，两种模式相互转换的需求都是出现在业务高峰时段，此时强制重启服务器将对用户业务造成严重影响。为了提高Per_thread模式与Thread_pool模式切换的灵活程度，提出了线程池动态切换的需求。

##  Per_thread模式分析

Per_thread模式为避免频繁创建和销毁pthread，对handle_connection有一个重用逻辑，下面以伪代码的方式分析Per_thread对用户连接的处理逻辑：

``` java
hanlde_connection 在退出前调用了my_thread_end函数
    my_thread_init
    for(;;) {
        thd = init_new_thd(channel_info)
        if (pthread_reused) //这里的复用是指一个线程退出了，也需要调用end_connection，close_connection等函数清理资源，最后阻塞在block_until_new_connection等待新的连接进来以后复用该pthread，挺有借鉴意义的
            PSI_THREAD_CALL(new_thread)(key_thread_one_connection, thd, thd->thread_id()); 
            PSI_THREAD_CALL(set_thread_os_id)(psi); 
            PSI_THREAD_CALL(set_thread)(psi);
        mysql_thread_set_psi_id(thd->thread_id());
        mysql_thread_set_psi_THD(thd);
        mysql_socket_set_thread_owner(thd->get_protocol_classic()->get_vio()->mysql_socket);

        thd_manager->add_thd(thd);//在thd_manager中注册新的连接
        thd_prepare_connection(thd)//新请求的连接与认证
        while (thd_connection_alive(thd))//不断处理用户请求直至连接退出
            do_command
        end_connection//连接成功后退出
        close_connection//关闭连接
        thd->get_stmt_da()->reset_diagnostics_area();
        thd->release_resources(); //清理thd资源
        thd_manager->remove_thd(thd);//在thd_manager中移除连接

        Connection_handler_manager::dec_connection_count(extra_port_connection);//全局线程连接数减一
        thd->set_psi(NULL);//清空thd psi信息
        PSI_THREAD_CALL(delete_current_thread)();
        delete thd;//删除thd
        Per_thread_connection_handler::block_until_new_connection(); //避免关闭当前handle_connection线程，阻塞至有新连接请求，重新当前线程
        pthread_reused = true;
    }

    my_thread_end();
        PSI_THREAD_CALL(delete_current_thread);
        --THR_thread_count
        set_mysys_thread_var(NULL);
    my_thread_exit(0);//handle_connection线程退出
		pthread_exit
```

##  Thread_pool模式分析

Thread_pool模式中，每个CPU对应一个线程组，线程组通常只有一个或多个活跃的worker线程和一个listener线程，每个线程组创建的线程上限为threadpool_oversubscribe。

全局的timer线程定期检查各个线程组，如果发现线程组中event queue很久未被消费，则唤醒或创建新的工作线程。

listener线程与worker线程可以灵活切换，当未到oversubscribed状态，如果线程组没有listener线程，则worker thread可以转化为listener线程。达到oversubscribed状态后，listener线程可能充当worker thread处理用户请求。

当新用户请求连接时，会按照一定的方式将其分配到对应的线程组中，该连接后续的所有请求都交于该工作组处理，直至该用户连接退出。

所有的用户请求（包括新连接的实际建立）都对应于一个Thread_pool event。listener线程用于监听用户请求，出现新event时将event按优先级存放到high_prio_queue或queue中，high_prio_queue中的event将在worker线程中优先被处理。worker线程的职责是不断重复get_event和handle_event的操作。下面来看handle_event的伪代码：

``` java
handle_event
	if (!connection->logged_in)
		err = threadpool_add_connection(connection->thd); //调用thd_prepare_connection进行新请求的连接与认证
		connection->logged_in = true;
	else
		err = threadpool_process_request(connection->thd); //调用do_command处理用户请求；详细堆栈如下

	err = start_io(connection); //1. 如果group_size变化，则调整connection到新的group；2. 重新bind to poll
	if (err) connection_abort(connection); //退出连接，详细堆栈如下


 threadpool_process_request
    thread_attach(thd);
    thd_set_net_read_write(thd, 0);
    do_command(thd);
    MYSQL_SOCKET_SET_STATE(thd->get_protocol_classic()->get_vio() ->mysql_socket, PSI_SOCKET_STATE_IDLE);
    MYSQL_START_IDLE_WAIT(thd->m_idle_psi, &thd->m_idle_state);
    thd->m_server_idle= true; 
	
connection_abort
    threadpool_remove_connection(connection->thd);
        thread_attach
        thd_set_net_read_write(thd, 0);
        end_connection(thd);       
        close_connection(thd, 0);  
        thd->release_resources();
        PSI_THREAD_CALL(delete_thread)(thd->get_psi());
        Global_THD_manager::get_instance()->remove_thd(thd);
        Connection_handler_manager::dec_connection_count();
        delete thd;
    group->connection_count--;
```

##  Per_thread to Thread_pool

Per_thread to Thread_pool分为两部分。第一部分是新连接自动切换为Thread_pool方式，这种方式只需要在内核中支持Per_thread、Thread_pool两种模式共存，并且采用Connection_handler_manager::thread_handling来控制新连接的处理方式即可，较为简单，这里不再详细讨论。

下面重点讨论如何将尚未关闭的Per_thread连接平滑迁移到Thread_pool中。

正常Per_thread模式下，handle_connection线程在thd_connection_alive(thd)时会不断循环do_command。

Per_thread to Thread_pool的关键在于打破上述循环，将已经完成连接的thd平滑迁移至Thread_pool某线程组的event queue中，并顺势关闭thd原先所在的handle_connection线程。

tp_migrate 类似于 Thread_pool_connection_handler::add_connection

``` java
Thread_pool_connection_handler::add_connection
{
	//1. 获取channel，通过channel创建thd
	THD *thd = channel_info->create_thd();
	//2. 创建connection，event队列中的基本元素
	connection_t *connection = alloc_connection(thd);
	//3. 设置新thread_id和线程开始时间
	thd->set_new_thread_id();
	thd->start_utime= thd->thr_create_utime= my_micro_time();
	//4. 设置线程的调度方法
	thd->scheduler= &tp_event_functions;
	//5. 将thd添加至全局的THD_manager中
	Global_THD_manager::get_instance()->add_thd(thd);
	//6. 线程池相关设置；connection放入相应的thread group中
	thd->event_scheduler.data= connection;
	thread_group_t *group= &all_groups[thd->thread_id() % group_count];
	connection->thread_group= group;
	group->connection_count++; 
	queue_put(group, connection);
}
```

由于THD在one_connection_one_thread阶段已经完成了THD_manager的注册，设置了thread_id和start_utime，完成了注册连接，因此上述步骤中1,3,5都不再需要。

handle_event的主要逻辑已经在第3节展示过，这里为了完成thd from Per_thread to Thread_pool的平滑迁移，还需要对thd做一定的装备：

``` java
handle_event
	if (!connection->logged_in)
		err = threadpool_add_connection(connection->thd); //调用thd_prepare_connection进行新请求的连接与认证
		connection->logged_in = true;
	else
		if (connection->from_per_thread) //如果是来自于Per_thread的thd，还需要进行thd thread_pool相关的状态设置
			connection->from_per_thread = false;
			threadpool_process_request_prepare(connection->thd)
		err = threadpool_process_request(connection->thd); //调用do_command处理用户请求；

	err = start_io(connection); //1. 如果group_size变化，则调整connection到新的group；2. 重新bind to poll
	if (err) connection_abort(connection); //退出连接，详细堆栈如下
```

迁移失败则继续使用Per_thread模式处理该连接，并且打印一条错误日志。

##  Thread_pool to Per_thread

Thread_pool to Per_thread同样分为两部分。第一部分是将新连接全部交付给Per_thread模式处理，这种方式同样是只需要通过Connection_handler_manager::thread_handling控制新连接的处理模式即可，我们也不再详细讨论。

下面讨论，如何将运行中Thread_pool connection平滑迁移至Per_thread中。

Thread_pool模式下的worker线程每次执行完threadpool_add_connection或threadpool_process_request会调用start_io，将connection重新bind to pool。这是一个合适的切换时间点。在此刻需要伪造channel_info，并新增channel_info->from_thread_pool状态。利用上述伪造的channel_info创建handle_connection线程。

``` java
handle_event
	if (!connection->logged_in)
		err = threadpool_add_connection(connection->thd); //调用thd_prepare_connection进行新请求的连接与认证
		connection->logged_in = true;
	else
		if (connection->from_per_thread) //如果是来自于Per_thread的thd，还需要进行thd thread_pool相关的状态设置
			connection->from_per_thread = false;
			threadpool_process_request_prepare(connection->thd)
		err = threadpool_process_request(connection->thd); //调用do_command处理用户请求；

	ulong tmp_thread_handling = Connection_handler_manager::thread_handling;
	if (Connection_handler_manager::thread_handling == SCHEDULER_ONE_THREAD_PER_CONNECTION)
		Connection_handler_manager *handler_manager = Connection_handler_manager::get_instance();
		handler_manager->get_connection_handler(tmp_thread_handling)->migrate(thd);//在migrate中伪造的channel_info，设置from_thread_pool为true，创建handle_connection线程
		clean_connection_from_thread_pool(connection);//从Thread_pool中清理该连接的信息，可借鉴connection_abort()，但不需要关闭连接和清理thd
		return；
	
	err = start_io(connection); //1. 如果group_size变化，则调整connection到新的group；2. 重新bind to poll
	if (err) connection_abort(connection); //退出连接，详细堆栈如下
	return;
```

在handle_connection线程中，通过channel_info->from_thread_pool状态跳过init_new_thd，thd_manager->add_thd(thd)，以及实际连接和认证过程。在进行必要的状态设置后，直接进入do_command环节。

``` java
hanlde_connection 在退出前调用了my_thread_end函数
    my_thread_init
    for(;;) {
		if (!channel_info->from_thread_pool)
        	thd = init_new_thd(channel_info) //利用from_thread_pool状态跳过
		else
			handle_connection_prepare//相应的状态准备
        if (pthread_reused || channel_info->from_thread_pool) //from_thread_pool状态下运行
            PSI_THREAD_CALL(new_thread)(key_thread_one_connection, thd, thd->thread_id()); 
            PSI_THREAD_CALL(set_thread_os_id)(psi); 
            PSI_THREAD_CALL(set_thread)(psi);
        mysql_thread_set_psi_id(thd->thread_id());
        mysql_thread_set_psi_THD(thd);
        mysql_socket_set_thread_owner(thd->get_protocol_classic()->get_vio()->mysql_socket);
	
		if (!channel_info->from_thread_pool)
        	thd_manager->add_thd(thd);//在thd_manager中注册新的连接
       	 	thd_prepare_connection(thd)//新请求的连接与认证

        while (thd_connection_alive(thd))//不断处理用户请求直至连接退出
            do_command
        end_connection//连接成功后退出
        close_connection//关闭连接
        thd->get_stmt_da()->reset_diagnostics_area();
        thd->release_resources(); //清理thd资源
        thd_manager->remove_thd(thd);//在thd_manager中移除连接

        Connection_handler_manager::dec_connection_count(extra_port_connection);//全局线程连接数减一
        thd->set_psi(NULL);//清空thd psi信息
        PSI_THREAD_CALL(delete_current_thread)();
        delete thd;//删除thd
        Per_thread_connection_handler::block_until_new_connection(); //避免关闭当前handle_connection线程，阻塞至有新连接请求，重新当前线程
        pthread_reused = true;
    }

    my_thread_end();
        PSI_THREAD_CALL(delete_current_thread);
        --THR_thread_count
        set_mysys_thread_var(NULL);
    my_thread_exit(0);//handle_connection线程退出
		pthread_exit
```

迁移失败则继续使用Thread_pool模式处理该连接，并且打印一条错误日志。