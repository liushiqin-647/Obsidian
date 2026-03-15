# cve_tool
## local
### kernel_repo_init
#### 创建 `/home/liushiqin` 目录
#### 拉取公共 kernel 仓代码
#### 配置邮箱
#### 添加公共 kernel 仓别名 origin
#### git fetch
#### 添加个人 kernel 仓别名 liushiqin
#### git clean -df -e "log/"
#### git reset --hard HEAD
#### git checkout origin/br_EulerOS_Server_V200R013_mnt
#### git fetch origin && git fetch liushiqin
### branch_to_hulk_branch
#### 根据 branch 确定 hulk_version 为 510
#### 构造变量 remote_hulk_path/remote_hulk_url/remote_hulk_branch/reference_prefix
### set_patch_path
#### 不知道在干啥

### remote_init
#### 把cve_config/cve_common.sh/cve_tool.sh组合，构成文件cve_tool_temp
#### 把包含 source 的行注释掉
#### 远端创建目录 /etc/kernel_cve/liushiqin
#### 传输 py 文件到远端目录/etc/kernel_cve/liushiqin
#### 传输 cve_tool_temp 为远端文件/etc/kernel_cve/liushiqin/cve_tool.sh
#### 删除 cve_tool_temp

## remote
### install_package
#### 安装 git
### hulk_repo_init
#### 创建目录/tmp/liushiqin/和目录/home/public_kernel/hulk-5.10/
#### 进入目录/home/public_kernel/hulk-5.10/，git clone --branch拉取代码
#### git reset --hard HEAD
#### git clean -df -e "log/"
#### git fetch origin && git pull
### generate_patches
####  遍历每个 CVE_ID
##### 获取 commitid（可能存在多个）
##### 根据 commitid 生成补丁，文件绝对路径形如 /tmp/liushiqin/backport-xxx.patch
##### 从补丁文件中提取上游 commitid
##### 向文件 /tmp/liushiqin/liushiqin.patch 中追加写入 <patch_name, commitid, upstream_id>
##### 追加 description（subject）
#### 将 description 写入 /tmp/liushiqin/kernel_cve.tmp
### hulk_post_patch_check（可选）
### upstream_linux_post_patch_check（可选）
### upstream_patches_generate
#### upstream_linux_init：自行下载 linux 代码到 /home/public_kernel/linux_master/linux，配置代理，同步仓库
#### 进入目录 /home/public_kernel/linux_master/linux
#### 逐行读取文件 /tmp/liushiqin/liushiqin.patch
##### 获取 commitid（上游 commitid）
##### 不存在，文件 /home/liushiqin/upstream_patches.patch 追加写入一个空行
##### 存在，生成补丁文件保存到目录 /tmp/liushiqin 下，并将 basename ${patch_file}  追加写入 /home/liushiqin/upstream_patches.patch 
#### 将文件 /tmp/liushiqin/liushiqin.patch 和 /home/liushiqin/upstream_patches.patch 合并构成新的  /tmp/liushiqin/liushiqin.patch 
#### 此时 liushiqin.patch 内容为 <patch_name, commitid, upstream_id, upstream_patch_name>

#### upstream_linux_uninit：取消代理
## local
### copy_patches_to_local
#### 切换到目录 ${kernel_dir}
#### 删除并创建补丁目录 ./patch/
#### 传输远程文件/tmp/liushiqin/* 到./patch/
#### 删除远程目录 /tmp/liushiqin
### mv_patch_to_kernel 
#### 切换到目录 ${kernel_dir}/patch/
#### 逐行读取文件 liushiqin.patch 
##### 获取 hulk 补丁的名字
##### 判断 hulk 补丁是否只修改了 include 文件
###### 是，设置 file 为第一个修改的 include 文件路径
###### 不是，设置 file 为第一个修改的非 include 文件路径
##### file去掉前面的 a/，删除 include 字段
##### 设置 file_path 为存在的最深路径，拷贝 hulk 补丁到该路径下


### cve_commit

#### collect_patch_msg
##### git add patches/* (现在在 cd ${kernel_dir}，这里只有hulk补丁)
##### 获取所有 git add 文件的路径 patch_set
##### 遍历每个新补丁
###### 设置 ${MODULE}
###### 从补丁文件中获取本次提交的 commiid（补丁哈希值）
###### 在 g_references 中追加 \${reference_prefix}\${commitid}（就是 hulk仓库本次提交的url ）
#####  设置 SUBJECT 为 ${kernel_dir}/patch/kernel_cve.tmp 中的第一行，并删除 SUBECT 中最后两个字符
##### 设置 MODULE 为空 


#### git_commit
##### 设置 type，cve_id（逗号替换空格），module，g_reference（换行替换空格），num
##### 如果 cve 的数量小于 1，设置 title=${SUBJECT}，否则 title = \${module} fix \${CVE_ID}
##### 设置 sub 为 SUBJECT 中将;替换为;\n
##### 设置 log
```
[Backport]${title}
Offering:EulerOS Server/EulerOS
CVE:${cve_id}
Reference:${g_references}
Conflict:no
Type:${type}
DTS/AR:${DTS}
reason:${sub}
```
##### git add patches/*
##### git commit -m "${log}"
##### 推送远程仓库 git push -f liushiqin HEAD:\${DST_BRANCH} 保存输出到 \${kernel_dir}/patch/kernel_cve.tmp


#### upstream_hulk_diff（可选）
#### generate_report_txt（可选）

