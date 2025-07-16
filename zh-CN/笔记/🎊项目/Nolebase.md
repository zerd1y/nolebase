# Nolebase 

## 官方文档

- 官方博客：[Nólëbase | 记录回忆，知识和畅想的地方](https://nolebase.ayaka.io/zh-cn/)
- 工具合集：[Nólëbase Integrations](https://nolebase-integrations.ayaka.io/pages/en/)
## 测试部署

>[!tip] 测试：（本地）
>```shell
>pnpm docs:dev 
>```
>有时候可能无法及时响应，重新运行上面的代码即可

>[!tip] 部署
>```shell
>git add .
>git commit
>git push
>```
>部署到 GitHub 网站上即可（ Vercel 会自动部署到线上）

## Tips

1. 笔记标题：
	- 创建文件时，文件头部是名字 
	- 下方一级标题要和文件名一样：网站渲染时，第一个一级标题就是博客标题
	- 后续文档从二级标题开始编辑
2. 图片显示：用精准的相对路径
3. 双向链接：用精准的相对路径
4. callout：只能使用 `tip`，`warning`，`danger`
5. obsidian 推荐框架：（支持双向链接、关系图谱）
	- nolebase ✔：相对文艺一点，但是关系图谱还不支持
	- Quartz 4 ✔：更简约、纯技术博客风
	- Digital Garden ❌：不好看
	- Obsidian Publish ⭕：obsidian 官方发布，界面和 Quartz 4 差不多，要付费 
6. [Reddit: Best way to self-host obsidian publish r/ObsidianMD](https://www.reddit.com/r/ObsidianMD/comments/16e5jek/best_way_to_selfhost_obsidian_publish/)