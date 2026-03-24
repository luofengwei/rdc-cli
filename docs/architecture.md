# rdc-cli 技术架构

## 整体架构：三层 Client-Server

```
┌─────────────────────────────────────────────────────┐
│  用户终端                                             │
│  $ rdc open test.rdc                                 │
│  $ rdc events --type draw                            │
│  $ rdc shader 42 ps                                  │
│  $ rdc close                                         │
└──────────────┬──────────────────────────────────────┘
               │
     ┌─────────▼─────────┐
     │  第1层: CLI命令     │  commands/*.py (Click框架)
     │  构造参数 → call()  │
     └─────────┬─────────┘
               │ JSON-RPC over TCP (localhost)
     ┌─────────▼──────────────┐
     │  第2层: Daemon后台进程    │  daemon_server.py (独立子进程)
     │  持有GPU回放状态          │  持续运行，直到 close/超时
     │  分发请求到 handler       │
     └─────────┬──────────────┘
               │ 直接Python调用
     ┌─────────▼──────────────┐
     │  第3层: renderdoc模块    │  renderdoc.pyd / renderdoc.so
     │  ReplayController API   │  编译自 renderdoc/ 目录
     └────────────────────────┘
```

## 典型命令工作流

### 命令1：`rdc open test.rdc`

一切的起点，启动 daemon 进程：

```
1. commands/session.py::open_cmd()
   │
2. services/session_service.py::open_session()
   ├── pick_port()            → 找一个空闲TCP端口 (如 51234)
   ├── token = random hex     → 生成认证token
   ├── start_daemon()         → 启动子进程:
   │     python -m rdc.daemon_server \
   │       --host 127.0.0.1 --port 51234 \
   │       --capture test.rdc --token abc123
   │
3. daemon_server.py::main() [子进程中执行]
   ├── _load_replay()
   │   ├── discover.find_renderdoc()   → 找到 renderdoc.pyd
   │   ├── rd.InitialiseReplay()       → 初始化RenderDoc
   │   ├── rd.OpenCaptureFile()        → 打开.rdc文件
   │   ├── cap.OpenCapture()           → 创建 ReplayController
   │   └── RenderDocAdapter(controller) → 包装成adapter
   │
   ├── _init_adapter_state()
   │   ├── get_root_actions()          → 获取整个drawcall树
   │   ├── get_resources/textures/buffers → 缓存所有资源
   │   └── build_vfs_skeleton()        → 构建虚拟文件系统
   │
   └── run_server()                    → TCP循环，等待JSON-RPC请求

4. wait_for_ping()   → 父进程等daemon就绪
5. create_session()  → 写 session.json (host/port/token/pid)
```

session.json 保存在 `~/.cache/rdc/sessions/default.json`，后续所有命令都读这个文件来找daemon。

### 命令2：`rdc events --type draw`

查询所有draw类型的事件（纯查询，无GPU操作）：

```
1. commands/events.py::events_cmd(event_type="draw")
   │  构造 params = {"type": "draw"}
   │
2. _helpers.py::call("events", params)
   ├── require_session()       → 读 session.json 拿到 host/port/token
   ├── _request("events", ...) → 构造 JSON-RPC:
   │     {"jsonrpc":"2.0", "method":"events", "id":1,
   │      "params":{"_token":"abc123", "type":"draw"}}
   │
   └── send_request()          → TCP连接daemon，发JSON，等回复

3. [daemon进程内] daemon_server.py
   ├── _handle_request()       → 验token → 查 _DISPATCH["events"]
   │
   └── handlers/query.py::_handle_events()
       ├── _get_flat_actions()         → 递归遍历drawcall树，扁平化
       ├── filter_by_type(flat, "draw") → 按ActionFlags过滤
       │     _DRAWCALL | _MESHDRAW 标志位匹配
       └── 返回 {"events": [{"eid":42, "type":"draw", "name":"DrawIndexed(36)"},...]}

4. [回到CLI进程]
   events_cmd 收到结果 → write_tsv() 输出:

   EID    TYPE    NAME
   42     draw    DrawIndexed(36)
   108    draw    DrawIndexed(72)
   ...
```

### 命令3：`rdc shader 42 ps`

查看 EID=42 处的 pixel shader（涉及GPU回放）：

```
1. commands/pipeline.py::shader_cmd(first="42", second="ps")
   │  解析出 eid=42, stage="ps"
   │
2. call("shader", {"eid": 42, "stage": "ps"})
   │  → JSON-RPC发给daemon

3. [daemon进程内] handlers/query.py::_handle_shader()
   ├── require_pipe(params, state)
   │   ├── adapter.set_frame_event(42)   ← 关键! 移动GPU回放到EID=42
   │   │     即 controller.SetFrameEvent(42, force=True)
   │   │     RenderDoc重新执行到第42个事件，GPU状态就绪
   │   └── adapter.get_pipeline_state()  ← 获取此刻的管线状态
   │         即 controller.GetPipelineState()
   │
   └── query_service.shader_row(42, pipe_state, "ps")
       ├── pipe_state.GetShader(ShaderStage.Pixel)  → shader资源ID
       ├── pipe_state.GetShaderEntryPoint()          → 入口函数名
       └── 统计 RO/RW binding数、cbuffer数

4. 返回:
   EID    STAGE   SHADER    ENTRY    RO   RW   CBUFFERS
   42     ps      <hash>    main     4    1    2
```

