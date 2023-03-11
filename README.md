# README
branch ***source*** is the source code of this blog

## install hexo
https://hexo.io/zh-cn/docs/

## build local

```
mkdir code
cd code
git clone https://github.com/procr/procr.github.io.git
git checkout source

cd ..
hexo init blog
cd blog
npm install
npm install hexo-deployer-git --save

rm -rf _config.yml themes source scaffolds
ln -sf ../procr.github.io/_config.yml _config.yml
ln -sf ../procr.github.io/themes themes
ln -sf ../procr.github.io/source/ source
ln -sf ../procr.github.io/scaffolds/ scaffolds

```

## show local
```
$ hexo server
INFO  Validating config
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/ . Press Ctrl+C to stop.
```

## deploy
```
hexo deploy
```
in this step, you may use github token instead of login passwd
https://blog.51cto.com/u_15064643/4215363