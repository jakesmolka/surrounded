---
title: Docker + Go
#author: Jake
date: '2017-11-20'
categories:
  - DevOps
tags:
  - Docker
  - Golang
slug: docker-and-go
---

# Docker + Go

The last days I was working on deploying a Golang program via docker. While this seemed to be an easy task in theory I didn't spend much time 
preparing it. When the time came to finally deploy the program it just didn't work.

Most online resources use some `Dockerfile` like [1]:
```
FROM scratch
COPY ./hello /hello
ENTRYPOINT ["/hello"]
```

And in fact that's an improvement to my usual `FROM alpine` which I gladly accept as unnecessary. But of course that shouldn't work in my 
case. Whatever I tried there was almost an error like:
```
standard_init_linux.go:195: exec user process caused "no such file or directory"
```

Since that kind of error message isn't really helpful I was poking in the dark and tried to find the solution with modifying every line of 
that `Dockerfile` with adding/removing '.', '/' and so on. After realizing that only further research might help I finally came across the 
missing link of information [2]: 

> Huh? What does that mean? Took me a while to figure it out, but our Go binary is looking for libraries on the operating system it’s running 
in. We compiled our app, but it still is dynamically linked to the libraries it needs to run (i.e., all the C libraries it binds to). 
Unfortunately, scratch is empty, so there are no libraries and no loadpath for it to look in. What we have to do is modify our build script to 
statically compile our app with all libraries built in:

What a relief, I thought. After statically linking my Go binary another error popped up.
```
time="2017-11-20T16:39:02Z" level=fatal msg="user: Current not implemented on linux/amd64"
```

That was definitively better than `no such file` but I was clueless again. A quick research was able to help me out. The `user.Current()` 
function used in my code was not able to be linked that way on purpose [3]. Already too much time spend on that task that was supposed to be 
easy I decided to go with the following `Dockerfile` and stay with dynamically linking:
```
FROM frolvlad/alpine-glibc
ADD app /
EXPOSE 8888
CMD ["/app"]
```

It uses an Alpine image with integrated libc support which is about 12MB [4]. That way my app was finally running, or more correct, it was 
finally able to try to connect to its database which fails because my next task is writing the `docker-compose.yml` to bring them together.

## Resources
- [1] https://blog.docker.com/2016/09/docker-golang/
- [2] https://blog.codeship.com/building-minimal-docker-containers-for-go-applications/
- [3] https://stackoverflow.com/questions/20609415/cross-compiling-user-current-not-implemented-on-linux-amd64#20610861
- [4] https://hub.docker.com/r/frolvlad/alpine-glibc/
