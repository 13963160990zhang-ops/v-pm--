# V消息推送配置

## 账号信息
- **公共号账号**：PN000824
- **账号名称**：联盟产品数据AI智能体
- **推送目标群**：groupId=9480745, groupCode=5cf6ad1a-d1f0-4f57-85be-73fbfe170bed

## 接口与密钥
| 接口 | URL | AES密钥 |
|------|-----|---------|
| 单聊消息 | https://vchat.vivo.xyz:8443/module-pn/v1/ext/pn/msg/send | HUGPXOMEkPGEoMfg |
| 群聊消息 | https://vchat.vivo.xyz:8443/module-pn/v1/ext/pn/group/msg/send | HRHRPFP1bcMMHEbu |
| 文件上传 | https://vchat-work.vivo.xyz/module-file/v1/common/exter/file/upload | 0QUSOEFMQxbkQyWP |

- **AES向量(IV)**：2098432527847288（固定值，所有接口共用）
- **加密方式**：AES-CBC, pkcs7padding, 128-bit, base64输出, UTF-8编码

## API调用限制
- 消息接口：4000次/天
- 文件上传：1000次/天
- 消息大小：4000字节(UTF-8)

## 推送流程

### 1. 上传图片（如需推送图片）
- 请求：POST 文件上传接口，Content-Type: multipart/form-data
- Headers：acc（账号PN000824）、ec（用文件上传密钥AES加密账号后的base64值）
- Body：file 字段为图片二进制数据
- 返回：returnMsg 包含 fileUrl, fileId, fileName, fileSize, imageWidth, imageHeight, isAuthSdFile

### 2. 发送群聊图片消息
- 请求：POST 群聊消息接口，Content-Type: application/json
- Body：{"account": "PN000824", "params": "AES加密后的参数JSON"}
- params JSON 结构：
  - account: PN000824
  - groupId: 9480745
  - groupCode: 5cf6ad1a-d1f0-4f57-85be-73fbfe170bed
  - chatType: IMAGE
  - msgText: JSON字符串，包含 imageWidth, fileName, fileSize, isAuthSdFile, fileThumbPath(null), filePath, imageHeight, fileId
  - ts: 毫秒级时间戳
- 注意：params 整体用群聊密钥加密

### 3. 发送群聊文本消息
- 与图片消息结构相同，区别：
  - chatType: TEXT
  - msgText: 纯文本内容（不超过4000字节）

### 推送顺序建议
1. 先上传图片
2. 发送图片消息
3. 发送文本说明消息（数据解读、关键指标等）

## Python代码示例

```python
import json, base64, time, requests
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
import urllib3
urllib3.disable_warnings()

ACCOUNT = 'PN000824'
AES_IV = '2098432527847288'
UPLOAD_KEY = '0QUSOEFMQxbkQyWP'
GROUP_KEY = 'HRHRPFP1bcMMHEbu'
UPLOAD_URL = 'https://vchat-work.vivo.xyz/module-file/v1/common/exter/file/upload'
GROUP_URL = 'https://vchat.vivo.xyz:8443/module-pn/v1/ext/pn/group/msg/send'
GROUP_ID = 9480745
GROUP_CODE = '5cf6ad1a-d1f0-4f57-85be-73fbfe170bed'

def aes_encrypt(data, key):
    cipher = AES.new(key.encode(), AES.MODE_CBC, AES_IV.encode())
    padded = pad(data.encode(), AES.block_size, style='pkcs7')
    return base64.b64encode(cipher.encrypt(padded)).decode()

def upload_image(image_path):
    ec = aes_encrypt(ACCOUNT, UPLOAD_KEY)
    headers = {'acc': ACCOUNT, 'ec': ec}
    with open(image_path, 'rb') as f:
        filename = image_path.replace('\\', '/').split('/')[-1]
        files = {'file': (filename, f, 'image/png')}
        resp = requests.post(UPLOAD_URL, files=files, headers=headers, timeout=60, verify=False)
    result = resp.json()
    if result.get('msgCode') != 200:
        raise Exception(f'upload failed: {result}')
    return result['returnMsg']

def send_group_image(file_info):
    msg_text = json.dumps({
        'imageWidth': file_info['imageWidth'],
        'fileName': file_info['fileName'],
        'fileSize': file_info['fileSize'],
        'isAuthSdFile': file_info.get('isAuthSdFile', 1),
        'fileThumbPath': None,
        'filePath': file_info['fileUrl'],
        'imageHeight': file_info['imageHeight'],
        'fileId': file_info['fileId']
    }, ensure_ascii=False)
    params = {
        'account': ACCOUNT, 'groupId': GROUP_ID,
        'groupCode': GROUP_CODE, 'chatType': 'IMAGE',
        'msgText': msg_text, 'ts': int(time.time() * 1000)
    }
    encrypted = aes_encrypt(json.dumps(params, ensure_ascii=False), GROUP_KEY)
    resp = requests.post(GROUP_URL, json={'account': ACCOUNT, 'params': encrypted},
                         headers={'Content-Type': 'application/json'}, timeout=10, verify=False)
    return resp.json()

def send_group_text(text):
    params = {
        'account': ACCOUNT, 'groupId': GROUP_ID,
        'groupCode': GROUP_CODE, 'chatType': 'TEXT',
        'msgText': text, 'ts': int(time.time() * 1000)
    }
    encrypted = aes_encrypt(json.dumps(params, ensure_ascii=False), GROUP_KEY)
    resp = requests.post(GROUP_URL, json={'account': ACCOUNT, 'params': encrypted},
                         headers={'Content-Type': 'application/json'}, timeout=10, verify=False)
    return resp.json()
```

## 注意事项

1. **三个接口三个密钥**：单聊、群聊、文件上传各有独立的AES密钥，不可混用
2. **加密对象**：文件上传接口只加密账号（acc），消息接口加密整个params JSON
3. **推送顺序**：必须先上传图片获得fileUrl等信息，再发送IMAGE消息
4. **群聊可见范围**：公共号发送的消息，群内所有成员可见
5. **依赖库**：需要安装 pycryptodome（pip install pycryptodome）和 requests
6. **SSL验证**：接口使用HTTPS，测试环境可能需要 verify=False
7. **时间戳**：ts 为毫秒级时间戳（int(time.time() * 1000)）
8. **消息大小**：msgText 不超过4000字节（UTF-8编码），长文本需拆分多条发送