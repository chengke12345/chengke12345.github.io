YML（或 .yaml）文件是 **YAML（YAML Ain't Markup Language）** 格式的文本文件，主要用于<font color="orange">配置</font>和<font color="orange">数据序列化</font>。
## <font color="grape">核心特点</font>

<font color="orange">用缩进表示层级关系</font>, 不用大括号或分号。		
<font color="#eb4349">YAML 是JSON的超集，任何合法的JSON都是合法的YAML</font>
YAML 对**缩进极其敏感**，而且**禁止用 Tab**，必须用空格（一般 2 个）。这是新手最常踩的坑——文件看起来没问题但解析报错，往往就是 Tab 和空格混用，或者某一层缩进对不齐。

## <font color="grape">YAML的使用者</font>

JSON主要是给程序之间交换数据使用的格式，主要是程序与程序或API之间。

YAML是既要给机器读取，又要方便人类编辑，定位是<font color="orange">“机器+人类”</font>.
	-<font color="#45ce6e"> 人类：写、改、审查（code review）。YAML 的设计目标就是让运维/开发能直接手写和读懂配置，不需要 GUI 工具。</font>
	- <font color="#45ce6e">程序：在启动或运行时加载、解析为内存中的数据结构。</font>

## <font color="grape">YAML与JSON</font>

YAML 和 JSON 的定位差别很明显：JSON 主要用于**程序之间通信**（API 请求/响应），人偶尔看看；YAML 主要用于**人写给程序看**的配置。所以 YAML 牺牲了一些解析效率（缩进敏感、语法更复杂）来换取人类可读性。

|      | JSON      | YAML    |
| ---- | --------- | ------- |
| 可读性  | 一般        | 好       |
| 注释   | 不支持       | 支持(`#`) |
| 引号   | 字符串必须加    | 通常可以    |
| 主要用途 | 数据交换(API) | 配置文件    |

## <font color="grape">YAML的解析</font> 

一般情况下，YAML完全不需要自己写解析程序，而且强烈不建议自己写。几乎每种主流编程语言，都有成熟的 YAML 解析库，把 YAML 直接映射成原生数据结构。
```python
import yaml

with open('config.yml') as f:
    config = yaml.safe_load(f)

# config 直接就是 dict
print(config['model']['name'])           # "Qwen3-32B-AWQ"
print(config['gpus'][0])                  # "2080Ti-22G-0"
```

加载完成后，config 直接就是一个 dict。
YAML 规范比看起来复杂得多，自己写解析器会掉进一堆坑，尽量不要自己写，完整的 YAML 1.2 规范有上百页。自己写解析器除非是做研究或者写 YAML 库本身，否则纯粹是浪费时间。

## <font color="grape">YAML数据格式</font>

YAML 和 JSON 表达的是更通用的**结构化数据**，键值对（map/dict）只是其中一种结构。完整地说，它们都支持：
- 标量（字符串、数字、布尔、null）
- 序列（list/array）—— 没有键，只有顺序
- 映射（map/dict）—— 键值对
- 以及上述的任意嵌套

**YAML 和 JSON 是同一类东西的两种风格**，差别在于优化目标（人 vs 机器）JSON解析快、语义无歧义、适合程序间交换。而YAML更适合人类阅读。

例如：
```python
import yaml

yaml_text = """
- 2080Ti
- 3090
- 4090
"""

data = yaml.safe_load(yaml_text)

print(data)         # ['2080Ti', '3090', '4090']
print(type(data))   # <class 'list'>

# 获取三个字符串
print(data[0])      # '2080Ti'
print(data[1])      # '3090'
print(data[2])      # '4090'

# 遍历
for gpu in data:
    print(gpu)
```

嵌套场景下的访问
```yaml
cluster:
  name: HeteroServe
  gpus:
    - 2080Ti
    - 3090
    - 4090
```
python中提取
```python
data = yaml.safe_load(text)
# data 是: {'cluster': {'name': 'HeteroServe', 'gpus': ['2080Ti', '3090', '4090']}}

print(data['cluster']['gpus'][0])  # '2080Ti'
print(data['cluster']['gpus'])     # ['2080Ti', '3090', '4090']
```

解析器做的事就是把 YAML 文本**反序列化**成宿主语言的原生数据结构——Python 里是 `dict` 和 `list`。后续就用语言本身的方式访问，不需要关心 YAML 语法了。YAML 和 JSON 本质上都是**序列化/反序列化**的过程，只不过一个面向人类配置，一个面向机器交换。

