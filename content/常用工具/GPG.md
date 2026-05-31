 GPG (GNU Privacy Guard) 是 OpenGPG 标准的开源实现，用于加密，签名，和密钥管理。它的核心功能主要分为3个部分。
 
> - <font color="#45ce6e">加密/解密：基于非对称加密（公钥加密、私钥解密），保护数据机密性</font>。对文件进行加密，收到文件的一端，只有具备私钥才能对文件进行解密阅读。比如加密邮件
>   <font color="orange">目的是让指定的人读到内容</font>。
> - <font color="#45ce6e">数字签名：用私钥签名、公钥验证，保证完整性和来源真实性</font>。将文件用私钥签名，相当于给文件盖章。所有持有我公钥的人，都可以对这份文件进行验证。比如 Linux 软件源的验证。
>   <font color="orange">目的是证明这份数据确实出自本人，且没有被篡改。</font>注意⚠️，这个和访问权限没有关系。
> - <font color="#45ce6e">密钥管理：生成、导入、导出、吊销密钥对。</font> 命令行入口是 `gpg`, 几乎所有 Linux 主流发行版都会预装。 

所以，它的核心用途就两种，<font color="orange">文件加密</font> 和 <font color="orange">文件防伪防篡改</font>。注意⚠️，GPG 针对的是文件而不是针对用户。GPG 针对数据本身，公钥加密配私钥解密管"谁能读"。私钥签名配公钥验证管"是不是真的、有没有被改"。

## <font color="grape">GPG与SSH publickey</font>
GPG 与 SSH publickey 底层密码学原语是同一套：非对称密钥对 + 签名/验证。区别在于**它们解决的问题不同，因此协议、密钥格式、信任模型都不一样**

<font color="#45ce6e">1. 解决的问题不同</font>
- SSH 公钥认证：解决**实时身份认证**。证明"连接过来的这个人持有对应私钥"，是一个在线的挑战-应答（challenge-response）过程。
- GPG：解决**离线的数据保护**。对一份数据本身做加密或签名，签名可以在几年后、脱离任何连接的情况下被任何人验证。

<font color="#45ce6e">2. 认证流程不同</font>
SSH 是交互式的：服务器发一个随机 nonce，客户端用私钥签名，服务器用 `authorized_keys` 里的公钥验证。签名只对这次会话的临时数据有效，验证完即抛弃。 
GPG 的签名是**附着在数据上的持久产物**——签名和被签数据一起存在，谁拿到都能验证，与时间、会话无关。

<font color="#45ce6e">3. 信任模型不同（这是最大的区别）</font>
- SSH：信任是**手动直配**的。你把公钥塞进服务器的 `authorized_keys`，就是点对点的白名单，没有任何第三方背书。
- GPG：有一套**信任传递机制**——Web of Trust（信任网络）。A 给 B 的公钥签名，表示"我确认这把钥匙属于 B"，C 信任 A 就可以间接信任 B。解决的是"陌生人之间如何验证身份"的问题。
  ▪︎<font color="gray">它解决的问题是，我怎么知道 “这是B的公钥” 这件事是真的。所谓A 给 B 的公钥签名，就是A用自己的私钥给 "这把公钥属于B" 这个声明进行签名。这个签名等于 A 公开担保："我 A 当面核实过，这把公钥的主人确实是 B，我拿我的信誉作保。"</font>
  ▪︎ <font color="gray">于是信任可以传递：</font>
	- <font color="gray">你认识 A，当面交换过公钥，确信 A 的公钥是真的 </font>
	- <font color="gray">你不认识 B，但你看到 B 的公钥上有 A 的签名</font>
	- <font color="gray">因为你信任 A 的判断，你就可以间接相信 B 的公钥也是真的</font>
<font color="gray">C 信任 A → A 担保 B → C 可以推及 B。一层层传下去，就织成一张"信任的网"（web）。</font>

<font color="orange">总结：把私钥想成同一种"印章技术"。SSH 是“门禁刷脸”——你站在门口实时验证，过了就进，不留凭证；GPG 是“公文盖章”——章盖在文件上，文件流转到哪里别人都能验真伪，还有一套"谁的章可信"的背书体系。原语相同，但一个是在线认证协议，一个是离线数据签名/加密体系。</font>

## <font color="grape">GPG Key的使用</font>

一般我们要下载一个可信任的包，就要经过GPG来验证。以 Ubuntu 的 apt 为例。
```bash
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add - 
```
这行命令就是去 NVDIA 的地址下载 GPG 公钥，然后导入 apt 的 keyrings 里。apt 可信 keyrings 在 `/usr/share/keryrings` (新的目录)，`/etc/apt/trusted.gpg.d/` 和 `/etc/apt/trusted.gpg` (旧的全局信任目录)。

第二个命令就是把下载下来的GPG key 加进去，`-` 代表**标准输入（stdin）**。很多命令约定用 `-` 表示"不从文件读，而是从标准输入读"。

这里 `apt-key add` 的正常用法是 `apt-key add <文件名>`,从一个文件读公钥。但这里没有文件,公钥是 curl 从网上抓下来、通过**管道 `|`** 传过来的。管道把左边命令的 stdout 接到右边命令的 stdin,所以要告诉 `apt-key`:"别找文件了,从 stdin 读"——这就是 `-` 的作用。

