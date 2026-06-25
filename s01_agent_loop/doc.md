# 1. 如何在当前项目创建 Python 虚拟环境？

在项目根目录执行：

```bash
python3 -m venv .venv
```

激活虚拟环境：

```bash
source .venv/bin/activate
```

确认当前 Python 来自项目内的 `.venv`：

```bash
which python
python --version
```

如果有 `requirements.txt`，安装依赖：

```bash
python -m pip install -r requirements.txt
```


# 2. `readline` 这段代码是干什么的？

```python
try:
    import readline
    readline.parse_and_bind('set bind-tty-special-chars off')
    readline.parse_and_bind('set input-meta on')
    readline.parse_and_bind('set output-meta on')
    readline.parse_and_bind('set convert-meta off')
except ImportError:
    pass
```

这段代码是为了优化终端输入体验，尤其是在 macOS 终端里输入中文时，避免退格、光标移动、字符显示出现异常。

`readline` 会影响 `input()` 的交互行为，比如上下键历史记录、左右移动光标、退格删除等。

放在 `try` 里是因为有些环境可能没有 `readline`。如果导入失败，就直接跳过，不影响程序继续运行。


# 3. `ANTHROPIC_BASE_URL` 和 `ANTHROPIC_AUTH_TOKEN` 这段是什么意思？

```python
if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)
```

意思是：如果当前环境里设置了 `ANTHROPIC_BASE_URL`，就从当前 Python 进程的环境变量里删除 `ANTHROPIC_AUTH_TOKEN`。

通常 `ANTHROPIC_BASE_URL` 表示使用自定义 API 地址，比如代理服务或兼容 Anthropic API 的第三方接口。

删除 `ANTHROPIC_AUTH_TOKEN` 是为了避免 SDK 自动带上 `Authorization: Bearer ...` 请求头，和自定义接口的认证方式冲突。

这里不会修改 `.env` 文件，只是修改当前 Python 进程里的环境变量。


# 4. `os.getenv` 的作用是？

`os.getenv("变量名")` 的作用是：从当前 Python 进程的环境变量里读取一个变量。

它本身不会主动读取 `.env` 文件。

之所以这里可以读取 `.env` 里的内容，是因为前面执行了：

```python
load_dotenv(override=True)
```

这句会先把 `.env` 文件里的变量加载到当前 Python 进程的环境变量中。之后 `os.getenv()` 才能读到这些值。

流程可以理解为：

```text
.env 文件
  -> load_dotenv()
  -> 当前 Python 进程的环境变量
  -> os.getenv()
```
