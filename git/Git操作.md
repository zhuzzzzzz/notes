#### 克隆仓库

##### 使用 SSH 克隆仓库

```shell
# 跳过密钥对设置

# 使用ssh默认端口时
git clone git@github.com:username/repo-name.git
# 验证端口连通性
ssh -T git@host


# 使用其他端口时
git clone ssh://git@git.example.com:222/mygroup/myproject.git
# 验证端口连通性
ssh -p 222 -T git@git.example.com
```

#### 上游仓库管理

```shell
#添加加上游仓库, 命名为 upstream
git remote add upstream https://github.com/original/repo.git

# 查看当前远程仓库
git remote -v

# 删除上游仓库
git remote remove upstream
# 或
git remote rm upstream

# 重命名
git remote rename <旧名称> <新名称>

# 重新修改地址
git remote set-url origin https://github.com/zhuzzzzzz/IocDock.git
```