## 关于空行

```python
import yaml

text = """
- 2080Ti
- 3090
- 4090



- a
- b
- c
"""

data = yaml.safe_load(text)
print(data)
# ['2080Ti', '3090', '4090', 'a', 'b', 'c']
print(len(data))  # 6
```
**YAML 会把它们解析成同一个列表**，空行不影响结构. **空行在 YAML 里只是视觉分隔，没有语法意义**。解析器只看缩进层级和 `-` 标记，只要这些 `-` 都在同一缩进层级，就属于同一个序列。

## <font color="grape">YAML列表(list)与字典(dict)</font>

区分规则（关键）: YAML 里靠 `:` 和 `-` 这两个符号区分容器类型：

| 符号  | 含义   | 对应python | 对应C++       |
| --- | ---- | -------- | ----------- |
| `:` | 映射   | dict     | std::map    |
| `-` | 序列元素 | list     | std::vector |

看到 `:` 就是 map，看到 `-` 就是 list。**两者可以任意嵌套**：map 的值可以是 list，list 的元素可以是 map。

## 字典中存放列表

```yaml
group1:
  - 2080Ti
  - 3090
  - 4090

group2:
  - a
  - b
  - c
```
python解析后得到的是
```python
{
    'group1': ['2080Ti', '3090', '4090'],
    'group2': ['a', 'b', 'c']
}
```

## 列表中存放字典

这是一个字典中的两个键分别对应两个列表。如果要得到最外层是一个列表
```yaml
- group1:
    - 2080Ti
    - 3090
    - 4090
- group2:
    - a
    - b
    - c
```

python解析出来

```python
[                                            # ← 这才是 list
    {'group1': ['2080Ti', '3090', '4090']},
    {'group2': ['a', 'b', 'c']}
]
```
注意对比，`key :` 被解析为字典的键，而 `-` 被解析为列表元素。

## 列表中存放列表

如果要得到最外层列表，还可以采用
```yaml
- - 2080Ti
  - 3090
  - 4090
- - a
  - b
  - c
```
或者采用流式语法
```yml
- [2080Ti, 3090, 4090]
- [a, b, c]
```
上面两种 yaml 文件，python 解析出来都是
```python
[['2080Ti', '3090', '4090'], ['a', 'b', 'c']]
```

## YAML多文档格式

用 `---` 分隔，可以在一个文件里塞多个独立文档。这是YAML特有的功能
```yml
- 2080Ti
- 3090
- 4090
---
- a
- b
- c
```
解析时要用 `safe_load_all`
```python
docs = list(yaml.safe_load_all(text))
print(docs)
# [['2080Ti', '3090', '4090'], ['a', 'b', 'c']]
print(docs[0])  # ['2080Ti', '3090', '4090']
print(docs[1])  # ['a', 'b', 'c']
```
python会把多个文档解析到一个列表中，每个元素代表一个文档。

## <font color="grape">同一份数据三种写法(YAML, JSON, Python)</font>

YAML
```yml
cluster:
  name: HeteroServe
  gpus:
    - 2080Ti
    - 3090
```

JSON
```JSON
{
  "cluster": {
    "name": "HeteroServe",
    "gpus": ["2080Ti", "3090"]
  }
}
```

Python
```python
{
    "cluster": {
        "name": "HeteroServe",
        "gpus": ["2080Ti", "3090"]
    }
}
```
三者完全等价，只是符号风格不同——YAML 用缩进和 `-`、`:`，JSON 和 Python 用 `[]`、`{}`。

## <font color="grape">YAML支持JSON格式</font>

YAML 是 JSON 的超集，所以下面这种写法在 YAML 里**完全合法**
```yml
cluster: {name: HeteroServe, gpus: [2080Ti, 3090]}
```

这叫 **flow style**（流式风格），和默认的 **block style**（块式风格，靠缩进）可以混用。短小的列表/字典经常用流式写法，更紧凑：

```yml
model:
  name: Qwen3-32B-AWQ
  dtype: float16
  gpu_ids: [0, 1, 2]        # flow style，比展开三行更清爽
  tags: {team: infra, env: prod}
```

所以 ，可以把 YAML 理解为：**JSON 的语法 + 缩进风格的可选简写**。掌握 `-` ↔ `[]`、`:` ↔ `{}` 这层对应。
