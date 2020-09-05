---
layout: post
title: "VSCode Remote: 서브디렉토리를 별도의 workspace로 띄우기"
date: 2020-09-05T23:39:00+09:00
categories: [vscode, git]
tags: [vscode,remote,docker,git]
---

한 프로젝트에 `backend`와 `frontend`가 있는데 이걸 각각 별도의 리모트 컨테이너로 띄우는게 목적이다.

## 서브프로젝트 분리해서 띄우기

일단 `docker-compose.yml`을 만들었다. \
개발에 사용할 디비라던가 이것저것 설정해주고, 각각의 서브프로젝트별 서비스를 추가해준다.

```yaml
version: '3.8'
services:
  db:
    image: postgres:12-alpine
    user: 1000:1000 # 디비를 먼저 생성하고 chmod 해줄것
    environment:
      TZ: Asia/Seoul
      POSTGRES_USER: [데이터 말소]
      POSTGRES_PASSWORD: [데이터 말소]
      POSTGRES_DB: [데이터 말소]
    volumes:
      - ./docker/db/data:/var/lib/postgresql/data
    ports:
      - 5432:5432
  backend:
    build:
      context: .
      dockerfile: ./docker/node.dockerfile
    command: sleep infinity
    user: node
    environment:
      TZ: Asia/Seoul
    volumes:
      - ~/.gitconfig:/home/node/.gitconfig:ro
      - ./.git:/workspace/.git
      - ./backend:/workspace/backend
    ports:
      - 4001:4000
      - 9222:9222
  frontend:
    build:
      context: .
      dockerfile: ./docker/node.dockerfile
    command: sleep infinity
    user: node
    environment:
      TZ: Asia/Seoul
    volumes:
      - ~/.gitconfig:/home/node/.gitconfig:ro
      - ./.git:/workspace/.git
      - ./frontend:/workspace/frontend
    ports:
      - 4000:4000
      - 9229:9229
```

그리고 각각의 서브프로젝트 디렉토리에 `.devcontainer.yml` 파일을 추가해서 컨테이너 설정을 해준다.

```json
{
  // 이름은 마음대로
  "name": "backend",
  "dockerComposeFile": ["../docker-compose.yml"],
  // 서비스 이름, 워크스페이스 디렉토리 이름 확인
  "service": "backend", 
  "workspaceFolder": "/workspace/backend",
  "settings": {
    // Alpine Linux 기반이 아니면 그냥 /bin/bash
    "terminal.integrated.shell.linux": "/bin/ash"
  },
  // 위에 설정한 서비스 이름과 추가로 필요한 서비스들만 여기 추가
  "runServices": ["db", "backend"],
  // 사용할 플러그인
  "extensions": ["dbaeumer.vscode-eslint", "prettier.prettier-vscode"]
}
```

그리고 각각의 서브디렉토리를 `Remote-Containers: Open Folder in Container...`로 열어주면 잘 열린다.

## 그리고 Git

그런데 말입니다...

분명 Git 저장소도 마운트가 되어있는데 (위 설정에서 `/workspace/.git`으로 되어있다.) `git`을 실행하면

```sh
/workspace/backend $ git status
fatal: not a git repository (or any parent up to mount point /workspace)
Stopping at filesystem boundary (GIT_DISCOVERY_ACROSS_FILESYSTEM not set).
```
모라고요?

그래서 각각의 서비스에 `GIT_DISCOVERY_ACROSS_FILESYSTEM` 환경변수를 추가해줘야 한다.

```diff
     environment:
       TZ: Asia/Seoul
+      GIT_DISCOVERY_ACROSS_FILESYSTEM: 1
     volumes:
```

그리고 컨테이너를 리빌드 하면 `git` 명령이 정상적으로 작동하게 된다.
