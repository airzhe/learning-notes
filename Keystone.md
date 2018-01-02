# Keystone

Keystone（OpenStack Identity Service）是OpenStack框架中，负责身份验证、服务规则和服务令牌的功能。用户从 Keystone 获取 Token，在 Token 中 提取 TokenId 以及服务列表；用户访问服务时携带TokenID；相关的服务使用过滤器 authtoken 向 Keystone 验证 TokenID 是否有效，同时返回用户 Token 信息，重新将用户、服务目录等字段添加到 HTTP 头。



### Role

安全包含两部分：Authentication  (认证）和 Authorization （鉴权）

- Authentication 解决的是 “你是谁”的问题

- Authorization 解决的是“你能干什么”的问题


Keystone 是通过 Roel 来实现 Authorization 的，Service 决定每个 Role 能做什么事情。

>一个角色，定义了用户在不同的服务中想用的权限。每一个服务都有自己的 policy.json 文件，该文件 指定了各种规则，每中规则对应着一种或者多种角色。而每个服务中的大部分方法，都声明使用 policy.json 文件中定义的某种规则。只有用户的角色满足了这样的规则，方法才能成功执行，否则会抛出未批准的异常。



### Tenant 

租户，相当于组织。例如：一个用户相当于一个人，一个租户相当于个一单位，一个人只有从属于某一个租户，才能获得相应的权限。



### Token

Token 验证 一是谁，二是否有权限操作 ，Keystone 支持两种保存 Token 的方式

- 使用 UUID 方式时，Token 的 ID 是随机生成的。
- 使用 PKI 方式时，Token 的 ID 是通过 openssl 加密 Token 的内容得到。

```
if config.token_format == 'UUID':
	token_id = uuid.uuid4().hex
elif config.token_format == 'PKI':
	token_id = cms.cms_sign_token(json_dumps(token_data),config.certfile,config.keyfile)
```

> 对于 PKI 格式的Token，可以通过验证证书文件来检验 Keystone 服务器的真实性。对于 UUID 格式的Token，必须向 Keystone 服务器发送验证请求。

cms_to_ken 把 cms 的签名结果转化为 Keystone 需要的结果，例如：删除换行符、将 ”/” 转化为 “-”



### 用户认证

用户的认证方式主要有两种

- 用户名密码认证，这通常用户客户端找不到用户的 Token 或者保存的 Token 失效的情况，本地认证比较慢，因为涉及到很多数据库查询的操作。
- Token 认证是指通过已经存在的 Token 来认证用户的身份，是最快捷的认证方式，在Keystone 客户端会通过 keyring 来保存用户的 Token。




### 角色管理

对于用户信息的更改，势必会涉及到对 Token 的删除操作，如果某一用户在某一租户下的角色发生变化，必须删除对应的用户 Token。

remove_role_from_user 方法主要做了3件事

1.  删除用户角色
2.  如果用户在租户中没有被赋予任何角色，则将其从租户中移除。
3.  删除用户在该租户下的 Token




### 权限管理

获取用户id，获取租户id，获取用户在租户中的角色，检查权限

```
def enfore(credentials,action,target) :
	init()  							#加载policy.json 文件
	match_list = ('rule:%s' % action,) 	#构造规则元组
```



### Policy 基于规则的身份验证引擎

policy.json 定义的 3 类规则


| 规则名        | 描述                                       |
| ---------- | :--------------------------------------- |
| role 规则    | 属于最基本的规则。用于判断 Token 的 roles 属性受满足要求      |
| generic 规则 | 数组最基本的规则。用于判断 Token 的其他属性是否满足要求          |
| rule 规则    | rule 规则相对复杂，它通常是一条规则链。一条规则链可以由若干条规则子链组成。一条规则子链可以由若干条规则组成。这样递归下去，便可以构造出许多赋值的规则。对于一条规则链，只需要其中的一条规则子链通过，就认为这条规则链通过了。对于一条规则子链，只有其中的所有规则满足，才认为规则子链满足（规则链中的子链之间是“或”运算，规则子链中的规则之间是“与”关系） |

> 例如 rule:admin_required 规则，它的定义为 规则链 [["role:admin"],["is_admin:1"]]。这条规则有两条规则子链，即 ["role:admin"] 和 ["is_admin:1"]。这两条子链中，各有一条规则。其中，role:admin 为 role 规则，is_admin:1 为generic 规则。当用户的Token 角色列表包含有admin 角色，或者Token的is_admin属性为1时（通过admin_token），该 Token 通过 rule:admin_required 规则。



### Token管理

auth_token_data 包含 用户信息，租户信息，角色id列表以及用户-租户的附加信息，和到期时间。

Token 是通过 token_api.create_token 方法存储的，drirver 默认为 keystone.token.backends.kvs.Token 

由于原始的 token_id 是通过 openssl 对 Token 进行签名的结果，因此通常比较长，不适合作为字典的key，因此在保存前，需要先对 token_id 做一个 hash 操作。

> 在服务器端，可以通过 token_id 来判断 Token 是否被撤销（删除）的，未被删除的token_id 以”token“ 开头，被删除的以 ”revoked-token“ 开头。



### 服务的安全认证

1. 调用 get_admin_token 方法获取管理员用户的 Token ID，首先查看保存的Token 是否过期，如果即将过去，使用 api-paste.ini (如 /etc/nova/api-pase.ini) 中指定的用户名和密码向 Keystone 申请新的Token 并保存在本地。
2. 调用_json_request 方法 向 Keystone 服务器发送请求，验证用户的Token

```
# 以管理员身份构造 HTTP 头
heders = {'X-Auth-Token':self.get_admin_token()}
response,data = self._json_request('GET','/v2.0/tokens/%s' % safe_quote(user_token))
```



### 服务端自己端验证Token （缺点是用户角色发生改变，服务端不知道）

客户端发送用户名密码到 Keystone 验证，生成Token

客户端发送 API 请求到服务端点，服务端点提取令牌信息，用本地存放的签名公钥证书进行签名。

服务端点处理合法的请求，拒绝验证未通过的请求




### Keystone 资源Controller 对象一览

tenant_controller

user_controller

role_controller

service_controller

endpoint_controller



### public_service（共用服务）应用程序实现的主要url映射

| url                          | http方法 | 调用的 controller 方法                 | 描述            |
| ---------------------------- | :----- | :-------------------------------- | :------------ |
| /tokens                      | POST   | token_controller.authenticate()   | 用户名密码方式认证     |
| /tokens/{token_id}           | GET    | token_controller.validate_token() | 验证token 合法性   |
| /tokens/{token_id}/endpoints | GET    | token_controller.endpoints()      | 查询token可访问端点  |
| /certificates/ca             | GET    | token_controller.ca_cert()        | 查询CA          |
| /certificates/signing        | GET    | token_controller.signing_cert()   | 查询Keystone的证书 |



### admin_servivce（管理服务）应用程序添加的url映射

| url                                  | http方法 | 调用的 controller 方法 | 描述          |
| ------------------------------------ | :----- | :---------------- | :---------- |
| /users/{user_id}                     | POST   | get_user()        | 根据id查询用户    |
| /tenants/{tenant_id}/{user_id}/roles | GET    | get_roles()       | 查询用户在租户中的角色 |



### Keystone 中的重要api一览

| 名称           | 对应的类                      | 默认额driver                              | 描述        |
| ------------ | :------------------------ | :------------------------------------- | :-------- |
| catalog_api  | keystone.catalog.Manager  | keystone.catalog.backenes.sql.Catelog  | 服务端点管理    |
| identity_api | keystone.identity.Manager | keystone.catalog.backenes.sql.Identity | 用户、租户管理   |
| toke_api     | keystone.token.Manager    | keystone.catalog.backenes.kvs.Token    | 用于Token管理 |
| policy_api   | keystone.policy.Manager   | keystone.catalog.backenes.sql.Policy   | 用于权限管理    |



### 其他

admin_token 方式认证，在Keystone 刚刚安装好的时候，数据库是没有用户信息的，因此没有办法使用户名密码方式认证，这时就需要使用 admin_token 认证方式，其实非常简单，只是看客户端传来的Token与配置文件里配置的 admin_token 是否一致。



### 待补充：

OpenStack 各个模块与 Keystone 的交互




### 参考：

每天五分钟玩转 OpenStack

OpenStack 设计与实现（第2版）

OpenStack 开源云 王者归来
