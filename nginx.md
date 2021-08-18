# nginx

## nginx指定lua包的位置

lua_package_path "/export/servers/nginx/lualib/?.lua;/export/App/gw.api.jd.local/current_limiting/?.lua;;";

## nginx中lua使用nginx共享变量

首先要在配置文件中配置共享区域：

lua_code_cache on;

lua_shared_dict dict 100m;  # 会创建一个名为dict的100m大小的共享区域。

有了以上配置，就可以直接在lua中进行使用:

```lua
local ngx_shared = ngx.shared
ngx_shared[name]:get(key)
ngx_shared[name]:set(key, value, timeout)
```

## 通过lua来控制nginx是否打印访问日志

set_by_lua_file $condition '/export/App/gw.api.jd.local/current_limiting/current/runner/logByLuaFile.lua';

access_log      /export/Logs/gw.api.jd.local_access.log log_json if=$condition;

以上两条语句通过lua定义了一个条件，**每次访问**的时候都会获取该条件，如果条件为1则打印日志，为0则不打印日志。

