---
layout: post
title: 업그레이드 런다운에 걸린 MongoDB 복구하기
date: 2020-08-09T06:38:00+09:00
categories: database
tags: [db, database, mongodb, recovery]
---

사내 소통 도구로 [Rocket.Chat](https://rocket.chat)를 구축해서 유용하게 써먹고 있는데, 이게 [MongoDB](https://www.mongodb.com)를 사용한다.

`docker-compose`로 Docker 컨테이너에 올려서 사용중인데... \
업데이트 돌리다가 디비가 깨지는 사태가 발생했다. (물논 제대로 설정 안하고 돌린 내 탓이지만)

난 아무 생각 없이 MongoDB의 이미지를 latest로 받았을 뿐이고... \
업데이트를 위해서 이미지 pull을 받는데 MongoDB가 조용히 4.4로 업데이트 되면서 모든 문제가 시작되었다.

**막간 캠페인: Docker 돌릴 때는 이미지 버전을 확인하고 돌립시다.**

아무 생각 없이 MongoDB가 4.4로 업데이트된 후 컨테이너를 올리면 다음과 같은 메세지가 당신의 멘탈을 박살낼 것이다.

```json
{"t":{"$date":"2020-08-09T07:06:46.356+09:00"},"s":"F",  "c":"CONTROL",  "id":20573,   "ctx":"initandlisten","msg":"Wrong mongod version","attr":{"error":"UPGRADE PROBLEM: Found an invalid featureCompatibilityVersion document (ERROR: BadValue: Invalid value for version, found 4.0, expected '4.4' or '4.2'. Contents of featureCompatibilityVersion document in admin.system.version: { _id: \"featureCompatibilityVersion\", version: \"4.0\" }. See https://docs.mongodb.com/master/release-notes/4.4-compatibility/#feature-compatibility.). If the current featureCompatibilityVersion is below 4.2, see the documentation on upgrading at https://docs.mongodb.com/master/release-notes/4.4/#upgrade-procedures."}}
```

그래서 다시 MongoDB를 4.2로 내리고 컨테이너를 돌렸을 때 재수가 없을 경우 다음과 비슷한 메세지가 당신의 멘탈을 확인사살 할 것이다.

```log
2020-04-15T16:35:10.362-0400 E STORAGE  [initandlisten] WiredTiger error (-31802) [1586982910:362655][25914:0x7f8b6f422a80], connection: __log_open_verify, 985: unsupported WiredTiger file version: this build only supports versions up to 4, and the file is version 5: WT_ERROR: non-specific WiredTiger error Raw: [1586982910:362655][25914:0x7f8b6f422a80], connection: __log_open_verify, 985: unsupported WiredTiger file version: this build only supports versions up to 4, and the file is version 5: WT_ERROR: non-specific WiredTiger error
```

여기 멘탈을 복구할 특효약이 있다.

```bash
mongod --repair
```

repair 작업이 완료된 후에 Replication 구성이 되어있었다면 기존거 지우고 다시 생성해주면 된다.