### 命令4：`rdc shader-replace 42 ps --source patched.hlsl`

替换shader（目前唯一的"修改回放行为"能力）：

```
1. commands/shader_edit.py::shader_replace_cmd()
   │
2. [daemon进程内] handlers/shader_edit.py::_handle_shader_replace()
   ├── SetFrameEvent(eid)           → 移到目标事件
   ├── GetShader(stage)             → 获取原shader ID
   ├── BuildTargetShader(...)       → 编译新shader
   ├── ReplaceResource(old, new)    → 替换! 后续回放用新shader
   └── 记录到 state.shader_replacements
```

## 关键对象关系

```
DaemonState (daemon_server.py)
├── adapter: RenderDocAdapter
│   └── controller: IReplayController    ← RenderDoc核心，所有GPU操作的入口
│       ├── SetFrameEvent(eid)           回放到指定事件
│       ├── GetRootActions()             获取drawcall树
│       ├── GetPipelineState()           获取当前管线状态
│       ├── GetResources/Textures/Buffers
│       ├── BuildTargetShader()          编译shader
│       ├── ReplaceResource()            替换资源
│       └── RemoveReplacement()          恢复原资源
│
├── cap: ICaptureFile                    ← .rdc文件句柄
├── structured_file                      ← API调用的结构化数据
├── tex_map / buf_map / res_names        ← 缓存
├── shader_meta / disasm_cache           ← shader缓存
└── vfs_tree                             ← 虚拟文件系统 (/draws, /passes, /resources)
```

## 请求分发机制

daemon_server.py 中维护一个全局分发表 `_DISPATCH`，汇聚了14个handler模块：

```python
_DISPATCH: dict[str, Handler] = {
    **_CORE_HANDLERS,       # ping, status, goto, count, shutdown, file_read
    **_QUERY_HANDLERS,      # events, draws, pipeline, shader, resources, passes...
    **_SHADER_HANDLERS,     # shader反汇编
    **_TEXTURE_HANDLERS,    # texture导出
    **_BUFFER_HANDLERS,     # buffer导出
    **_PIPE_STATE_HANDLERS, # 管线状态查询 (topology, blend, stencil...)
    **_DESCRIPTOR_HANDLERS, # 描述符查询
    **_SCRIPT_HANDLERS,     # 自定义脚本
    **_PIXEL_HANDLERS,      # 像素查询
    **_VFS_HANDLERS,        # 虚拟文件系统 (ls, cat, tree)
    **_DEBUG_HANDLERS,      # shader调试 (debug_pixel, debug_vertex)
    **_SHADER_EDIT_HANDLERS,# shader替换
    **_CAPTURE_HANDLERS,    # 抓帧控制
    **_CAPTUREFILE_HANDLERS,# .rdc文件元数据
    **_UNUSED_HANDLERS,     # 未使用资源分析
}
```

每个handler签名统一：

```python
def handler(request_id: int, params: dict[str, Any], state: DaemonState) -> tuple[dict[str, Any], bool]:
    # 返回 (JSON-RPC响应, 是否继续运行daemon)
```

## RenderDoc模块发现

`discover.py` 按以下顺序查找 `renderdoc` Python模块：

1. `RENDERDOC_PYTHON_PATH` 环境变量
2. 系统路径 (`/usr/lib/renderdoc`, `/usr/local/lib/renderdoc`)
3. `renderdoccmd` 可执行文件的同级目录
4. site-packages

使用 subprocess 安全探测，避免导入失败时crash。

## 版本兼容

`RenderDocAdapter`（adapter.py）抹平 RenderDoc 版本差异：

| API差异 | v1.32+ | v1.31及以前 |
|---------|--------|------------|
| 获取drawcall树 | `GetRootActions()` | `GetDrawcalls()` |

## 当前能力边界

| 能力 | 支持 | 说明 |
|------|------|------|
| 查询drawcall/事件 | ✅ | 扁平化遍历 + 类型/名称过滤 |
| 查询管线状态 | ✅ | 任意EID处的完整管线 |
| 查询/导出资源 | ✅ | texture、buffer、shader |
| 替换shader | ✅ | BuildTargetShader + ReplaceResource |
| 禁用/跳过drawcall | ❌ | action树只读，无RemoveAction API |
| shader调试 | ✅ | debug_pixel, debug_vertex, debug_thread |
| 远程回放 | ✅ | 通过RemoteServer代理 |
| Android支持 | ✅ | adb forward + 远程回放 |
