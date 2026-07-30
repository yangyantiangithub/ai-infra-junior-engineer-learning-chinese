# Linux / Shell 速查表 — AI 基础设施版

在基于 Linux 的基础设施工作中每天都会用到的命令和模式。

## 文件和目录

```bash
ls -lah                          # 详细列表、全部、人类可读大小
ls -lt                           # 按时间最新排序
ls -lS                           # 按大小最大排序

cd -                             # 返回上一个目录
pushd /tmp && popd               # 书签 + 返回

cp -r src/ dest/                 # 递归复制
cp -a src/ dest/                 # 存档模式：保留权限、时间戳、符号链接
mv old new
rm -rf bad/                      # 小心使用

mkdir -p a/b/c                   # 级联创建父目录
ln -sf /actual /link             # 符号链接（强制替换）

stat file                        # 详细元数据
file file                        # 查看文件类型
du -sh dir/                      # 总大小，人类可读
df -h                            # 磁盘剩余空间，人类可读
```

## 查找文件

```bash
find . -name '*.log' -mtime -1                  # 24 小时内修改过的 *.log
find . -type f -size +100M                       # 大于 100 MB 的文件
find . -type f -exec grep -l "pattern" {} +      # 在所有文件中搜索
find . -type d -empty -delete                    # 删除空目录

# 如果安装了更好的替代工具
fd 'pattern'                                     # 更快的 find
rg 'pattern'                                     # 更快的 grep
```

## grep 相关

```bash
grep -rn 'pattern' src/                          # 递归，显示行号
grep -rni 'pattern' --include='*.py' src/        # 不区分大小写，通配符
grep -rl 'pattern' src/                          # 仅显示文件名
grep -v 'noise'                                  # 反向匹配
grep -A 3 -B 1 'pattern'                         # 后 3 行，前 1 行
grep -E 'foo|bar'                                # 正则表达式
```

## 管道和组合

```bash
# 目录树中最大的 5 个文件
du -h ./* 2>/dev/null | sort -h | tail -5

# 最近 10 分钟日志中包含 ERROR 的行
journalctl --since "10 min ago" | grep -i error

# 统计 nginx access log 中各状态码的请求数
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 访问某个端点的 Top IP
grep '/predict' access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head
```

## 权限

```bash
chmod 755 script.sh              # rwxr-xr-x
chmod +x script.sh               # 添加执行权限
chmod -R u+rw,go-w dir/          # 递归符号模式
chown -R user:group dir/

# 查看权限
stat -c '%a %U %G %n' file
ls -l                            # 人类可读

# 诊断权限问题
namei -l /path/to/thing          # 诊断整个路径的 ACL
```

## 进程

```bash
ps aux                           # 所有进程
ps -ef --forest                  # 树形结构
pgrep -a -f 'pattern'            # 按命令名搜索
pkill -f 'pattern'               # 按命令名杀死
kill -9 <pid>                    # SIGKILL（最后手段）
kill -15 <pid>                   # SIGTERM（让进程清理）

top                              # 实时查看
htop                             # 更好的 top（如果安装）
iotop                            # IO 使用情况
iftop                            # 网络使用情况

# 查看端口监听
ss -tlnp                         # TCP 监听 + 程序名
ss -tunp                         # 所有 TCP/UDP + 连接
lsof -i :8080                    # 8080 端口上的程序
lsof -p <pid>                    # 该 pid 打开的文件/套接字
```

## systemd

```bash
systemctl status nginx
systemctl start|stop|restart nginx
systemctl enable nginx           # 开机自启
systemctl daemon-reload          # 编辑 .service 文件后执行

journalctl -u nginx -f           # 跟踪日志
journalctl --since "1 hour ago" --until "10 min ago"
journalctl -u nginx --since today --no-pager | tail -200
```

## 网络

```bash
# 连通性
ping -c 4 host
traceroute host
mtr host

# DNS
dig +short example.com
dig example.com @1.1.1.1         # 查询特定 DNS 服务器
host -t MX example.com

# HTTP
curl -sS https://example.com
curl -v -X POST -H 'content-type: application/json' \
     -d '{"x":1}' https://example.com/api
curl -o file.tar.gz -L https://example.com/release.tar.gz
curl -I https://example.com      # 仅查看响应头
curl -w '\n%{http_code} %{time_total}s\n' https://example.com

# 接口信息
ip a
ip route
ip -s link show eth0             # 统计信息
```

## SSH

