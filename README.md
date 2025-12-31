# 这是什么
这里是一个利用github写文章，然后利用`mkdocs`生成gh-page静态页面
的基础目录模板文件。你可以随便 clone 下来改改就能用

# 目录文件结构
```base
find | grep -v ./.git/
.
./overrides
./overrides/partials
./overrides/partials/comments.html
./.github
./.github/workflows
./.github/workflows/ci.yml
./README.md
./docs
./docs/about.md
./docs/index.md
./docs/blog
./docs/blog/index.md
./docs/blog/posts
./docs/blog/posts/2025-12-31-example.md
./docs/blog/assets
./docs/tags.md
./docs/assets
./docs/assets/img
./mkdocs.yml

```
- docs 所有的文章目录包括资源文件比如图片等在 docs/assets 目录
- docs/blog/posts/*.md 博客文章
- docs/blog/index.md 博客首页
- docs/index.md 首页
- docs/about.md 关于


# 关于评论

https://giscus.app/zh-CN

# 怎么使用
直接 fork 本分支 然后修改你想改的地方，比如 `mkdocs.yml`,
`./overrides/partials/comments.html` 

- 发表新文章 参考 `./docs/blog/posts/2025-12-31-example.md`

