---
title: 使hexo自动部署把markdown文章备份到git仓库
date: 2017-11-18 21:14:54
tags:
---

> 由于wordpress对markdown的支持不是很友好，看到这款hexo主题后我开始回归使用hexo
> hexo 有个很友好的module 就是 hexo-deployer-git，这个插件可以一键生成和上传静态网页
> 但是我希望我的文章markdown可以一同备份 到git仓库



安装部署插件
===

```
npm install hexo-deployer-git --save #
```

_config.yml
---
```
deploy:
  type: git
  repo: git@github.com:wongkimhung/wongkimhung.github.io.git
  branch: master
```

修改部署源码
===

找到项目下 **node_modules/hexo-deployer-git/lib/deployer.js**

替换源码
---
```javascript
'use strict';

var pathFn = require('path');
var fs = require('hexo-fs');
var chalk = require('chalk');
var swig = require('swig');
var moment = require('moment');
var Promise = require('bluebird');
var spawn = require('hexo-util/lib/spawn');
var parseConfig = require('./parse_config');

var swigHelpers = {
  now: function(format) {
    return moment().format(format);
  }
};

module.exports = function(args) {
  var baseDir = this.base_dir;
  var deployDir = pathFn.join(baseDir, '.deploy_git');
  var publicDir = this.public_dir;
  var extendDirs = args.extend_dirs;
  var sourceDir = this.source_dir;
  var ignoreHidden = args.ignore_hidden;
  var ignorePattern = args.ignore_pattern;
  var log = this.log;
  var message = commitMessage(args);
  var verbose = !args.silent;

  if (!args.repo && process.env.HEXO_DEPLOYER_REPO) {
    args.repo = process.env.HEXO_DEPLOYER_REPO;
  }

  if (!args.repo && !args.repository) {
    var help = '';

    help += 'You have to configure the deployment settings in _config.yml first!\n\n';
    help += 'Example:\n';
    help += '  deploy:\n';
    help += '    type: git\n';
    help += '    repo: <repository url>\n';
    help += '    branch: [branch]\n';
    help += '    message: [message]\n\n';
    help += '    extend_dirs: [extend directory]\n\n';
    help += 'For more help, you can check the docs: ' + chalk.underline('http://hexo.io/docs/deployment.html');

    console.log(help);
    return;
  }

  function git() {
    var len = arguments.length;
    var args = new Array(len);

    for (var i = 0; i < len; i++) {
      args[i] = arguments[i];
    }

    return spawn('git', args, {
      cwd: deployDir,
      verbose: verbose
    });
  }

  function setup() {
    var userName = args.name || args.user || args.userName || '';
    var userEmail = args.email || args.userEmail || '';

    // Create a placeholder for the first commit
    return fs.writeFile(pathFn.join(deployDir, 'placeholder'), '').then(function() {
      return git('init');
    }).then(function() {
      return userName && git('config', 'user.name', userName);
    }).then(function() {
      return userEmail && git('config', 'user.email', userEmail);
    }).then(function() {
      return git('add', '-A');
    }).then(function() {
      return git('commit', '-m', 'First commit');
    });
  }

  function push(repo) {
    return git('add', '-A').then(function() {
      return git('commit', '-m', message).catch(function() {
        // Do nothing. It's OK if nothing to commit.
      });
    }).then(function() {
      return git('push', '-u', repo.url, 'HEAD:' + repo.branch, '--force');
    });
  }

  return fs.exists(deployDir).then(function(exist) {
    if (exist) return;

    log.info('Setting up Git deployment...');
    return setup();
  }).then(function() {
    log.info('Clearing .deploy_git folder...');
    return fs.emptyDir(deployDir);
  }).then(function() {
    var opts = {};
    log.info('Copying files from public folder...');
    if (typeof ignoreHidden === 'object') {
      opts.ignoreHidden = ignoreHidden.public;
    } else {
      opts.ignoreHidden = ignoreHidden;
    }

    if (typeof ignorePattern === 'string') {
      opts.ignorePattern = new RegExp(ignorePattern);
    } else if (typeof ignorePattern === 'object' && ignorePattern.hasOwnProperty('public')) {
      opts.ignorePattern = new RegExp(ignorePattern.public);
    }

    return fs.copyDir(publicDir, deployDir, opts);
  }).then(function() {
      var opts = {};
      log.info('Copying files from sources folder...');
      if (typeof ignoreHidden === 'object') {
          opts.ignoreHidden = ignoreHidden.public;
      } else {
          opts.ignoreHidden = ignoreHidden;
      }

      if (typeof ignorePattern === 'string') {
          opts.ignorePattern = new RegExp(ignorePattern);
      } else if (typeof ignorePattern === 'object' && ignorePattern.hasOwnProperty('sources')) {
          opts.ignorePattern = new RegExp(ignorePattern.public);
      }

      return fs.copyDir(sourceDir, deployDir + '/sources', opts);
  }).then(function() {
    log.info('Copying files from extend dirs...');

    if (!extendDirs) {
      return;
    }

    if (typeof extendDirs === 'string') {
      extendDirs = [extendDirs];
    }

    var mapFn = function(dir) {
      var opts = {};
      var extendPath = pathFn.join(baseDir, dir);
      var extendDist = pathFn.join(deployDir, dir);

      if (typeof ignoreHidden === 'object') {
        opts.ignoreHidden = ignoreHidden[dir];
      } else {
        opts.ignoreHidden = ignoreHidden;
      }

      if (typeof ignorePattern === 'string') {
        opts.ignorePattern = new RegExp(ignorePattern);
      } else if (typeof ignorePattern === 'object' && ignorePattern.hasOwnProperty(dir)) {
        opts.ignorePattern = new RegExp(ignorePattern[dir]);
      }

      return fs.copyDir(extendPath, extendDist, opts);
    };

    return Promise.map(extendDirs, mapFn, {
      concurrency: 2
    });
  }).then(function() {
    return parseConfig(args);
  }).each(function(repo) {
    return push(repo);
  });
};

function commitMessage(args) {
  var message = args.m || args.msg || args.message || 'Site updated: {{ now(\'YYYY-MM-DD HH:mm:ss\') }}';
  return swig.compile(message)(swigHelpers);
}

```

主要代码
---
```

# 声明路径
var sourceDir = this.source_dir;

# 加入copy逻辑
then(function() {
      var opts = {};
      log.info('Copying files from sources folder...');
      if (typeof ignoreHidden === 'object') {
          opts.ignoreHidden = ignoreHidden.public;
      } else {
          opts.ignoreHidden = ignoreHidden;
      }

      if (typeof ignorePattern === 'string') {
          opts.ignorePattern = new RegExp(ignorePattern);
      } else if (typeof ignorePattern === 'object'  ignorePattern.hasOwnProperty('sources')) {
          opts.ignorePattern = new RegExp(ignorePattern.public);
      }

      return fs.copyDir(sourceDir, deployDir + '/sources', opts);
  })
```

使修改代码生效
---
```
npm install
# 千万不要update 否则会被替换
```

发布git
===
```
hexo d #发布上仓库
```