```bash
ssh user@host
ssh -i ~/.ssh/key.pem user@host
ssh -L 8080:remote-svc:80 user@host        # 本地端口转发
ssh -R 8080:localhost:80 user@host         # 远程端口转发
ssh-keygen -t ed25519                       # 生成密钥
ssh-copy-id user@host                       # 安装公钥
~/.ssh/config:
    Host bastion
        HostName 10.0.0.1
        User you
        IdentityFile ~/.ssh/id_ed25519
```

## 文本处理

```bash
sed -i 's/old/new/g' file        # 原地替换
sed -n '20,40p' file             # 打印 20-40 行

awk '{print $1, $3}' file
awk -F',' '{print $2}' csv       # 逗号分隔
awk '$3 > 100 {print}' file       # 按第 3 列过滤

cut -d',' -f1,3 csv               # 第 1 和 3 列
sort -k2 -n file                  # 按第 2 列数字排序
sort -u                           # 去重
uniq -c                           # 统计重复数量（先 sort）

tr '[:upper:]' '[:lower:]'        # 转小写
tr -d '\r' < file                 # 去除回车符

wc -l file                        # 行数统计
head -n 20 file                   # 前 20 行
tail -f file                      # 跟踪文件增长
tail -n 100 file
```

## tar / gzip

```bash
tar czf archive.tar.gz dir/      # 创建 gzip 压缩包
tar xzf archive.tar.gz           # 解压
tar tzf archive.tar.gz | head    # 列出内容
tar xzf archive.tar.gz -C /dst   # 解压到指定路径

gzip -9 file                     # 压缩（最大压缩比）
zcat file.gz | grep pattern      # 搜索压缩文件内容
```

## Bash 脚本基础

```bash
#!/usr/bin/env bash
set -euo pipefail        # 出错退出、未定义变量、管道错误
IFS=$'\n\t'              # 更安全的字段分割

# 变量
name="value"
readonly CONFIG="/etc/app.conf"
PATH="/usr/local/bin:$PATH"

# 参数
if [[ $# -lt 1 ]]; then
    echo "用法: $0 <arg>" >&2
    exit 1
fi
input="$1"
shift

# 条件判断
if [[ -f /etc/passwd ]]; then ...; fi
if [[ "$x" == "go" ]]; then ...; fi

# 循环
for f in *.log; do
    echo "$f"
done

while IFS= read -r line; do
    echo "$line"
done < file.txt

# 函数
greet() {
    local name="$1"
    echo "你好 $name"
}
greet world

# 清理陷阱
tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT
```

## 作业控制

```bash
sleep 100 &              # 后台运行
jobs                     # 列出任务
fg %1                    # 切换到前台
bg %1                    # 在后台继续运行
Ctrl-Z                   # 挂起
nohup long_cmd &         # 退出登录后继续运行
disown                   # 从 shell 分离
```

## 磁盘和文件系统

```bash
df -h                    # 文件系统使用情况
du -sh dir/              # 单个目录总大小
du -h --max-depth=1 .    # 深度为 1
ncdu                     # 交互式磁盘使用分析（如果安装）

lsblk                    # 块设备
mount | column -t        # 已挂载的文件系统
```

## 有用的快捷键

| 按键 | 作用 |
|---|---|
| `Ctrl-R` | 反向增量历史搜索 |
| `Ctrl-A` / `Ctrl-E` | 行首 / 行尾 |
| `Ctrl-W` | 删除前一个单词 |
| `Ctrl-U` | 删除到行首 |
| `Ctrl-L` | 清屏 |
| `Alt-.` | 上一条命令的最后一个参数 |
| `!!` | 上一条命令 |
| `!$` | 上一条命令的最后一个参数 |
| `^old^new` | 替换后重新执行上一条命令 |

## 常见陷阱

- **`rm -rf $VAR/`** — 如果 `VAR` 为空，会变成 `rm -rf /`。始终使用引号并检查。
- **文件名含空格** — 始终引用变量：`"$f"`，而不是 `$f`。
- **`cat foo | grep bar`** — 无用的 cat。直接 `grep bar foo`。
- **`for f in $(ls)`** — 遇到空格会出错。使用 `for f in *` 或 `find ... -print0 | xargs -0`。
- **脚本中没有 `set -e`** — 错误会被静默忽略。始终使用 `set -euo pipefail`。
- **Bash 版本** — macOS 自带上古版本 bash 3.2。使用 `#!/usr/bin/env bash` 并在 Linux 上测试。

## 另请参阅

- `man bash`, `man bash-builtins`
- [ShellCheck](https://www.shellcheck.net/) — 脚本代码检查
- [explainshell.com](https://explainshell.com/) — 命令拆分解读
