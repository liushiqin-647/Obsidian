```c
void path_put(const struct path *path)
{
    dput(path->dentry); // fs树中的一个节点
    mntput(path->mnt); // vfsmount
}
```
path 表示“在某个挂载（mnt）下的某个 dentry”

```c
REG("xcall", 0640, proc_pid_xcall_operations),
```
- **"xcall"**：在 `/proc/<pid>/`目录下创建名为 `xcall`的文件
- **0640**：文件权限（用户可读写，组只读，其他无权限）
- **proc_pid_xcall_operations**：文件操作函数集合