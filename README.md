# egg-scripts--title
egg-scripts stop 缺失title参数情况可复现仓库。

## 隐患/Bug之误杀其他进程
egg-scripts stop 缺失title参数情况下，会停掉本机所有eggjs(严格讲是start-cluster，如果pm2启动一个入口js则不会)启动的服务。

## 最小可复现仓库
[仓库地址](https://github.com/diyao/egg-scripts--title)

### start多个服务
分别启动service1和service2
```bash
$ cd service1
$ npm i 
$ npm run start
$ cd ../service2
$ npm i 
$ npm run start
```
### stop任一服务
停掉service1或service2中的一个：
```bash
$ npm run stop
```
然后2个服务都停了。

## 原因
`egg-scripts stop`停止服务的过程是通过
```bash
ps -eo pid,args
```
列出大概如下进程信息：
```text
23756 /usr/local/bin/node /Users/mac/Desktop/www/github/egg-scripts--title/service1/node_modules/egg-cluster/lib/agent_worker.js {"framework":"/Users/mac/Desktop/www/github/egg-scripts--title/service1/node_modules/egg","baseDir":"/Users/mac/Desktop/www/github/egg-scripts--title/service1","workers":4,"plugins":null,"https":false,"title":"egg-server-service1","clusterPort":62486}
23757 /usr/local/bin/node /Users/mac/Desktop/www/github/egg-scripts--title/service1/node_modules/egg-cluster/lib/app_worker.js {"framework":"/Users/mac/Desktop/www/github/egg-scripts--title/service1/node_modules/egg","baseDir":"/Users/mac/Desktop/www/github/egg-scripts--title/service1","workers":4,"plugins":null,"https":false,"title":"egg-server-service1","clusterPort":62486}
```
然后过滤：
```javascript
let processList = yield this.helper.findNodeProcess(item => {
  const cmd = item.cmd;
  // 匹配egg-scripts start启动的进程，匹配不了pm2 start service.js启动的进程
  return argv.title ?
    cmd.includes('start-cluster') && cmd.includes(util.format(osRelated.titleTemplate, argv.title)) :
        cmd.includes('start-cluster');
});
```
由于只校验了子串`start-cluster`，而误杀其他进程。即便有`title`参数时，都会杀掉同`title`名进程，没有`title`参数时会杀掉所有`start-cluster`启动的进程。

如果校验带上`process.cwd()`，则只会对本项目而言。
[一个已验证的简单粗暴commit](https://github.com/diyao/egg-scripts/commit/b203d5a42798e98c1942808042afff112c34f44b)

## 隐患/Bug之列出未killed的进程有误
同上，校验进程子串`'egg-cluster\\lib\\agent_worker.js'`，而导致其他服务被列出。
```txt
[egg-scripts] stopping egg application 
[egg-scripts] got master pid ["5660"]
[egg-scripts] got worker/agent pids ["6423","6784","6785","6790","6798"] that is not killed by master
[egg-scripts] stopped
```
虽然上一个`stop`误杀其他进程，但是剩下由`pm2`启动一个入口js文件的方式没有被杀，而列出的那些信息让人以为进程kill失败（其实它列出的是`pm2`启动的那些进程）。究其原因如下：同上，过滤子串而不够彻底。
```javascript
// 不管是egg-scripts start 还是pm2 start service.js都会匹配。
const osRelated = {
  titleTemplate: isWin ? '\\"title\\":\\"%s\\"' : '"title":"%s"',
  appWorkerPath: isWin ? 'egg-cluster\\lib\\app_worker.js' : 'egg-cluster/lib/app_worker.js',
  agentWorkerPath: isWin ? 'egg-cluster\\lib\\agent_worker.js' : 'egg-cluster/lib/agent_worker.js',
};
processList = yield this.helper.findNodeProcess(item => {
  const cmd = item.cmd;
  return argv.title ?
    (cmd.includes(osRelated.appWorkerPath) || cmd.includes(osRelated.agentWorkerPath)) && cmd.includes(util.format(osRelated.titleTemplate, argv.title)) :
    (cmd.includes(osRelated.appWorkerPath) || cmd.includes(osRelated.agentWorkerPath));
});
```
```javascript
// pm2入口启动
'use strict';
const egg = require('egg');
const workers = 4;
// ...
egg.startCluster({
  workers,
  baseDir: __dirname,
});
```

## Context
- **Node Version**: 8.12.0
- **Egg Version**: 2.23.0
- **Plugin Name**: egg-scripts
- **Plugin Version**: 2.11.0
- **Platform**: ubuntu16.04
