cas <-> server의 구조와 request를 처리하는 방법에 대해 확인합니다.

<br/>

중심 내용
1) request/session 각각이 스레드를 가지지 않습니다.
2) 모든 처리는 task 단위로 이루어지며, 이 task들은 css_Connection_worker_pool이라는 worker pool에서 처리됩니다.

<br/>

cas에서 연결 요청이 들어오면 server의 master thread에서 poll을 사용하여, 소켓 연결을 처리합니다. master 연결에 대해 아래 코드처럼 request 별로 처리를 진행합니다.

`server_support.c: 532`
```c
switch (request)
{
  case SERVER_START_NEW_CLIENT:
    css_process_new_client (master_fd);
    break;

  case SERVER_START_SHUTDOWN:
    css_process_shutdown_request (master_fd);
    r = 0;
    break;

    ...
```

새 클라이언트의 연결 요청인 경우 css_process_new_client를 호출하여 최종적으로 connection handler를 호출하게 됩니다.

connection handler는 아래 함수로 등록되어 있습니다. CSS_CONN_ENTRY * conn을 통해서 connection 정보 및 request 정보를 관리합니다.

`server_support.c: 1179`
```c
static css_internal_connection_handler (CSS_CONN_ENTRY * conn)
```

호출된 connection handler (위 함수)는 css_Connection_worker_pool에 request에 대한 task를 생성합니다.

이후 worker pool 내부에서 task의 execute를 호출하여 request 처리가 진행되기 때문에, 여기까지가 하나의 흐름이 됩니다.

생성된 task는 차례가 오면 worker가 실행하게 됩니다. 즉, worker thread는 request task를 실행한 순간 해당 request에 대한 스레드로 취급됩니다.

task가 처음 실행되면 thread의 entry에 필요한 정보를 설정하고, 이후 internal request handler를 호출합니다. internal request handler는 아래와 같습니다.

`server_support.c: 1203`
```c
static int css_internal_request_handler (THREAD_ENTRY & thread_ref, CSS_CONN_ENTRY & conn_ref)
```

request handler는 thread의 entry에 필요한 기본적인 정보를 설정합니다. 아래는 해당 코드들입니다.
```c
thread_p->client_id = client_id;
thread_p->rid = rid;
thread_p->tran_index = tran_index;
thread_p->net_request_index = net_request_index;
thread_p->victim_request_fail = false;
thread_p->next_wait_thrd = NULL;
thread_p->wait_for_latch_promote = false;
thread_p->lockwait = NULL;
thread_p->lockwait_state = -1;
thread_p->query_entry = NULL;
thread_p->tran_next_wait = NULL;
```

이 경우에는 client_id, rid, net_request_index를 제외하고는 아직 유효한 정보는 크게 없습니다. 사실 상의 초기화 과정입니다.

초기화가 끝났으면 실제 request를 처리하는 request handler를 호출합니다. 

`network_sr.c: 1027`
```c
static int net_server_request (THREAD_ENTRY * thread_p, unsigned int rid, int request, int size, char *buffer)
```

위 함수는 크게 세 부분으로 나눌 수 있습니다. 각각 특수한 경우인 handshake와 shutdown, 이외의 request 처리 부분이 됩니다.

이 함수에서 request 번호에 따라 에서 무수히 많이 나열된 net_Requests 배열 중 하나를 호출하게 됩니다.

`network_sr.c: 100`
```c
/* ping */
req_p = &net_Requests[NET_SERVER_PING];
req_p->processing_function = server_ping;
req_p->name = "NET_SERVER_PING";

/* boot */
req_p = &net_Requests[NET_SERVER_BO_INIT_SERVER];
req_p->processing_function = sboot_initialize_server;
req_p->name = "NET_SERVER_BO_INIT_SERVER";

req_p = &net_Requests[NET_SERVER_BO_REGISTER_CLIENT];
req_p->processing_function = sboot_register_client;
req_p->name = "NET_SERVER_BO_REGISTER_CLIENT";

req_p = &net_Requests[NET_SERVER_BO_UNREGISTER_CLIENT];
req_p->processing_function = sboot_notify_unregister_client;
req_p->name = "NET_SERVER_BO_UNREGISTER_CLIENT";

...
```

이후의 처리는 request 번호에 따라 갈리게 됩니다. 따라서 이 부분부터는 배열에 올라가있는 함수를 분석하는 것으로 이후 과정을 확인할 수 있습니다.

추가적인 내용은 session에서 정리했습니다.