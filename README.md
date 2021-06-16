
发布一个帖子需要这几个步骤：

- `hugo new post/my-first-post.md` 来新建帖子
- 编辑完帖子后，通过`hugo server -D`来预览效果
- 效果确认无误后，执行`hugo -d docs`来生成新的blog静态文件到`docs`文件夹中
- 最后，git add ci push 到 github
