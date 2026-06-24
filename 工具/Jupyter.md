在项目根目录（有 `.venv`的那个目录）执行：

## 第一步：往虚拟环境装 ipykernel

```bash
uv pip install ipykernel
```

## 第二步：注册 Kernel 给全局 Jupyter

**Linux / macOS：**

```bash
python -m ipykernel install --user --name numpy-100 --display-name "Python (numpy-100)"
```