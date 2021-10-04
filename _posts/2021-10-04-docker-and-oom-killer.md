---
title: "Docker의 container가 별다른 log없이 죽을때, OOM Killer를 확인해보자"
date: 2021-10-04 21:00:00 +0900 
categories: Problem-Solving
tags: docker linux
---
## 들어가며
회사에서 운영하는 서비스를 위해, docker container로 scheduler를 띄어 놓았는데, 대략 일주일 간격으로 container가 죽는 일이 발생했다.
container가 바로 죽지 않는다는 점과 일주일정도는 지속된다는 점을 통해 memory 문제이지 않을까 의심이 들었다.

## 탐색
`docker stats` 명령으로 2~3일 간 지켜본 결과 아래와 같이 사용중인 memory가 점차 증가하는 것을 확인하였다.

![09-28 memory status](/assets/images/2021-10-04 09-28 memory.png)
9월 28일

![09-29 memory status](/assets/images/2021-10-04 09-29 memory.png)
9월 29일

![09-30 memory status](/assets/images/2021-10-04 09-30 memory.png)
9월 30일.
이때는 운이 좋게도 확인했을 때 container가 죽은지 3분정도 밖에 지나지 않아서 죽기 직전 정보를 얻을 수 있었다.

하지만 죽은 container의 log를 `docker logs container-name > container.log` 명령을
이용해 파일로 출력하여 확인해봐도 죽기 직전 메모리와 관련된 log를 찾아볼 수 없었다.
심지어 container와 관련된 log도 존재하지 않았고 그저 scheduler에서 출력한 내용만이 있었다.
scheduler가 직접 남긴 log에서도 에러와 관련된 내용을 찾을 수 없었다.
심지어 위의 사진에서 보듯이 memory의 limit에 도달하기 이전에 종료된 것이었다.

그러다 문득 docker가 죽인게 아니라면, OS에서 container를 종료시킨게 아닐까 생각이 들었다.
그래서 scheduler server에서 사용중인 OS, linux에 memory를 정리하는 무언가가 있는지 조사했다.

## 원인
OOM(Out Of Memory) Killer는 linux kernal이 사용하는 process로,
linux kernal은 사용 가능한 memory보다 많은 크기를 process에게 할당해 주는데,
여러 process가 한번에 많이 memory를 쓰게 되면 실제 사용할 수 있는 memory가 없어 OOM이 발생하게 된다.
이때 OOM Killer가 각 process 별로 평가한 점수에 따라 강제로 종료하고 memory를 확보하는 것이다.

실제로 `/var/log/syslog`에서 다음과 같은 log를 찾을 수 있었다.
```text
Sep 30 13:00:20 localhost kernel: [2343070.752072] docker invoked oom-killer: gfp_mask=0x100cca(GFP_HIGHUSER_MOVABLE), order=0, oom_score_adj=0
Sep 30 13:00:20 localhost kernel: [2343070.752081] CPU: 0 PID: 162339 Comm: docker Not tainted 5.11.0-1016-aws #17~20.04.1-Ubuntu
Sep 30 13:00:20 localhost kernel: [2343070.752084] Hardware name: Amazon EC2 t3.small/, BIOS 1.0 10/16/2017
Sep 30 13:00:20 localhost kernel: [2343070.752086] Call Trace:
Sep 30 13:00:20 localhost kernel: [2343070.752091]  dump_stack+0x74/0x92
Sep 30 13:00:20 localhost kernel: [2343070.752098]  dump_header+0x4f/0x1f6
Sep 30 13:00:20 localhost kernel: [2343070.752104]  oom_kill_process.cold+0xb/0x10
Sep 30 13:00:20 localhost kernel: [2343070.752108]  out_of_memory.part.0+0x1ee/0x460
Sep 30 13:00:20 localhost kernel: [2343070.752113]  out_of_memory+0x6d/0xd0
                                                  ...
Sep 30 13:00:20 localhost kernel: [2343070.752224] Mem-Info:
                                                  ...
Sep 30 13:00:20 localhost kernel: [2343070.752331] 3509 total pagecache pages
Sep 30 13:00:20 localhost kernel: [2343070.752333] 0 pages in swap cache
Sep 30 13:00:20 localhost kernel: [2343070.752334] Swap cache stats: add 0, delete 0, find 0/0
Sep 30 13:00:20 localhost kernel: [2343070.752336] Free swap  = 0kB
Sep 30 13:00:20 localhost kernel: [2343070.752336] Total swap = 0kB
Sep 30 13:00:20 localhost kernel: [2343070.752338] 514440 pages RAM
Sep 30 13:00:20 localhost kernel: [2343070.752339] 0 pages HighMem/MovableOnly
Sep 30 13:00:20 localhost kernel: [2343070.752340] 16508 pages reserved
Sep 30 13:00:20 localhost kernel: [2343070.752341] 0 pages hwpoisoned
Sep 30 13:00:20 localhost kernel: [2343070.752342] Tasks state (memory values in pages):
Sep 30 13:00:20 localhost kernel: [2343070.752342] [  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
                                                  ...
Sep 30 13:00:20 localhost kernel: [2343070.752434] [ 116935]     0 116935   666952   385476  3645440        0             0 python
                                                  ...
Sep 30 13:00:20 localhost kernel: [2343070.752451] [ 162333]  1001 162333   191738     6739   303104        0             0 docker
                                                  ...
Sep 30 13:00:20 localhost kernel: [2343070.752460] oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),cpuset=/,mems_allowed=0,global_oom,task_memcg=/docker/6141f63fdc9f5bca4ad9c5a98e50a8ad7588948f281668a09bb4f1309af7b059,task=python,pid=116935,uid=0
Sep 30 13:00:20 localhost kernel: [2343070.752481] Out of memory: Killed process 116935 (python) total-vm:2667808kB, anon-rss:1541904kB, file-rss:0kB, shmem-rss:0kB, UID:0 pgtables:3560kB oom_score_adj:0
Sep 30 13:00:20 localhost kernel: [2343070.853682] oom_reaper: reaped process 116935 (python), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```
이로써 OOM Killer에 의해 container가 죽은 것이 확인되었다.

## To Do
`docker logs`로 확인이 안되는 memory 문제를 만났다면 OS의 memory management process를 의심해봐야 한다는 점을 배웠다.

하지만 아직 근본적인 원인인 사용중인 memory가 왜 지속적으로 증가하는지에 대한 부분은 좀 더 조사가 필요하다.

일단 python의 tracemalloc을 사용하여 scheduler가 실행 되기 전과 실행한 후
각각 memory에서 큰 비중을 차지하는 상위 10개의 code 위치를 log에 출력하도록 설정해놓았다.
이로써 어느 부분에서 memory leak가 발생하는지 알아볼 계획이다.

OOM Killer와 linux의 memory management라는 공부하기 좋은 주제도 알게되었으니 따로 정리해보려 한다. 

## References
- [Linux OOM Killer by Ssup2](https://ssup2.github.io/theory_analysis/Linux_OOM_Killer/)
- [Out Of Memory Management](https://www.kernel.org/doc/gorman/html/understand/understand016.html)
