# picup

一个支持插件化的终端操作的图床程序. 可以大概视为 PicGo 的简陋终端版?

![](https://raw.githubusercontent.com/wzk0/photo/main/2024-02-24%2018%3A12%3A25.png)

## 特点

- 快速的, 简洁的, 轻量的;
- 支持不同的插件以添加不同的图床(**插件**=**图床**);
- 支持系统通知弹窗;
- 允许自定义剪切板内容, 亦允许将图片/文件重命名后上传;
- 支持快速设置配置;
- 支持直接上传剪切板图片;
- 支持直接上传剪切板内的文件路径;
- 支持与 `nautilus`, 或与其他 Gnome 插件集成, 实现右键文件即可上传, 或点击即可上传剪切板图片;
- ...

## 使用

> picup 只适用于 Linux 发行版, Gnome 桌面效果更佳.

先安装以下依赖:

```sh
sudo apt/dnf install xclip notify-send -y
```

下载仓库中的 `picup`.

将其移动到 `/usr/bin` 或将其所在路径export进环境.

也可以将其移动到 `nautilus` 的插件文件夹以实现`右键文件即可上传到图床`.

> /home/<用户名>/.local/share/nautilus/scripts

运行一次 `picup` 以生成默认的配置文件:

```sh
picup -h
```

设置 Github 图床:

```sh
picup set default github
picup set github user <你的 Github 用户名>
picup set github token <你的 Github Token>
picup set github repo <你的 Github 图床仓库>
```

若要启用 Gitee 图床

```sh
picup set new gitee
```

> 其他图床方法一样, 只需粘贴图床代码, 再执行一次 `set new <插件名>` 即可.

完成啦!

接下来只需:

```sh
picup hello.jpg
```

或右键 `nautilus` 中的文件上传, 即可将 `hello.jpg` 上传到默认的 Github 图床了.

此外, 可以使用:

- `-h` 查看帮助
- `-e` 直接编辑整个配置文件
- `-l` 列出信息
- `-c <图床名>` 指定图床
- `文件名 文件名` 指定上传的文件名
- `set 变量 新值` 快速设定变量
- ...

> 可 `set` 的变量有: `default` - 默认的图床, `editor` - 默认的编辑器, `form` - 剪切板格式, `@url` 为原链接, `new <插件名>` - 初始化指定插件的配置, `<插件名> 变量` - 设定指定插件的变量, 如 `repo` 等.

## 插件

> 此插件非彼插件, 是插在程序内部的, 而非拎出来的单文件. 在 `picup` 中, **插件=图床**.

若要开发插件, 可以参考下面的 Github 插件:

```python
# 以插件名定义类
class github:

    # 声明需要用户提供的参数, 此变量名不可变
    need=['token','user','repo']

    # 上传函数, 传入文件路径与可选的文件名, 函数名不可变
    def up(file,name=now):

        # 以下三行内容固定, 只需修改['github']内容与类名一致即可
        global data
        info=read_from(config+data,'data')['github']
        pic=read_from(file,'pic')

        # 请求函数, 插件开发的主要部分
        headers={"Authorization": "Bearer %s"%info['token'],"Accept": "application/vnd.github+json"}
        url='https://api.github.com/repos/%s/%s/contents/%s'%(info['user'],info['repo'],name+'.'+file.split('.')[-1])
        r=requests.put(url,headers=headers,json={"message":name,"content":pic})
        
        # 状态码判断
        if r.status_code==201:
            success(json.loads(r.text)['content']['download_url'])
        elif r.status_code==422:
            error('上传','该文件名已存在')
        elif r.status_code==401 or r.status_code==404:
            error('上传','未设定相关变量, 可使用 set 快速设定, 或使用 -e 进入编辑模式')
        else:
            error('上传','%s(错误状态码)'%r.status_code)
```

---

- 提供了 `read_from` 函数

传入`文件路径`和`类型`即可(若不传入, 则默认是 `pic`)

类型有 `pic` (图片)和 `data` (配置)两种, 当设定为 `pic` 时, 会获取文件的二进制内容; `data` 则会获取配置文件.

为保证插件仅读取自己的配置, 请在 `read_from(config+data,'data')` 后紧接`['插件名']`.

> `config+data` 无需且不能修改, 因为这是配置文件的路径.

---

- 提供了 `success` 函数

将获取到的图片直链传入即可完成`复制到剪切板`, `系统通知弹窗`, `终端文字输出`.

---

- 提供了 `error` 函数

传入`错误类型`与`错误具体原因`即可实现和 `success`一样的功能(没有`复制到剪切板`).

---

> 注意: 不可使用 `new`, `set`作类名.

综上所述, 若要开发一个插件, 只需设定好类名 > 确定 `need` 以读取此图床的配置(例如用户的token, 用户名什么的) > 在 `up` 函数内使用 `requests` 编写`上传图片+获取图片直链`部分的代码 > 再用 `success` 和 `error` 传出或提醒用户即可.

因此, 在写好一个插件后, 最好说明此插件需要用户进行哪些 `set`, 继而插件也可以读取这些配置. 例如:

> 不说明也无所谓, 因为加入了提醒.

```sh
picup set new github
picup set github user <你的 Github 用户名>
picup set github token <你的 Github Token>
picup set github repo <你的 Github 图床仓库>
```

那么在 Github 插件中, 只需 `user`, `token`, `repo` 三个配置信息.

此外, 如果你有更好的想法或建议, 欢迎在 `issues` 中提出噢~