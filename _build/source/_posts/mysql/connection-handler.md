---
title: MySQL 建连过程分析
date: 2020-07-15 22:09:20
categories: 
- MySQL

---

本文结合 MySQL 8.0 的源码对 MySQL 的连接处理过程进行一个介绍，其它版本与 8.0 版本基本类似。

<!-- more -->

## 数据结构

在介绍 MySQL 建连过程之前，先整体看一下相关类的关系：

<img src="/images/connection-handler-1.jpg" width="88%"/>

MySQL 8.0 在 Linux 平台下使用的是 poll 机制，默认情况下每个用户连接会使用一个单独的线程进行处理。

## MySQL 建连过程

MySQL 建连过程可以分为以下几个步骤：

### 初始化

```c++
static Connection_acceptor<Mysqld_socket_listener> *mysqld_socket_acceptor = NULL;

static bool network_init(void) {
  ...
  /* 解析网络配置信息 */
  Mysqld_socket_listener *mysqld_socket_listener = new (std::nothrow)
      Mysqld_socket_listener(bind_addresses_info, mysqld_port,
                             admin_address_info, mysqld_admin_port,
                             listen_admin_interface_in_separate_thread,
                             back_log, mysqld_port_timeout, unix_sock_name);

  mysqld_socket_acceptor = new (std::nothrow)
      Connection_acceptor<Mysqld_socket_listener>(mysqld_socket_listener);
  ...
  /* 初始化监听端口 */
  if (mysqld_socket_acceptor->init_connection_acceptor())
    return true;
  ...
}

bool init_connection_acceptor() { return m_listener->setup_listener(); }

bool Mysqld_socket_listener::setup_listener() {
  ...
  if (m_tcp_port) {
    for (const auto &bind_address_info : m_bind_addresses) {
      TCP_socket tcp_socket(bind_address_info.address,
                            bind_address_info.network_namespace, m_tcp_port,
                            m_backlog, m_port_timeout);

      /* socket 初始化，底层调用的还是 socket/bind/listen 函数 */
      MYSQL_SOCKET mysql_socket = tcp_socket.get_listener_socket();
      if (mysql_socket.fd == INVALID_SOCKET) return true;

      Socket_attr s(Socket_type::TCP_SOCKET,
                    bind_address_info.network_namespace);
      m_socket_map.insert(
          std::pair<MYSQL_SOCKET, Socket_attr>(mysql_socket, s));
    }
  }

  /* 将所有监听的 socket 信息加入到 m_poll_info 中 */
  setup_connection_events(m_socket_map);
  return false;
}

/* socker 初始化 */
MYSQL_SOCKET TCP_socket::get_listener_socket() {
  ...
  MYSQL_SOCKET listener_socket = create_socket(ai_ptr.get(), AF_INET, &a);
  mysql_socket_bind(listener_socket, a->ai_addr, a->ai_addrlen);
  mysql_socket_listen(listener_socket, static_cast<int>(m_backlog);
  ...
}

void Mysqld_socket_listener::setup_connection_events(
    const socket_map_t &socket_map) {
#ifdef HAVE_POLL
  const socket_map_t::size_type total_number_of_addresses_to_bind =
      socket_map.size();
  m_poll_info.m_fds.reserve(total_number_of_addresses_to_bind);
  m_poll_info.m_pfs_fds.reserve(total_number_of_addresses_to_bind);
#endif

  for (const auto &element : socket_map) add_socket_to_listener(element.first);
}

void Mysqld_socket_listener::add_socket_to_listener(
    MYSQL_SOCKET listen_socket) {
  mysql_socket_set_thread_owner(listen_socket);

#ifdef HAVE_POLL
  m_poll_info.m_fds.emplace_back(
      pollfd{mysql_socket_getfd(listen_socket), POLLIN, 0});
  m_poll_info.m_pfs_fds.push_back(listen_socket);
#else
  FD_SET(mysql_socket_getfd(listen_socket), &m_select_info.m_client_fds);
  if ((uint)mysql_socket_getfd(listen_socket) >
      m_select_info.m_max_used_connection)
    m_select_info.m_max_used_connection = mysql_socket_getfd(listen_socket);
#endif
}
```

