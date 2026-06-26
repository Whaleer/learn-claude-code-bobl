# 1. `"mkfs"`, `"dd if="`, `"> /dev/sda"` 这几个是啥指令？

相关代码：

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if=", "> /dev/sda"]
```

这几个字符串不是普通业务参数，而是放在 `DENY_LIST` 里的危险命令片段。

`DENY_LIST` 的意思是：只要模型生成的 bash 命令里包含这些片段，就直接拦截，不进入后面的工具执行流程。


## `mkfs`

`mkfs` 是 `make filesystem` 的缩写，意思是创建文件系统，通常也就是格式化磁盘或分区。

比如：

```bash
mkfs.ext4 /dev/sda1
```

这条命令表示把 `/dev/sda1` 格式化成 ext4 文件系统。

危险点在于：如果目标分区上原来有数据，格式化会破坏原来的文件系统，导致数据丢失。


## `dd if=`

`dd` 是一个底层数据复制工具，可以按字节或块直接读写文件、镜像、磁盘设备。

其中 `if=` 表示 `input file`，也就是输入来源。

比如：

```bash
dd if=image.iso of=/dev/sda
```

这条命令表示从 `image.iso` 读取内容，然后写入 `/dev/sda`。

如果 `of=` 后面是一个真实磁盘设备，这个操作可能会直接覆盖整块磁盘。

所以代码里用 `"dd if="` 作为危险片段，是为了拦截这类底层磁盘读写命令。


## `> /dev/sda`

`>` 是 shell 的输出重定向符号，不是单独的命令。

比如：

```bash
echo hello > /dev/sda
```

意思是把 `echo hello` 的输出写入 `/dev/sda`。

`/dev/sda` 在 Linux 里通常表示一整块磁盘设备。往它里面直接写内容，可能破坏磁盘上的分区表或文件数据。

所以 `"> /dev/sda"` 也是一个非常危险的命令片段。


# 2. `PERMISSION_RULES` 是干什么的？

相关代码：

```python
# Gate 2: Rule matching - context-dependent checks
PERMISSION_RULES = [
    {"tools": ["write_file", "edit_file"],
     "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
     "message": "Writing outside workspace"},
    {"tools": ["bash"],
     "check": lambda args: any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"]),
     "message": "Potentially destructive command"},
]
```

这段是权限系统里的第二道门：

```text
Gate 2: Rule matching
```

它不像 `DENY_LIST` 那样“只要出现就绝对禁止”，而是根据不同工具、不同参数，判断这次工具调用是不是有风险。

`PERMISSION_RULES` 是一个规则列表。每一条规则都有三个字段：

```python
{
    "tools": [...],
    "check": lambda args: ...,
    "message": "..."
}
```

它们分别表示：

```text
tools   -> 这条规则适用于哪些工具
check   -> 怎么判断这次调用是否有风险
message -> 如果命中规则，提示什么原因
```


## 第一条规则：写文件不能逃出工作区

```python
{"tools": ["write_file", "edit_file"],
 "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
 "message": "Writing outside workspace"}
```

这条规则只适用于两个工具：

```python
["write_file", "edit_file"]
```

也就是写文件和编辑文件。

它检查的是：模型要写入或编辑的路径，最终是否还在 `WORKDIR` 工作目录里面。

比如：

```text
path = "notes/a.txt"
```

通常会被解析成：

```text
WORKDIR/notes/a.txt
```

这还在工作区里，是安全的。

但如果模型传入：

```text
path = "../../secret.txt"
```

这个路径经过 `.resolve()` 之后，可能会逃出当前项目目录。

所以这里用：

```python
(WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR)
```

判断最终真实路径是否仍然属于 `WORKDIR`。

前面还有一个 `not`：

```python
not (...is_relative_to(WORKDIR))
```

所以完整意思是：

```text
如果路径不在 WORKDIR 里面
  -> 命中风险规则
  -> 返回 "Writing outside workspace"
```


## 第二条规则：bash 里不能出现潜在破坏性命令

```python
{"tools": ["bash"],
 "check": lambda args: any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"]),
 "message": "Potentially destructive command"}
```

这条规则只适用于 `bash` 工具。

它检查 bash 命令字符串里有没有这些危险片段：

```python
["rm ", "> /etc/", "chmod 777"]
```

比如：

```bash
rm test.txt
```

包含：

```python
"rm "
```

会命中规则。

再比如：

```bash
echo abc > /etc/hosts
```

包含：

```python
"> /etc/"
```

也会命中规则。

这句代码：

```python
any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"])
```

可以拆成：

```text
拿到 bash 命令 command
  -> 遍历危险关键词列表
  -> 只要有任意一个关键词出现在 command 里
  -> 返回 True
```

所以这条规则的意思是：

```text
如果 bash 命令看起来可能会删除文件、写系统目录、放宽权限
  -> 命中风险规则
  -> 返回 "Potentially destructive command"
```


## `check_rules()` 怎么使用这些规则？

下面这个函数负责真正执行规则匹配：

```python
def check_rules(tool_name: str, args: dict) -> str | None:
    for rule in PERMISSION_RULES:
        if tool_name in rule["tools"] and rule["check"](args):
            return rule["message"]
    return None
```

执行流程是：

```text
拿到工具名和参数
  -> 遍历 PERMISSION_RULES
  -> 看这条规则是否适用于当前工具
  -> 如果适用，再执行 check(args)
  -> 如果 check 返回 True，说明命中风险
  -> 返回对应 message
  -> 如果全部没命中，返回 None
```

举个例子：

```python
tool_name = "bash"
args = {"command": "rm test.txt"}
```

因为 `tool_name` 是 `"bash"`，所以会匹配第二条规则。

然后检查：

```python
any(kw in "rm test.txt" for kw in ["rm ", "> /etc/", "chmod 777"])
```

因为 `"rm "` 出现在 `"rm test.txt"` 里，所以返回 `True`。

于是 `check_rules()` 返回：

```python
"Potentially destructive command"
```

一句话总结：

```text
PERMISSION_RULES 是一张风险规则表。
它根据工具名和工具参数判断这次调用是否敏感。
命中后返回一个风险原因，交给后面的权限流程处理。
```
