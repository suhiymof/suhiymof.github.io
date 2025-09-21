---
layout:     post
title:      "skynet收到tcp数据处理流程"
date:       2025-09-21
author:     "suhiymof"
header-img: "img/post-bg-2015.jpg"
tags:
    - skynet
---

1. <span id = "jump1">`socket.lua` 会注册 `PTYPE_SOCKET` 类型消息的处理</span>

```lua {.line-numbers}
skynet.register_protocol {
	name = "socket",
	id = skynet.PTYPE_SOCKET,	-- PTYPE_SOCKET = 6
	unpack = driver.unpack,--把struct skynet_socket_message内的数据提取出来 即 type id ud buffer 注意type不同 后面参数的意义也是不同的 对于收到 SKYNET_SOCKET_TYPE_DATA (id ud buffer) 是 id, size, data
	dispatch = function (_, _, t, ...)  -- t 是 SKYNET_SOCKET_TYPE_DATA
		socket_message[t](...)
	end
}
```

2. 监听到可读后会进入`forward_message_tcp`函数, 此时result各个字段的定义如下
    
```c {.line-numbers}
    struct socket_message {
        int id; // 所属socket
        uintptr_t opaque;   // 需要发送的服务
        int ud;	// 收到数据的大小
        char * data;    // 收到的数据
    };
    static int
    forward_message_tcp(struct socket_server *ss, struct socket *s, struct socket_lock *l, struct socket_message * result)
```

3. 如果一切顺利 函数返回 `SOCKET_DATA ` , skynet_socket_poll() 会进入 `forward_message(SKYNET_SOCKET_TYPE_DATA, false, &result);`, 函数原型

```c {.line-numbers}
static void
forward_message(int type, bool padding, struct socket_message * result)
```

4. 首先会包装成 `skynet_socket_message`,此种情况下各字段定义如下

```c {.line-numbers}
struct skynet_socket_message {
	int type;   // SKYNET_SOCKET_TYPE_DATA
	int id;     // 收到数据的大小
	int ud;     // 收到数据的大小
	char * buffer;  // 收到的数据 
};
```

5. 把 `skynet_socket_message` 附加到 `skynet_message` 的 `data` 字段

```c {.line-numbers}
struct skynet_message {
	uint32_t source;    // 0
	int session;    // 0
	void * data;    // skynet_socket_message
	size_t sz;  // sizeof(skynet_socket_message) | ((size_t)PTYPE_SOCKET << MESSAGE_TYPE_SHIFT);//PTYPE_SOCKET放置在sz的高八位
};
```

6. 调用 `skynet_context_push((uint32_t)result->opaque, &message)` 把消息塞入消息队列， snlua module的回调入口在 `static int cb(struct skynet_context * context, void * ud, int type, int session, uint32_t source, const void * msg, size_t sz)`

```c {.line-numbers}
static int
_cb(struct skynet_context * context, void * ud, int type, int session, uint32_t source, const void * msg, size_t sz) {
	lua_State *L = ud;//这里是主协程
	int trace = 1;
	int r;
	int top = lua_gettop(L);
	if (top == 0) {
		lua_pushcfunction(L, traceback);
		lua_rawgetp(L, LUA_REGISTRYINDEX, _cb);
	} else {
		assert(top == 2);
	}
	lua_pushvalue(L,2);

	lua_pushinteger(L, type);
	lua_pushlightuserdata(L, (void *)msg);
	lua_pushinteger(L,sz);
	lua_pushinteger(L, session);
	lua_pushinteger(L, source);

	r = lua_pcall(L, 5, 0 , trace);//调用lua层的skynet.dispatch_message 其参数是prototype, msg, sz, session, source

	if (r == LUA_OK) {
		return 0;
	}
	const char * self = skynet_command(context, "REG", NULL);
	switch (r) {
	case LUA_ERRRUN:
		skynet_error(context, "lua call [%x to %s : %d msgsz = %d] error : " KRED "%s" KNRM, source , self, session, sz, lua_tostring(L,-1));
		break;
	case LUA_ERRMEM:
		skynet_error(context, "lua memory error : [%x to %s : %d]", source , self, session);
		break;
	case LUA_ERRERR:
		skynet_error(context, "lua error in error : [%x to %s : %d]", source , self, session);
		break;
	};

	lua_pop(L,1);

	return 0;
}
```

7. 看21行的注释, 会调用 `local function raw_dispatch_message(prototype, msg, sz, session, source)`, 因为 `prototype` 是  `PTYPE_SOCKET`,所以最终调用的是 [这里](#jump1) 的 `dispatch`, 传入的参数是 session, source, 和 unpack(msg,sz)([也就是这里](#jump1)  `unpack` 的返回值)

8. `unpack` 代码如下

```c {.line-numbers}
static int
lunpack(lua_State *L) {
	struct skynet_socket_message *message = lua_touserdata(L,1);
	int size = luaL_checkinteger(L,2);

	lua_pushinteger(L, message->type);      // SKYNET_SOCKET_TYPE_DATA
	lua_pushinteger(L, message->id);        // socketid
	lua_pushinteger(L, message->ud);        // 消息大小
	if (message->buffer == NULL) {
		lua_pushlstring(L, (char *)(message+1),size - sizeof(*message));
	} else {
		lua_pushlightuserdata(L, message->buffer);  // 走这里，消息内容
	}
	if (message->type == SKYNET_SOCKET_TYPE_UDP) {
		int addrsz = 0;
		const char * addrstring = skynet_socket_udp_address(message, &addrsz);
		if (addrstring) {
			lua_pushlstring(L, addrstring, addrsz);
			return 5;
		}
	}
	return 4;
}
```

9. 所以传给 `PTYPE_SOCKET` 消息的 `dispatch` 的参数为 `session, source, SKYNET_SOCKET_TYPE_DATA, socketid, size, buffdata`

```lua {.line-numbers}
-- SKYNET_SOCKET_TYPE_DATA = 1
socket_message[1] = function(id, size, data)
	local s = socket_pool[id]
	if s == nil then
		skynet.error("socket: drop package from " .. id)
		driver.drop(data, size)
		return
	end

	local sz = driver.push(s.buffer, s.pool, data, size)    -- 将数据塞入 s.buffer 的结尾
	local rr = s.read_required  -- 调用read的时候会设置,如果 s.buffer 里面没有数据就会挂起
	local rrt = type(rr)
	if rrt == "number" then
		-- read size
		if sz >= rr then
			s.read_required = nil
			if sz > BUFFER_LIMIT then
				pause_socket(s, sz)
			end
			wakeup(s)   -- 唤起之前因为没数据挂起的协程
		end
	else
		if s.buffer_limit and sz > s.buffer_limit then
			skynet.error(string.format("socket buffer overflow: fd=%d size=%d", id , sz))
			driver.close(id)
			return
		end
		if rrt == "string" then
			-- read line
			if driver.readline(s.buffer,nil,rr) then
				s.read_required = nil
				if sz > BUFFER_LIMIT then
					pause_socket(s, sz)
				end
				wakeup(s)
			end
		elseif sz > BUFFER_LIMIT and not s.pause then
			pause_socket(s, sz)
		end
	end
end
```
