## Linux IPC
1. Data Transfer
* Byte Stream
    - Pipe
    - Named Pipe(FIFO)
    - Socket(stream)
* Message
    - POSIX Message Queue
    - SysV Message Queue
    - Socket(datagram)

2. Shared Memory
* File-Memory mapping
    - Anonymous mapping
    - File mapping
* Shared memory
    - SysV shared memory
    - POSIX shared memory

3. Synchronization
* Semaphore
    - POSIX semaphore
    - SysV semaphore
* File lock
    - file lock
    - record lock

## Pipe
- Uni-directional byte stream
- name or ID가 없음
- related process 간에 사용 가능(e.g. fork())

(Parent) Process --write-->(fd[1]) Pipe (fd[0]) --read--> (Child) Process

사용하지 않을 fd(file descriptor)를 close

```c
// Pipe API
int pipe(int pipefd[2]);
```

### Pipe I/O
- pipe가 full 일 때 write 시도하면 blocking
- pipe가 empty 일 떄 read 시도하면 blocking
- write size가 PIPE_BUF보다 작으면 atomic, 크면 interleaved 될 수 있음.
    - Linux PIPE_BUF: 4KB
    - multiple writer 환경에서 유의해야 함

### example
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(int argc, char **argv)
{
    int pipe_fds[2];
    pid_t pid;
    char buf[1024];
    int wstatus;

    printf("[%d] start of function\n", getpid());
    memset(buf, 0, sizeof(buf));

    if (pipe(pipe_fds))
    {
        perror("pipe()");
        return -1;
    }

    pid = fork();
    if (pid == 0)
    {
        /* child process */
        close(pipe_fds[1]);
        read(pipe_fds[0], buf, sizeof(buf));
        printf("[%d] parent said... %s\n", getpid(), buf);
        close(pipe_fds[0]);
    }
    else if (pid > 0)
    {
        /* parent process */
        close(pipe_fds[0]);
        strncpy(buf, "hello child~", sizeof(buf) - 1);
        write(pipe_fds[1], buf, strlen(buf));
        close(pipe_fds[1]);

        pid = wait(&wstatus);
    }
    else
    {
        /* error case */
        perror("fork()");
        goto err;
    }
    return 0;

err:
    close(pipe_fds[0]);
    close(pipe_fds[1]);
    return -1;
}
```

## Named Pipe (FIFO)
- Uni-directional byte stream
- file path가 ID (unrelated process 간에도 사용 가능)
- FIFO 생성과 open이 분리되어 있음 (open을 별도로 호출해야 함)
- open() 시 read-side와 write-side가 동기화 됨
    * 양쪽 모두 open시도가 있어야 성공
    * open 시 O_NONBLOCK이 유용하게 사용될 수 있음

process --write--> Named Pipe ("/tmp/my_fifo") --read--> Process

```c
// Named Pipe API
int mkfifo(const char *pathname, mode_t mode);
```

### example
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>

#define FIFO_FILENAME "./testfifo"

static void print_usage(char *progname)
{
    printf("%s (w|r)\n", progname);
    return;
}

static int do_reader(void)
{
    int fd;
    char buf[128];

    printf("call open()\n");
    fd = open(FIFO_FILENAME, O_RDONLY);
    if (fd < 0)
    {
        perror("open()");
        return -1;
    }
    printf("call read()\n");
    read(fd, buf, sizeof(buf));
    printf("writer said...%s\n", buf);
    close(fd);

    return 0;
}

static int do_writer(void)
{
    int fd;
    char buf[128];

    printf("make fifo\n");
    unlink(FIFO_FILENAME);
    if (mkfifo(FIFO_FILENAME, 0644))
    {
        perror("mkfifo()");
        return -1;
    }

    printf("call open()\n");
    fd = open(FIFO_FILENAME, O_WRONLY);
    if (fd < 0)
    {
        perror("open()");
        return -1;
    }
    printf("call strncpy()\n");
    strncpy(buf, "hello", sizeof(buf));
    printf("call write()\n");
    write(fd, buf, strlen(buf));
    close(fd);
    return 0;
}

int main(int argc, char **argv)
{
    if (argc < 2)
    {
        print_usage(argv[0]);
        return -1;
    }

    if (!strcmp(argv[1], "r"))
    {
        /* reader */
        do_reader();
    }
    else if (!strcmp(argv[1], "w"))
    {
        /* writer */
        do_writer();
    }
    else
    {
        print_usage(argv[0]);
        return -1;
    }

    return 0;
}
```

