当时写了一个异步的函数，兑换卡密

抽象出来就一个函数

一开始的时候是这样的

```python
async def check_cakey(cakey_request: CAKEYRequest, token: str):
    current_user =get_current_user(token)
    #剩余的省略
```

发现current_user根本不会获取

debug后发现，是因为get_current_user这个函数无法执行

思考，查阅资料后才知道，原来check_cakey用async异步修饰后，里面的函数想要执行必须用await

修改后如此

```python
async def check_cakey(cakey_request: CAKEYRequest, token: str):
    current_user =await get_current_user(token)
    #剩余的省略
```

