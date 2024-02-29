## 利用Github Action 自动在 github Pages 托管 hugo blog

1. 自行创建在本地blog根目录创建hugo workflow配置（`.github/workflows/hugo.yaml`）

2. 添加下面示例配置（注意需要自行修改配置部分）

    ```yaml
    name: pages-auto-build-deploy
    on:
      # workflow_dispatch: 
      push:
        branches:
          - main
    
    jobs:
      build-and-deploy:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v2
            with:
              submodules: true
              fetch-depth: 0
    
          - name: Setup Hugo
            uses: peaceiris/actions-hugo@v2
            with:
              hugo-version: '0.122.0' # 自行修改自己的版本
              extended: true
    
          - name: Build Hugo
            # 自行修改 baseURL
            run: hugo --gc --minify --baseURL "https://blog.codingforjoy.com/"
    
          - name: Deploy
            uses: peaceiris/actions-gh-pages@v3
            with:
              # 自行添加 action GH_PAGE_ACTION_TOKEN token，参考上面博客的截图方式
              github_token: ${{ secrets.GH_PAGE_ACTION_TOKEN }}
              publish_dir: ./public
              commit_message: ${{ github.event.head_commit.message }}

3. 在个人`GitHub`页面，依次点击`Settings`->`Developer settings`->`Personal access tokens`-> 进入创建token页面[Personal Access Tokens (Classic) (github.com)](https://github.com/settings/tokens)
   - 点击`Generate new token`
   - 添加一个名称
   - Expiration，选择没有期限
   - 勾选权限 `repo`、`workflow`
   - 完成token创建
   - 复制 token
4. 进入blog仓库 `Settings`->`Secrets and variables`->`Actions`
   - 点击`New repository secret`
   - Name设置为hugo.yml中[github_token的变量] `GH_PAGE_ACTION_TOKEN`(自己命令了的话，记得修改 hugo.yml)
   - Secret 粘贴刚刚创建的token
   - Add Secret，设置token完成
5. 本地推送，在 Action中查看进度，完成后，回到仓库首页，查看分支是否多了gh-pages
6. 然后，在仓库Settings->Github Pages，选择 gh-pages 分支 / 部署完成
7. 同时也可以修改下 domain，刷新等待，部署完成
8. 最后，访问查看效果

参考：[利用GitHub Action实现Hugo博客在GitHub Pages自动部署 - 飞狐的部落格 (lucumt.info)](https://lucumt.info/post/hugo/using-github-action-to-auto-build-deploy/)