## Message Queue
* Message 기반 communication
    - byte stream이 아님
    - 하나의 message는 하나의 덩어리처럼 처리됨
* FIFO(First-in, First-out)를 이용한 message queue
    - unrelated process 간에도 사용 가능

Process A --send--> | Message Queue | --recv--> Process B

## SysV Message Queue
* Message 기반 communication
    - message 별 message type 지원
* Message queue
    - FIFO
    - IPC Key 기반 Identifier(not fd)
* Management tools
    - ipcs: IPC object 관련 정보 조회
    - ipcrm: IPC object 삭제


### SysVMQ APIs
#### msgget
```c
int msgget(key_t key, int msgflg);
```
- message queue ID를 구함
- 옵션에 따라 생성도 가능

parameters
- key: IPC object key or IPC_PRIVATE
  - IPC_PRIVATE: 지정 시 새로운 message queue ID 생성
- msgflg: permission + mask
  - IPC_CREAT: key에 해당하는 message queue ID 없으면 생성
  - IPC_EXCL(exclusive): key에 해당하는 message queue ID가 있으면 error

return value
- 성공 시 message queue ID return. integer type이긴 하나 file descriptor가 아니기 떄문에 관련 operation은 불가함
- 실패 시 -1

#### ftok (file to key)
```c
key_t ftok(const char *pathname, int proj_id);
```
- filepath와 proj_id를 조합하여 key 값을 구함
- best effort. unique 보장은 안됨

parameters
- pathname
    - 조합할 파일 경로
    - 파일이 존재해야 하고, readable 해야 함
- proj_id
    - 임의의 project ID

return value
- 성공 시 IPC key return
- 실패 시 -1

#### msgsnd (message send)
```c
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);

struct msgbuf {
    long mtype;  /* message type, must be > 0 */
    char mtext[1];  /* message data */
}
```

parameters
- msqid: message queue id
- msgp: 전송할 message buffer
- msgsz: 전송 message size (mtext의 길이, in bytes)
- msgflg
    - IPC_NOWAIT: non-blocking I/O

return value
- 성공 시 0, 실패 시 -1

#### msgrcv (message receive)
```c
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);
```

parameters
- msqid: message queue id
- msgp: message 수신 buffer
- msgsz: 최대 수신 size (mtext의 길이, in bytes)
- msgtyp: 수신할 message type
    - 0: message type에 상관없이 첫번째 message 수신
    - positive: message type에 match되는 첫 번째 message 수신
    - negative: 지정된 절대값보다 작거나 같은 message type에 match되는 첫 번째 message 수신
- msgflg
    - IPC_NOWAIT: non-blocking I/O
    - MSG_COPY:
        - n번째 message를 복사해서 수신(msgtyp이 index로 사용됨)
        - 반드시 IPC_NOWAIT과 함께 사용해야 함
    - MSG_EXCEPT: msgtyp과 match되지 않는 message 수신
    - MSG_NOERROR: message size가 msgsz보다 크다면 truncate

return value
- 성공 시 실제 받은 data 길이(mtext의 길이, in bytes)
- 실패 시 -1

