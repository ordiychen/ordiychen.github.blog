# <div align="center"><a href="https://github.com/theme-next/hexo-theme-next">NexT Theme</a> <a href="https://theme-next.org/">Website</a> <a href="https://github.com/theme-next/theme-next.org/tree/source/source">Source</a></div>

<p align="center">
  <a href="https://app.netlify.com/sites/theme-next/deploys"><img src="https://api.netlify.com/api/v1/badges/1d59e9ba-019f-4d9e-ac93-c73df98957c1/deploy-status" title="Building and Deploying status [Multi-Schemes]"></a>
  <a href="https://docs.theme-next.org"><img src="https://d322cqt584bo4o.cloudfront.net/theme-next-org/localized.svg" title="Add or improve translation in few seconds!"></a>
</p>

## Schemes

* :heart_decoration: [Muse](https://muse.theme-next.org)
* :six_pointed_star: [Mist](https://mist.theme-next.org)
* :pisces: [Pisces](https://pisces.theme-next.org)
* :gemini: [Gemini](https://theme-next.org) (**Default**)

## Autoinstall Hexo & NexT & NexT Website Source on Localhost

```bash
git clone https://github.com/theme-next/theme-next.org
cd theme-next.org
sh ./hexo-theme-next-autoinstall.sh
```


## use
```shell script
clean     Remove generated files and cache.
  config    Get or set configurations.
  deploy    Deploy your website.
  generate  Generate static files.
  help      Get help on a command.
  init      Create a new Hexo folder.
  list      List the information of the site
  migrate   Migrate your site from other system to Hexo.
  new       Create a new post.
  publish   Moves a draft post from _drafts to _posts folder.
  render    Render files with renderer plugins.
  server    Start the server.
  version   Display version information.

Global Options:
  --config  Specify config file instead of using _config.yml
  --cwd     Specify the CWD
  --debug   Display all verbose messages in the terminal
  --draft   Display draft posts
  --safe    Disable all plugins and scripts
  --silent  Hide output on console


hexo generate 
hexo server

```
部署到github
```shell script
#安装插件
npm install hexo-deployer-git --save
hexo clien
hexo generate && hexo deploy
```

## 创建文章
```shell script
hexo new 

# 创建草稿
hexo new draft '文章标题` 
#发布文章
hexo publish  `文件名称`
```

```shell script
npm install hexo-symbols-count-time --save

symbols_count_time:
  symbols: true
  time: true
  total_symbols: true 
  total_time: true 
```

```shell script
#监听文件状态 能够监视文件变动并立即重新生成静态文件。在生成时会比对文件的 SHA1 checksum
hexo generate --watch
```

```shell script
完成后部署
 hexo generate --deploy

#一件部署
hexo clean && hexo deploy
```