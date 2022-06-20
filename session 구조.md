아래는 필요한 부분만 간략하게 나타낸 구성들과 그 구성들의 멤버들입니다.

```c
                              SERVER
CAS                <->            THREAD (entry)
                                      index (int)
                                      client_id (int)
                                      tran_index (int)
                                      tran_index_lock (pthread_mutex_t)
                                      conn_entry (css_conn_entry *)
                                          fd (SOCKET)
                                          client_id (int)
                                          idx (int, connection index)
                                          session_p (SESSION_STATE *)
                                              active_time (time_t)
                                          session_id (SESSION_ID = unsigned int)
```

cas-server 처리 구조에 정리된 것처럼 request는 worker를 통해 처리됩니다.

session은 어떻게 관리되고 어떤 구조를 가지고 있는 지 확인합니다.

<br/>

중심 내용
1) 전역으로 선언되어 사용되는 sessions 변수는 모든 session에 대한 정보를 가지고 있습니다.
2) session은 hashmap에서 정보의 형태로 관리되다가 필요할 때 thread에 세팅되어 사용됩니다.

<br/>

```c++
using session_hashmap_type = cubthread::lockfree_hashmap<SESSION_ID, session_state>;

typedef struct active_sessions
{
  session_hashmap_type states_hashmap;
  SESSION_ID last_session_id;

  ...

} ACTIVE_SESSIONS;

static ACTIVE_SESSIONS sessions;
```

위처럼 활성화된 모든 session에 대한 정보들은 ACTIVE_SESSIONS 구조체로 선언된 sessions라는 정보에 저장됩니다.

session 정보는 hashmap에 `SESSION_ID - SESSION_STATE` 형태로 저장됩니다. 때문에 session id를 통해 필요한 session을 불러올 수 있습니다.

```c
typedef struct session_state SESSION_STATE;
struct session_state
{
  SESSION_ID id;		/* session id */
  SESSION_STATE *stack;		/* used in freelist */
  SESSION_STATE *next;		/* used in hash table */
  pthread_mutex_t mutex;	/* state mutex */
  UINT64 del_id;		/* delete transaction ID (for lock free) */

  bool auto_commit;
  SESSION_VARIABLE *session_variables;
  PREPARED_STATEMENT *statements;
  SESSION_QUERY_ENTRY *queries;
  time_t active_time;

  ...

  session_state ();
  ~session_state ();
};
```

SESSION_STATE는 위와 같은 멤버를 가지고 있고, 사실상 SESSION STATE를 하나의 session이라고 볼 수 있습니다.

<br/>

---

<br/>

#### 세션 시작

session의 시작은 cas에서 서버와 연결된 이후 생성됩니다.

cas가 실행되고 process_request를 거쳐 csession_find_or_create_session를 호출하면 cas와 server의 master는 `cas-server 처리 구조`에서 설명한 과정을 모두 진행합니다.

이후 request에 따라 net_Requests 배열에서 등록된 함수를 호출합니다.

cas에서 실행한 csession_find_or_create_session 함수는 `NET_SERVER_SES_CHECK_SESSION` request를 보내기 때문에 아래 함수가 실행됩니다.

```c
/* session state */
req_p = &net_Requests[NET_SERVER_SES_CHECK_SESSION];
req_p->processing_function = ssession_find_or_create_session;
req_p->name = "NET_SERVER_SES_CHECK_SESSION";
```

따라서 server는 cas에서 보낸 client 정보를 entry로 ssession_find_or_create_session 함수를 실행합니다.

ssession_find_or_create_session 함수는 sessions.hashmap에서 request를 보낸 client에 대한 session을 찾아보고, 없다면 생성합니다.

만약 client에 대한 session이 이미 존재한다면, session의 active_time을 현재 시각으로 설정해줍니다.

<br/>

#### 세션 제어 데몬
`session.c: 229`
```c
static cubthread::daemon *session_Control_daemon
```

sessions가 활성화된 session만 가질 수 있도록 정리하는 데몬입니다. 주기(looper)로 60초를 가집니다.

이 daemon은 sessions.states_hashmap을 순회하며 session의 active_time이 현재 시각으로부터 191초 이상 차이나는 모든 세션들을 제거합니다.

주기가 60초이기 때문에 무조건 191초에 제거되지는 않습니다.

<br/>

#### 세션 생성 
`session.c: 638`
```c
int session_state_create (THREAD_ENTRY * thread_p, SESSION_ID * id)
```

이 함수는 새 session을 생성합니다. sessions.next_session_id + 1한 값을 session id, 즉 key로 사용하여 새로운 session_state와 states_hashmap에 삽입합니다.

session의 active_time을 현재 시각으로 설정합니다.

<br/>

#### 세션 체크
`session.c: 811`
```c
int session_check_session (THREAD_ENTRY * thread_p, const SESSION_ID id)
```

session의 시작에서 ssession_find_or_create_session가 호출되었을 때 session에 대해 몇 가지 확인을 하게 됩니다.

`ssession_find_or_create_session ( ... )`
```c
if (id == DB_EMPTY_SESSION
    || memcmp (server_session_key, xboot_get_server_session_key (), SERVER_SESSION_KEY_SIZE) != 0
    || xsession_check_session (thread_p, id) != NO_ERROR)
```

이때 이전 조건들이 만족되어 session이 empty하지 않다면 `xsession_check_session ( ... )` 함수를 통해 session이 유효한 지 확인하게 됩니다.

`call stack`
```
=> session_check_session
=> xsession_check_session
ssession_find_or_create_session
```

session이 유효하다면, 즉 active session storage인 sessions.session_hashmap에서 찾을 수 있다면 해당 session의 active_time를 현재 시각으로 업데이트합니다.

session을 sessions.session_hashmap에서 찾을 수 없다면, `ER_SES_SESSION_EXPIRED`를 반환하여 세션이 유효하지 않음을 알립니다.

`ssession_find_or_create_session ( ... )`
```c
if (id == DB_EMPTY_SESSION
    || memcmp (server_session_key, xboot_get_server_session_key (), SERVER_SESSION_KEY_SIZE) != 0
    || xsession_check_session (thread_p, id) != NO_ERROR)
  {
    /* not an error yet */
    er_clear ();
    /* create new session */
    error = xsession_create_new (thread_p, &id);

    ...
```

session이 유효하지 않다면 if 안쪽이 실행되어 새 session을 생성하는 `xsession_create_new ( ... )`를 실행하고

`call stack`
```
=> session_state_create
xsession_create_new
```

최종적으로 위에서 살펴본 `session_state_create ( ... )` 함수를 호출하게 됩니다.