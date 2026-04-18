# BeautifulSoup 节点导航笔记

## 获取子节点 (Children)

在 BeautifulSoup 中，可以通过以下属性访问当前节点的子层级。

- **`.contents`**: 返回一个包含所有子节点的 **列表**（包括字符串和标签）。
- **`.children`**: 返回一个 **生成器**（Generator），适合处理子节点数量较多的情况。
- **`.descendants`**: **子孙节点迭代器**。递归遍历所有子节点、子节点的子节点等。

```python
p_tag = soup.p

print(p_tag.contents) 

for child in p_tag.children:
    print(child)
```

## 获取兄弟节点 (Siblings)

兄弟节点是指与当前节点处于同一层级的节点。

- **`.next_sibling`**: 下一个兄弟节点。
- **`.previous_sibling`**: 上一个兄弟节点。
- **`.next_siblings`**: 当前节点之后的所有兄弟节点的 **生成器**。
- **`.previous_siblings`**: 当前节点之前的所有兄弟节点的 **生成器**。

> **注意**：由于 HTML 源码中存在换行符，`.next_sibling` 往往返回的是 `\n`（字符串节点），而非下一个 HTML 标签。

```python
for sibling in soup.p.next_siblings:
    if sibling.name is not None: # 过滤掉换行符字符串
        print(sibling)
```
## 获取父节点 (Parents)

- **`.parent`**: 获取直接父节点。
- **`.parents`**: 递归获取所有父辈节点（从直接父节点一直到 `html` 和 `[document]`）。

## 节点导航常用属性对照表

| **导航属性**            | **描述**    | **返回类型**         |
| ------------------- | --------- | ---------------- |
| `tag.contents`      | 获取直接子节点   | List             |
| `tag.children`      | 获取直接子节点   | Generator        |
| `tag.descendants`   | 获取所有子孙节点  | Generator        |
| `tag.next_sibling`  | 下一个兄弟节点   | Element / String |
| `tag.next_siblings` | 之后的所有兄弟节点 | Generator        |
| `tag.parent`        | 直接父节点     | Element          |
| `tag.parents`       | 所有父辈节点    | Generator        |
