# HeMusic 第三方音源接入标准 (Integration Standard)

本文档基于 `HeMusic v0.4 Alpha` 源代码生成，详细定义了客户端与服务端之间的通信协议及职责划分。

**重要提示：**

1. **鉴权限制**：客户端仅向官方域名 (`mindrift.cn`, `music.mindrift.cn`) 发送 `Authorization`、`X-Device-Id` 和 `X-License` 等鉴权头。**第三方私有服务端将不会收到这些 Headers**，请在设计服务端时无需强制校验鉴权（或需修改客户端源码白名单）。
2. **混合模式**：客户端采用了“服务端解析”与“客户端直连”混合的模式。并非所有请求都会流向服务端。

---

## 1\. 职责划分 (Responsibilities)

|功能模块|处理方式|备注|
|-|-|-|
|**搜索 (Search)**|**客户端直连**|客户端直接调用 Tencent/Netease/Kugou 的公开 API，**不经过服务端**。|
|**音乐地址 (Music URL)**|**服务端解析**|所有源 (`tx`, `wy`, `kg`) 均请求服务端 `/resolve` 接口。|
|**歌词 (Lyric) - TX/WY**|**服务端解析**|优先请求服务端 `/resolve`，失败后客户端尝试直连。|
|**歌词 (Lyric) - KG**|**客户端直连**|酷狗源歌词完全由客户端本地解析，**不发送给服务端**。|
|**主题 (Theme)**|**服务端获取**|请求服务端 `/api/theme/list/{id}`。|

---

## 2\. 服务端 API 标准

如果您要开发一个兼容 HeMusic 的服务端，您只需要实现以下接口。

### 2.1 媒体解析接口 (`/resolve`)

用于解析音乐播放地址和（部分）歌词。

* **URL**: `/resolve`
* **Method**: `POST`
* **Headers**: 无鉴权头 (第三方服务端)
* **Content-Type**: `application/json`

#### 请求体 (Request Body)

```json
{
  "action": "musicUrl" | "lyric",
  "source": "tx" | "wy" | "kg",
  "quality": "128k",
  "musicInfo": {
    // 关键差异点：不同源使用不同的 ID 字段
    "songmid": "004UgXW..."  // 仅当 source 为 "tx" 或 "wy" 时
    "hash": "A1B2C3..."      // 仅当 source 为 "kg" 时
  }
}
```

#### 2.1.1 获取音乐地址 (`action: "musicUrl"`)

* **所有源 (`tx`, `wy`, `kg`)** 都会调用此接口。
* **响应 (Response)**:

&nbsp;   ```json
    {
      "code": 200,
      "data": {
        "url": "http://stream.example.com/file.mp3"
      }
    }
    // 客户端兼容性说明：
    // 客户端会依次尝试读取：
    // 1. response.data.url
    // 2. response.data (如果data直接是字符串url)
    // 3. response.url
    ```

## 3\. 数据结构

### 歌曲信息对象

```json
{
  "id": "歌曲ID (songmid 或 hash)",
  "source": "平台代码 (tx/wy/kg)",
  "title": "歌曲名",
  "artist": "歌手",
  "url": "播放地址 (获取后)",
  "isDownloaded": false
}
```

