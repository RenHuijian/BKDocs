### 请求地址

/api/c/compapi/v2/cc/search_group_privilege/

### 请求方法

POST

### 功能描述

查询分组权限

### 请求参数

#### 通用参数

| 字段 | 类型 | 必选 | 描述 |
|-----------|------------|--------|------------|
| bk_app_code | string | 是 | 应用 ID |
| bk_app_secret| string | 是 | 安全密钥(应用 TOKEN)，可以通过 蓝鲸智云开发者中心 -&gt; 点击应用 ID -&gt; 基本信息 获取 |
| bk_token | string | 否 | 当前用户登录态，bk_token 与 bk_username 必须一个有效，bk_token 可以通过 Cookie 获取 |
| bk_username | string | 否 | 当前用户用户名，应用免登录态验证白名单中的应用，用此字段指定当前用户 |

#### 接口参数

| 字段 | 类型 | 必选 | 描述 |
|----------------------|------------|--------|-------------|
| bk_supplier_account | string | 是 | 开发商账号 |
| group_id | string | 是 | 分组 ID |

### 请求参数示例

```json
{
    "bk_supplier_account":"0",
    "group_id":"test"
}
```

### 返回结果示例

```json
{
    "result": true,
    "code": 0,
    "message": "",
    "data": {
         "group_id":1,
         "sys_config":{
             "global_busi":[
                 "resource"
             ],
             "back_config":[
                 "event",
                 "model",
                 "audit"
             ]
         },
         "model_config":{
             "network":{
                 "router":[
                     "update",
                     "delete"
                 ]
             }
         }
    }
}
```

### 返回结果参数说明

| 字段 | 类型 | 描述 |
|-----------|-----------|-----------|
| result | bool | 请求成功与否，true:请求成功，false:请求失败 |
| code | string | 组件返回错误编码，0 表示 success，>0 表示失败错误 |
| message | string | 请求失败返回的错误消息 |
| data | object | 请求返回的数据 |

#### data

| 字段 | 类型 | 描述 |
|---------------|----------|----------|
| group_id | string | 分组 ID |
| sys_config | object | 系统配置 |
| back_config | object | 后台配置 |
| model_config | object | 模型配置 |


#### sys_config  目前仅有global_busi

| 名称 | 类型 | 描述 |
|---------|--------|------------|
| resource| string | 主机资源池 |

#### back_config

| 名称 | 类型 | 描述 |
|---------|--------|--------------|
| event | string | 事件推送配置 |
| model | string | 模型配置 |
| audit | string | 审计配置 |

#### model_config

| 名称 | 类型 | 描述 |
|--------|--------|------|
| create | string | 新增 |
| update | string | 编辑 |
| delete | string | 删除 |
| search | string | 查询 |
