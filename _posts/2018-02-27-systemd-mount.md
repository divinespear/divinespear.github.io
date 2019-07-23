---
layout: post
title:  "systemd로 파티션 마운트하기"
date:   2018-02-27 03:26:00 +0900
categories: tech
tags: [systemd, mount, 마운트, 파티션, samba, nfs]
---

# [`systemd.mount`](https://www.freedesktop.org/software/systemd/man/systemd.mount.html)

부팅시 samba하고 NFS가 자동으로 안붙어서 짜증났는데, 덕분에 쉽게 해결했다.  
autofs 보다는 확실히 편하지...

단, 파일명 붙일 때만 조심해야 한다.

귀찮으면 mount할 대상 경로를 가지고 `systemd-escape`를 돌려서 그걸로 파일명을 붙이면 되지롱.

## 예제

systemd 파일명과 `[Mount]`의 `Where` 항목을 잘 비교해보면 좋다.

### samba

mnt-192.168.1.7.mount

```systemd
[Unit]
Description=Samba 192.168.1.7

[Mount]
What=\\192.168.1.7\[데이터말소]
Where=/mnt/192.168.1.7
Type=cifs
Options=_netdev,user,uid=[데이터말소],gid=[데이터말소],rw,suid,credentials=/etc/samba/192.178.1.7

[Install]
WantedBy=multi-user.target
```
이 예제에서는 `username`과 `password`를 `credentials`에 지정한 파일로 분리했다.

### NFS

mnt-192.168.1.13.mount

```systemd
[Unit]
Description=NFS 192.168.1.13

[Mount]
What=192.168.1.13:[데이터말소]
Where=/mnt/192.168.1.13
Type=nfs
Options=_netdev,relatime,vers=3,rsize=65536,wsize=65536,namlen=255,hard,nolock,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.13,mountvers=3,mountport=32780,mountproto=udp,local_lock=all

[Install]
WantedBy=multi-user.target
```
