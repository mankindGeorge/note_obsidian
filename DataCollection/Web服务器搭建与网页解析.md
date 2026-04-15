# flask
第一个启动程序：
主程序app.py
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