### 连接建立

```c++
int mysqld_main(int argc, char **argv) {
  ...
  mysqld_socket_acceptor->connection_event_loop();
  ...
}

void connection_event_loop() {
  Connection_handler_manager *mgr =
    Connection_handler_manager::get_instance();
  while (!connection_events_loop_aborted()) {
    /* 监听到连接事件，开始 channel 处理 */
    Channel_info *channel_info = m_listener->listen_for_connection_event();
    if (channel_info != NULL) mgr->process_new_connection(channel_info);
  }
}

Channel_info *Mysqld_socket_listener::listen_for_connection_event() {
#ifdef HAVE_POLL
  /* 通过 poll 机制进行监听 */
  int retval = poll(&m_poll_info.m_fds[0], m_socket_map.size(), -1);
#else
  m_select_info.m_read_fds = m_select_info.m_client_fds;
  int retval = select((int)m_select_info.m_max_used_connection,
                      &m_select_info.m_read_fds, 0, 0, 0);
#endif

  bool is_unix_socket = false, is_admin_sock;
  /* 获取到一个 ready 的 socket */
  MYSQL_SOCKET listen_sock = get_ready_socket(&is_unix_socket, &is_admin_sock);
  DBUG_ASSERT(mysql_socket_getfd(listen_sock) != INVALID_SOCKET);
  
  MYSQL_SOCKET connect_sock;
  accept_connection(listen_sock, &connect_sock);

  Channel_info *channel_info = NULL;
  if (is_unix_socket)
    channel_info = new (std::nothrow) Channel_info_local_socket(connect_sock);
  else
    channel_info = new (std::nothrow)
        Channel_info_tcpip_socket(connect_sock, is_admin_sock);
}

void Connection_handler_manager::process_new_connection(
    Channel_info *channel_info) {
  ...
  if (m_connection_handler->add_connection(channel_info)) {
    inc_aborted_connects();
    delete channel_info;
  }
}

/* 一对一处理模式 */
bool Per_thread_connection_handler::add_connection(Channel_info *channel_info) {
  ...
  /* 检查是否有空闲的线程 */
  if (!check_idle_thread_and_enqueue_connection(channel_info)) return false;
  
  channel_info->set_prior_thr_create_utime();
  /* 新建一个线程进行处理 */
  error =
      mysql_thread_create(key_thread_one_connection, &id, &connection_attrib,
                          handle_connection, (void *)channel_info);
  ...
  if (error) {
    connection_errors_internal++;
    if (!create_thd_err_log_throttle.log())
      LogErr(ERROR_LEVEL, ER_CONN_PER_THREAD_NO_THREAD, error);
    channel_info->send_error_and_close_channel(ER_CANT_CREATE_THREAD, error,
                                               true);

    im::global_manager_dec_connection(NULL);
    return true;
  }
  ...
}

static void *handle_connection(void *arg) {
  ...
  Channel_info *channel_info = static_cast<Channel_info *>(arg);
  if (my_thread_init()) {
    ...
  }
  ...
  for (;;) {
    THD *thd = init_new_thd(channel_info);
    ...
    while (thd_connection_alive(thd)) {
      /* 开始请求处理 */
      if (do_command(thd)) break;
    }
    ...
    /* thread 复用 */
    channel_info = Per_thread_connection_handler::block_until_new_connection();
  }
}
```

### 请求处理

```c++
bool do_command(THD *thd,
                std::function<bool(THD *, const COM_DATA *,
                                   enum enum_server_command)> *dispatcher) {
  ...
  thd->m_server_idle = true;
  rc = thd->get_protocol()->get_command(&com_data, &command);
  thd->m_server_idle = false;
  ...
  return_value = dispatch_command(thd, &com_data, command);
  ...
}

bool dispatch_command(THD *thd, const COM_DATA *com_data,
                      enum enum_server_command command) {
  ...
  thd->set_command(command);
  ...
  switch (command) {
    ...
    case COM_QUERY: {...}
    ...
  }
}
```

## 附

MySQL 5.7 可以参考：http://mysql.taobao.org/monthly/2016/07/04/