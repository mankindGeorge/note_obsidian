# Flask 基础入门笔记

## 1. 核心启动程序 (app.py)

Flask 应用实例的初始化、路由定义与服务配置。

```python
from flask import Flask, render_template, request

app = Flask(__name__)

# 1. 基础路由：返回纯文本或 JSON
@app.route('/')
def index():
    return "Hello Flask 后端服务已启动！"

# 2. 接口路由：返回字典时 Flask 会自动转换为 JSON 格式
@app.route('/api/hello')
def hello():
    return {
        "code": 200,
        "msg": "success",
        "data": "Hello, API"
    }

if __name__ == '__main__':
    # 参数说明：
    # debug=True：开发模式，修改代码自动刷新
    # host='0.0.0.0'：允许本机及局域网内其他设备访问
    # port=5000：监听端口
    app.run(debug=True, host='0.0.0.0', port=5000)
```
## 2. 网页模板渲染 (Jinja2)

- **目录要求**：HTML 文件必须存放在项目根目录下的 `templates` 文件夹内。
- **渲染函数**：使用 `render_template` 将后端变量传递至前端。

**templates/hello.html**

```HTML
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Hello Page</title>
</head>
<body>
    <h1>
        {% if name %}
            Hello, {{ name }}!  {# 使用 {{ }} 获取变量 #}
        {% else %}
            Hello, Flask!
        {% endif %}
    </h1>
</body>
</html>
```
## 3. 静态文件处理与 url_for 函数

- **目录要求**：图片、CSS、JS 文件存放在 `static` 文件夹内。
- **引用方式**：优先使用 `url_for` 动态生成路径，增强代码可移植性。

|**引用方式**|**示例代码**|
|---|---|
|**相对路径**|`<img src="/static/images/test.jpg">`|
|**url_for 函数**|`<img src="{{ url_for('static', filename='head.webp') }}">`|

**url_for 参数对照：**

- **普通路由**：`url_for('index')` 对应视图**函数名** `index`。
- **静态文件**：`url_for('static', filename='...')` 固定端点为 `static`。
- **图片缩放**：`style='object-fit: cover'` 可实现不改变比例的裁剪填充。
## 4. 请求处理：GET 与 POST

通过 `methods` 参数控制路由允许的请求类型，并利用 `request` 对象获取表单数据。

**app.py 逻辑：**

```Python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        # 获取表单中 name="username" 的输入值
        username = request.form.get('username')
        password = request.form.get('password')

        if username == 'admin' and password == '123456':
            return f'登录成功！欢迎 {username}'
        return '用户名或密码错误'
        
    # GET 请求时返回登录页面
    return render_template('login.html')
```

**templates/login.html 关键要素：**

- **form 标签**：必须设置 `method="post"` 和 `action="/login"`。
- **input 标签**：必须设置 `name` 属性，后端根据此属性名接收数据。
### 5. 目录结构参考

```Plaintext
/Project_Root
    ├── app.py             (主程序)
    ├── static/            (存放图片/CSS/JS)
    │   └── head.webp
    └── templates/         (存放 HTML 模板)
        ├── hello.html
        └── login.html
```

# Flask 蓝图 (Blueprint)

蓝图是 Flask 提供的一种模块化管理代码的方式。它允许你将相关的视图函数、模板和静态文件组织成独立的分块，解决主程序 `app.py` 过大且难以维护的问题。

## 1. 蓝图的核心作用

- **模块化**：将应用拆分为用户模块、商品模块、后台管理模块等。
- **路径前缀**：为整个模块的路由统一添加前缀（如 `/auth/login`, `/auth/register`）。
- **独立资源**：每个蓝图可以拥有独立的 `templates` 和 `static` 文件夹。

## 2. 创建蓝图模块

通常在项目中新建一个文件夹或文件（例如 `auth.py`）来定义蓝图。

**auth.py**

```Python
from flask import Blueprint

# 1. 初始化蓝图对象
# 参数：蓝图名称, 导入名, 路由前缀
auth_bp = Blueprint('auth', __name__, url_prefix='/auth')

# 2. 定义路由
@auth_bp.route('/login')
def login():
    return "这是登录页面"

@auth_bp.route('/register')
def register():
    return "这是注册页面"
```

## 3. 在主程序中注册蓝图

只有将蓝图注册到 `app` 实例中，这些路由才会生效。

**app.py**

```Python
from flask import Flask
from auth import auth_bp  # 导入刚才创建的蓝图

app = Flask(__name__)

# 注册蓝图
app.register_blueprint(auth_bp)

if __name__ == '__main__':
    app.run(debug=True)
```

## 4. 蓝图中的 url_for 反向解析

在蓝图内部或外部使用 `url_for` 时，必须加上蓝图的名称前缀。

|**场景**|**语法**|**解析结果**|
|---|---|---|
|**解析蓝图路由**|`url_for('auth.login')`|`/auth/login`|
|**同一蓝图内跳转**|`url_for('.login')`|`/auth/login` (点号表示当前蓝图)|

## 5. 模块化项目结构参考

使用蓝图后的推荐目录结构：

```Plaintext
/Project_Root
    ├── app.py              (主启动文件，负责注册蓝图)
    ├── auth.py             (用户认证相关蓝图)
    ├── goods.py            (商品业务相关蓝图)
    ├── static/             (公共静态资源)
    └── templates/          (公共模板)
```

## 6. 蓝图独立资源目录

如果在创建蓝图时指定了模板目录，Flask 会优先在全局 `templates` 找，找不到再去蓝图目录找。

```Python
# 指定独立的模板和静态资源路径
admin_bp = Blueprint('admin', __name__, 
    template_folder='admin_templates',
    static_folder='admin_static')
```