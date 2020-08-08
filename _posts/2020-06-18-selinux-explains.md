---
layout: post
title: SELinux 가지고 놀기
date: 2020-06-18T17:16:00+09:00
categories: [linux, security]
tags: [linux, redhat, centos, selinux, security]
---

SELinux가 기본으로 설치되는 레드햇/CentOS 계열을 기준으로 쓰는 글이다.

데비안/우분투 계열은 SELinux를 안깔아주기 때문에 직접 깔 게 아니면 신경 안써도 되느니라.<br>
(구글신의 신탁을 받아보면 데비안/우분투 계열에 SELinux를 설치하면 정신줄을 안드로메다로 보내버리는 결과가 나옴을 알 수 있다.)

## 기본적인 것들

SELinux 끄기, `ls -Z`, `chcon`, `restorecon` 같은거 말하는거다.

그냥 구글신을 영접해라. 자고로 바이트 낭비는 (특히 모바일에서는) 죄악이다.

## SELinux를 가지고 놀려면

일단 `semanage`가 필요하다.

없으면 `policycoreutils-python`을 설치한다.

## 웹 서비스 관련 (Apache, Nginx, ...)

### 프록시를 사용하려면

웹서버가 네트워크 연결을 할 수 있게 설정하면 된다.
```sh
setsebool -P httpd_can_network_connect 1
```

### PHP에서 직접 명령을 실행하려면

[`exec()`](https://www.php.net/manual/en/function.exec.php) 같은거 쓰고 싶을 때를 이야기 하는 것이다.
```sh
setsebool -P httpd_execmem 1
```

### 업로드시 Permission Denied (특히 PHP)

업로드 대상 디렉토리 및 그 하위 디렉토리에는 `httpd_user_rw_content_t` 가 적용되어야 한다.

이건 `chcon`으로 넣어주든지 (업로드 대상 경로를 모두 동일하게 강제할 수 있다면) `semanage`로 넣어주든지 알아서 하믄 된다.

```sh
chcon -Rv -t httpd_user_rw_content_t (디렉토리)
```

### 지정된 포트가 아닌 다른 포트를 열고 싶으면

기존에 포트가 다른 서비스로 등록되어있으면
```sh
semanage port -m -t http_port_t -p tcp 포트번호
```

등록되어있지 않은 포트면
```sh
semanage port -a -t http_port_t -p tcp 포트번호
```

등록되어있는건지 새로 등록해야하는 건지 모르겠으면 둘 중에 하나 잡고 해보면 된다.<br>
에러 뜨면 다른쪽 명령을 실행하면 되지롱.

기본으로 HTTP 포트로 등록된 포트 번호는 다음과 같다.

* `http_port_t`: 80, 81, 443, 488, 8008, 8009, 8443, 9000
* `http_cache_port_t`: 8080, 8118, 8123, 10001-10010

## 커스텀 규칙

### 가지고 놀기

현재 설정된 fcontext를 보고 싶으면
```sh
semanage fcontext -l
```

커스텀 설정을 추가하고 싶으면
```sh
semanage fcontext -a -t (타입) (디렉토리/파일 규칙)
```
디렉토리/파일 규칙은 정규표현식이다.

설정이 제대로 되었다면 `restorecon`을 시도해보자. 설정한 타입으로 변경될 것이다.


semanage 명령으로 추가한 커스텀 설정은
```sh
/etc/selinux/targeted/contexts/files/file_contexts.local
/etc/selinux/targeted/modules/active/file_contexts.local
```
이 두 군데 저장된다.<br>
룰만 정확하게 안다면 이 파일들을 직접 수정해도 된다. (주의: 둘의 내용은 동일해야한다.)

두번째 파일이 없다면 설정파일 동기화는 무시해도 된다.

### 다른 디렉토리를 `/home` 처럼 만들기

(아마 이걸 몰라서 다들 SELinux를 끄는가 싶은데...)

`/etc/selinux/targeted/contexts/files/file_contexts.homedirs` 파일을 본다.<br>
이게 `/home` 및 그 하위 디렉토리에 대한 SELinux 설정 파일이다.

이 파일을 저 위에 끄적인 `file_contexts.local` 파일들에 덧붙이고 `/home` 을 설정할 디렉토리로 바꿔주면 된다.<br>
(파일 내용이 동일해야 하므로, 하나를 고치고 나서 다른 하나를 엎어쓰면 된다. 참 쉽죠?)

일일히 고치기 귀찮은가? vi는 정규표현식 지원한다.

수정 완료하면 재부팅 안해도 타입 룰이 적용된다.

`restorecon -Rv (디렉토리)` 를 해주자. 아주 기분이 HIGH해질 것이다.