#### msgctl (message control)
```c
int msgctl(int msqid, int cmd, struct msqid_ds *buf);

struct msqid_ds {
    struct ipc_perm msg_perm;   /* Ownership and permissions */
    time_t          msg_stime;  /* Time of last msgsnd(2) */
    time_t          msg_rtime;  /* Time of last msgrcv(2) */
    time_t          msg_ctime;  /* Time of creation or last
                                    modification by msgctl() */
    unsigned long   msg_cbytes; /* # of bytes in queue */
    msgqnum_t       msg_qnum;   /* # number of messages in queue */
    msglen_t        msg_qbytes; /* Maximum # of bytes in queue */
    pid_t           msg_lspid;  /* PID of last msgsnd(2) */
    pid_t           msg_lrpid;  /* PID of last msgrcv(2) */
};

struct ipc_perm {
    key_t          __key;       /* Key supplied to msgget(2) */
    uid_t          uid;         /* Effective UID of owner */
    gid_t          gid;         /* Effective GID of owner */
    uid_t          cuid;        /* Effective UID of creator */
    gid_t          cgid;        /* Effective GID of creator */
    unsigned short mode;        /* Permissions */
    unsigned short __seq;       /* Sequence number */
};
```
message queue control

parameters
- msqid: message queue id
- cmd
    - IPC_STAT: kernel의 msgid_ds 정보 조회
    - IPC_SET: kernel의 msgid_ds에 설정
    - IPC_RMDI: message queue 제거
- buf

return value
- 성공 시 0
- 실패 시 -1

#### example
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

#define IPC_KEY_FILENAME "/proc"
#define IPC_KEY_PROJ_ID 'a'

struct msgbuf
{
    long mtype;
#define MSGBUF_STR_SIZE 64
    char string[MSGBUF_STR_SIZE];
};

static void print_usage(const char *progname)
{
    printf("%s (send|recv) MTYPE\n", progname);
}

static int init_msgq(void)
{
    int msgq;
    key_t key;

    key = ftok(IPC_KEY_FILENAME, IPC_KEY_PROJ_ID);
    if (key == -1)
    {
        perror("ftok()");
        return -1;
    }

    msgq = msgget(key, 0644 | IPC_CREAT);
    if (msgq == -1)
    {
        perror("msgget()");
        return -1;
    }

    return msgq;
}

static int do_send(long mtype)
{
    int msgq;
    struct msgbuf mbuf;

    msgq = init_msgq();
    if (msgq == -1)
    {
        perror("init_msgq()");
        return -1;
    }

    memset(&mbuf, 0, sizeof(mbuf));
    mbuf.mtype = mtype;
    snprintf(mbuf.string, sizeof(mbuf.string), "hello world mtype %ld", mtype);
    if (msgsnd(msgq, &mbuf, sizeof(mbuf.string), 0) == -1)
    {
        perror("msgsnd()");
        return -1;
    }
    return 0;
}

static int do_recv(long mtype)
{
    int msgq, ret;
    struct msgbuf mbuf;

    msgq = init_msgq();
    if (msgq == -1)
    {
        perror("init_msgq()");
        return -1;
    }

    memset(&mbuf, 0, sizeof(mbuf));
    ret = msgrcv(msgq, &mbuf, sizeof(mbuf.string), mtype, 0);
    if (ret == -1)
    {
        perror("msgrcv()");
        return -1;
    }
    printf("received msg: mtype %ld, msg [%s]\n", mbuf.mtype, mbuf.string);

    return 0;
}

int main(int argc, char **argv)
{
    int ret;
    long mtype;

    if (argc < 2)
    {
        print_usage(argv[0]);
        return -1;
    }

    mtype = strtol(argv[2], NULL, 10);

    if (!strcmp(argv[1], "send"))
    {
        if (mtype <= 0)
        {
            print_usage(argv[0]);
            return -1;
        }
        ret = do_send(mtype);
    }
    else if (!strcmp(argv[1], "recv"))
    {
        ret = do_recv(mtype);
    }
    else
    {
        print_usage(argv[0]);
        return -1;
    }

    return ret;
}
```

`ipcs -q`를 통해 queue의 status를 확인할 수 있다.