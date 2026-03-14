# collect_patch_msg
## git add patches/* (现在在哪个目录下面啊！！！)
## 获取新创建的补丁文件的路径
## 遍历每个新补丁
### 设置 no_need_module
### 从补丁文件中获取 commiid
### 在 g_references 中追加 \${reference_prefix}\${commitid}
##  设置 SUBJECT 为 ${kernel_dir}/patch/kernel_cve.tmp 中的第一行，并删除 SUBECT 中最后两个字符
## 设置 MODULE 为空 


# git_commit
## 设置 type，cve_id（逗号替换空格），module，g_reference（换行替换空格），num
## 如果 cve 的数量小于 1，设置 title=${SUBJECT}，否则 title = ${module}fix ${CVE_ID}
## 设置 sub 为 SUBJECT 中将;替换为;\n
## 设置 log
## git add patches/*
## git commit -m "${log}"
## 推送远程仓库 git push -f liushiqin HEAD:${DST_BRANCH} 保存输出到 ${kernel_dir}/patch/kernel_cve.tmp


# upstream_hulk_diff（可选）
# generate_report_txt（可选）
