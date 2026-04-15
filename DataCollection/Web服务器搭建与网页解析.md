# Flask 基础
## 第一个启动程序

主程序 app.py

```python
from flask import Flask

app = Flask(__name__)

# 首页路由
@app.route('/')
def index():
    return "Hello Flask 后端服务已启动！"

# 示例接口
@app.route('/api/hello')
def hello():
    return {
        "code": 200,
        "msg": "success",
        "data": "Hello, API"
    }

if __name__ == '__main__':
    # 开启调试模式，本地开发用
    app.run(debug=True, host='0.0.0.0', port=5000)
```

**参数说明：**
> `debug=True`：开发模式，修改代码自动刷新，**上线关闭**
> `host='0.0.0.0'`：允许本机/局域网访问
> `port=5000`：端口号，可自定义 8080、3000 等

---

# 网页模板
## 模板文件

templates/hello.html

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Hello Page</title>
</head>
<body>
    <h1>
        {% if name %}
            Hello, {{ name }}!
        {% else %}
            Hello, Flask!
        {% endif %}
    </h1>
</body>
</html>
```

## 主程序

app.py

```python
from flask import Flask, render_template

app = Flask(__name__)

# 首页路由
@app.route('/')
def index():
    return "Flask 后端已启动"

# 带参数的网页模板
@app.route('/hello')
@app.route('/hello/<name>')
def hello(name=None):
    return render_template('hello.html', name=name)

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```

---

# 显示静态图片

**方法一：相对路径**

```html
<img src="/static/images/test.jpg" alt="图片">
```

**方法二：url_for() 函数**

```html
<img src="{{ url_for('static', filename='head.webp') }}" alt="图片">
```

实际路径：`static/head.webp`

---

# 用户提交请求

**说明：**
> GET：显示页面
> POST：接收用户名密码
> 路由：`/login`

## 后端代码

app.py

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        # 获取表单数据
        username = request.form.get('username')
        password = request.form.get('password')

        # 简单判断
        if username == 'admin' and password == '123456':
            return f'登录成功！欢迎 {username}'
        else:
            return '用户名或密码错误'
    return render_template('login.html')

if __name__ == '__main__':
    app.run(debug=True)
```

## 表单页面

templates/login.html

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>
    <h2>用户登录</h2>
    <form action="/login" method="post">
        <div>
            用户名：<input type="text" name="username" required>
        </div>
        <br>
        <div>
            密码：<input type="password" name="password" required>
        </div>
        <br>
        <button type="submit">登录</button>
    </form>
</body>
</html>
```

 **关键说明：**

1. `@app.route('/login', methods=['GET', 'POST'])`
> GET：打开页面
> POST：提交表单

2. HTML 表单关键：
> `method="post"`
> `action="/login"`
> input 必须有 name 属性
