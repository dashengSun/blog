---
date: 2019-01-02
title: Cheatsheet for Common Usage CLI
---

## Curl Post JSON
```shell
curl -i -d '{"id": "111111","date": "2018-04-27T00:00:00+10:00","des": "test"}]' -H "Content-Type: application/json" -X POST https://endpoint

curl -i -d '{"a":"1","b":"2"}' -H "Content-Type: application/json" -X POST https://endpoint
```

## Database

### MySQL

```shell
SHOW VARIABLES;

SELECT FROM_UNIXTIME(1510289234);

SELECT NOW();

SELECT UNIX_TIMESTAMP();
```
### MongoDB

```shell
mongod --dbpath ~/data/mongodb
```

## Zip / tar

```
zip filename.zip file1 file2
```

## SSH / scp
```
// setup tunel
ssh -L 8002:target-host-ip:80 -i ~/.ssh/ssh-key hostname@host-ip

// from local to remote
scp /filename hostname@ip:~

// from remote to local
scp -i ~/.ssh/ssh-key root@host:~/hello ./
```

## Docker
```
// docker download file
docker cp <containerId>:/file/path/within/container /host/path/target
```

## Journalctl
```
// Display kernal message:
journalctl -k | grep -i -e memory -e oom
```

## Generate table of contents for markdown
```
gh-md-toc README.md
```

## Calculate lines of code in project
```
find . -name '*.scala' | xargs wc -l
```

## Docker override entrypoint

```
docker run -it --entrypoint /usr/bin/redis-cli example/redis
```

## Git 
**Reset local branch after force push:**

```shell
git reset origin/master --hard
```

**Rebase to certain commit, like change commit msg:**
```shell
git rebase -i commit-id // p to pick, r to change msg
```

** Delete branch with illegal name:**
```shell
git branch -D -- --track
```

## Nodejs npm mirror

```
npm config set registry=http://registry.npm.taobao.org
```
