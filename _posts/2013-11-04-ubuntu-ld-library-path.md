---
layout: post
title: Ubuntu에서 LD_LIBRARY_PATH가 안먹을때
date: 2013-11-04T15:42:00+09:00
categories: tech
tags: [리눅스, 우분투, LD_LIBRARY_PATH, ldconfig, linux, ubuntu]
---

우분투에서는 `LD_LIBRARY_PATH`를 직접 설정하는 것을 권장하지 않는다.  
따라서 `/etc/environment`나 `/etc/profile`, `~/.bashrc`, `~/.profile` 같은 파일에 있는 `LD_LIBRARY_PATH` 항목들을 가볍게 씹어버린다.

그러면 `LD_LIBRARY_PATH` 없이 필요한 so 파일들을 어떻게 찾을 것인가?  
우분투에서는 `ldconfig`를 사용할 것을 권장하고 있다.

`/etc/ld.so.conf.d` 디렉토리에 `conf` 확장자를 가지는 적절한 파일을 추가한 다음에 그 파일에 `LD_LIBRARY_PATH`에 설정할 디렉토리 경로를 집어넣으면 된다.

예를 들어서, OCI (오라클 인스턴트 클라이언트) 라이브러리를 설정하려면 `/etc/ld.so.conf.d/` 디렉토리 안에 적절한 이름 (`oracle.conf` 같은) 을 가진 파일을 만들고 라이브러리의 경로를 추가한다. (64-bit rpm을 deb로 변환했을 경우 `/usr/lib/oracle/11.2/client64/lib`이 될 것이다.)


그 다음 `sudo ldconfig -v` 명령을 실행해서 설정을 업데이트 해 주면 된다.  
(추가로 이 명령은 `ld.so.conf`에 등록된 모든 so 파일들의 목록을 보여준다. 여기에 제대로 표시되면 정상적으로 등록된 것이다.)

정상적으로 인식되고 나면, 더이상 `LD_LIBRARY_PATH`를 쓸 필요가 없어진다. 만세!