数据流串起来:

```
curl 下载公钥 → stdout → | 管道 → stdin → apt-key add - 读取并导入
```

<font color="red">一个要提醒:`apt-key` 已经废弃了</font>

现在推荐的做法是把公钥单独放到一个文件,然后**只授权给特定的源**:

```bash
# 下载并转成 keyring 格式,放到专门目录
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# 在源定义里用 signed-by 显式指定:这个源只认这一把钥匙
# deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://...
```

这样这把 NVIDIA 的钥匙就只为 NVIDIA 这个源背书,而不是给全系统的包来源开后门。
**`gpg`** —— 就是前面说的那个 GPG 工具本体。
**`--dearmor`** —— 做格式转换。要理解它,先得知道 GPG 公钥有两种存储形态:
- **ASCII-armored(装甲格式)**:纯文本形态,内容是可打印字符,通常以 `-----BEGIN PGP PUBLIC KEY BLOCK-----` 开头。好处是能直接贴进邮件、网页、聊天框里不会损坏。你 `curl` 从 NVIDIA 那个地址下下来的 `gpgkey` 就是这种文本格式。
- **binary(二进制格式)**:原始的二进制字节,体积更小,是 GPG 内部实际处理的形态。
`--dearmor` 的作用就是**把 ASCII-armored 文本转回二进制**。"armor"是"加装甲/包装成文本"的意思,`de-armor` 就是"脱掉文本外壳,还原成二进制"。反向操作是 `--armor`(也叫 `--enarmor`),把二进制裹成文本。

为什么要转? 因为 `signed-by` 方案的 keyring 文件(`.gpg` 结尾那个)需要的是二进制格式,而网上发布的公钥常是文本格式,所以中间要 `--dearmor` 转一道。

**`-o`** —— `--output` 的简写,指定输出写到哪个文件。`-o xxx.gpg` 就是"转换结果写进 xxx.gpg 这个文件",而不是默认打印到屏幕(stdout)。

数据流:
```
curl 下载文本格式公钥 → stdout → 管道 → gpg 的 stdin
                                          ↓
                            --dearmor 转成二进制
                                          ↓
                          -o 写入 keyring 文件(.gpg)
```

注意这里 `gpg` 没写输入文件名,所以它默认从 **stdin** 读。

**一句话总结**:这条命令把 curl 抓来的文本格式公钥,转成二进制 keyring 格式,存成一个专用文件,供后面 `signed-by` 指定使用。

#### <font color="#b48ff4">signed-by的使用</font>

`signed-by` 用在 **apt 源的定义里**,作用是把"这个源"和"验证它的那把公钥"绑定起来——告诉 apt:这个源的包,只认这一把钥匙的签名,别的钥匙一概不算数。

这正是它取代 `apt-key` 的核心价值:把信任**限定到单个源**,而不是丢进全局 keyring 给所有源背书。

**两种写法**

apt 源定义有新旧两种格式,`signed-by` 在两种里都能用。

**写法一:传统的 `.list` 格式(单行)**

文件放在 `/etc/apt/sources.list.d/` 下,比如 `nvidia-container-toolkit.list`:

```
deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://nvidia.github.io/libnvidia-container/stable/deb/$(ARCH) /
```

拆解这一行:

- `deb` —— 这是个二进制包源(对应 `deb-src` 是源码源)
- `[signed-by=...]` —— **关键部分**,方括号里指定验证用的 keyring 文件路径,就是你前面 `gpg --dearmor` 生成的那个 `.gpg` 文件
- 后面是源的 URL 和路径

**写法二:新的 `.sources` 格式(DEB822,多行)**

较新系统更推荐这种,可读性更好。文件名如 `/etc/apt/sources.list.d/nvidia-container-toolkit.sources`:

```
Types: deb
URIs: https://nvidia.github.io/libnvidia-container/stable/deb/$(ARCH)
Suites: /
Signed-By: /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

这里 `Signed-By:` 单独成行,指向同一个 keyring 文件。

**完整流程串起来**

从零配置一个带签名验证的源,通常是两步:

```bash
# 第一步:下载公钥,转二进制,放进专用 keyring 文件
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# 第二步:写源定义,用 signed-by 指向刚才那把钥匙
echo "deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://nvidia.github.io/libnvidia-container/stable/deb/\$(ARCH) /" \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

然后 `sudo apt update`,apt 拉取这个源的包索引时,就会用你指定的那把钥匙去验证签名,验不过就报错拒绝。

**几个实务要点**

`signed-by` 后面的路径必须是**绝对路径**,且指向的是 `--dearmor` 后的二进制 keyring,不是文本格式的原始公钥——格式不对 apt 会验证失败。

如果一个源需要认多把钥匙,`signed-by` 可以跟多个文件,用逗号隔开。

配好后想验证是否生效,跑 `sudo apt update`,看对应源那行有没有报 `NO_PUBKEY` 或签名错误;没报错就说明钥匙和源对上了。


> [!NOTE] 注意
> <font color="orange">关于GPG的使用，这一节内容，具有实效性。这里阐述的是 GPG Key 使用的逻辑和原理。具体的操作步骤要根据当下最新的系统机制而定。</font>


