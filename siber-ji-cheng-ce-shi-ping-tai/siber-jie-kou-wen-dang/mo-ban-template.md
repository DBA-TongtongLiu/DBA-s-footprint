# 模板 template

## 模板的增、删、改、查

### request

```text
{
    "manage_mode": "CREATE",
    "check_template": {
        "template_name": "测试",
        "checker": [
            {
                "key": "$ResponseStatus",
                "relation": "=",
                "content": 200
            }
        ]
    }
}
```

manage\_mode：

* CREATE：创建
* QUERY：查询
* UPDATE：更新
* DELETE：删除

### response

```text
{
    "template_id": "60f91dedbdbbe1f8432bf023",
    "template_name": "测试",
    "checker": [
        {
            "key": "$ResponseStatus",
            "relation": "=",
            "content": 200
        }
    ],
    "insert_time": "1626938861",
    "update_time": "1626938861"
}
```



## 模板的 proto 定义

### \[message\]CheckTemplate

```text
message CheckTemplate {
    string template_id = 1;
    string template_name = 2;
    repeated CheckSub checker = 3;
    int64 invalid_date = 4;
    int64 insert_time = 5;
    int64 update_time = 6;
    string user_update = 7;
}
```

### \[message\]ManageCheckTemplateInfo

```text
message ManageCheckTemplateInfo {
    string manage_mode = 1;
    CheckTemplate check_template = 2;
}
```

### \[message\]CheckTemplateList

```text
message CheckTemplateList{
    repeated CheckTemplate check_template = 1;
}
```

### \[message\]CheckTemplateList

```text
message CheckTemplateList{
    repeated CheckTemplate check_template = 1;
}
```

### \[rpc\]ManageCheckTemplate

```text
rpc ManageCheckTemplate(ManageCheckTemplateInfo) returns (CheckTemplate){
    option (google.api.http) = {
            post:"/siberhttp/manage/check_template"
            body:"*"
        };
}
```

### \[rpc\]ManageTemplateList

```text
rpc ManageTemplateList (FilterInfo) returns (CheckTemplateList){
        option (google.api.http) = {
             post:"/siberhttp/list/check_template"
             body:"*"
         };
    }
```

