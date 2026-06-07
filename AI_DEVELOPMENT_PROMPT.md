
```markdown
# EasyAdmin 后端 API + 后台功能改造 Prompt

当前 AI 后台已经绑定 GitHub 仓库,仓库里已经是解压后的 EasyAdmin 源码,不是压缩包。请直接读取当前仓库代码,先检查项目结构,再在现有 EasyAdmin 源码基础上完成后端 API 和后台功能改造。不要重新创建新项目,不要脱离现有 EasyAdmin 目录结构,不要破坏原后台登录、菜单、权限系统。服务器环境:域名 `https://moe.123vps.cn`,API 基础地址 `https://moe.123vps.cn/api`,PHP `8.4`,MySQL `5.7`,Nginx `1.30`,框架 EasyAdmin/ThinkPHP,数据库字符集 `utf8mb4`,Web 根目录必须指向项目 `public` 目录,所有 App 接口前缀统一为 `/api/`。

## 开发前检查要求

请先读取当前 GitHub 仓库源码,检查项目目录结构、`composer.json`、ThinkPHP 版本、后台控制器目录、模型目录、路由文件位置、数据库配置方式、EasyAdmin 后台菜单和权限添加方式。如果原项目没有 API 模块,请新建 API 控制器目录。所有新增代码必须适配当前 EasyAdmin 后台风格。如果当前依赖不兼容 PHP 8.4,请指出需要修改的 composer 依赖版本,并给出可执行命令。MySQL 5.7 不要使用 MySQL 8 才支持的语法,不要使用窗口函数、CTE 等。建表 SQL 使用 InnoDB + utf8mb4。金额字段统一使用 `decimal(18,2)`。涉及金额变动的操作必须使用数据库事务。

## 质量优先要求

本次开发不限制代码行数、不限制回复字数、不限制实现时间,只考虑最终代码质量和功能完整度。不要为了简短而省略必要代码,不要只给思路,必须给可落地的完整实现,不要用伪代码代替真实代码,不要跳过错误处理、参数校验、事务处理、权限校验和安全处理,不要因为内容较长就省略文件。如果一个文件需要修改,请逐个说明文件路径和完整代码。所有新增和修改都必须能在当前 EasyAdmin 源码中实际运行。优先保证稳定、安全、可维护,其次才考虑代码简洁。如果发现现有源码结构和我的描述不一致,请以仓库实际结构为准,但必须说明原因和调整方案。

## 基础接口规范

所有 App 端接口统一返回 JSON:`{"code":0,"msg":"成功","data":{}}`,`code=0` 表示成功,非 0 表示失败。除注册、登录、找回密码、发送验证码、版本检查、公告/用户须知、支付回调外,其他接口都需要 JWT 鉴权。JWT Token 放在请求头:`Authorization: Bearer <token>`。登录密码、支付密码/提现密码都使用 `password_hash`/bcrypt 存储,不允许存明文。用户状态为封禁时禁止登录和使用需要登录的接口。钱包余额就是 ONE 币,1 ONE = 1 元,充值、提现、转账、红包都使用同一余额,金额展示精确到分。

## 数据库表结构

请新增以下表,表名前缀使用 `app_`,字段按下面要求创建,并给出完整建表 SQL。

`app_user`:用户表。字段:id int 主键自增,username varchar(50) 唯一 用户名注册时自动生成,email varchar(100) 唯一,password varchar(255) 登录密码,pay_password varchar(255) 支付密码/提现密码可为空,balance decimal(18,2) 默认 0.00,wallet_address varchar(18) 唯一 18 位随机大小写字母和数字,status tinyint 默认 1 代表 1 正常 0 封禁,create_time int,update_time int。

`app_email_code`:邮箱验证码表。字段:id int 主键自增,email varchar(100),code varchar(10) 6 位数字验证码,scene varchar(20) 场景 register/login/reset/bind/paypwd,expire_time int 5 分钟有效,used tinyint 默认 0,create_time int。

`app_wallet_log`:钱包流水表。字段:id int 主键自增,user_id int,type varchar(30) 流水类型 recharge/withdraw/withdraw_refund/transfer_in/transfer_out/redpacket_send/redpacket_grab/admin,amount decimal(18,2) 正数增加负数减少,balance_after decimal(18,2),remark varchar(255),related_id int 默认 0,create_time int。

`app_recharge_order`:充值订单表。字段:id int 主键自增,order_no varchar(32) 唯一,user_id int,amount decimal(18,2),pay_type varchar(10) 支付方式 alipay/wxpay,status tinyint 默认 0 代表 0 待支付 1 已支付,trade_no varchar(64) 易支付平台订单号,pay_time int,create_time int,update_time int。

`app_withdraw_account`:提现账号表。字段:id int 主键自增,user_id int,type varchar(10) wechat/alipay,real_name varchar(50),account varchar(100),qrcode varchar(255) 收款码图片地址,create_time int,update_time int。

`app_withdraw_order`:提现订单表。字段:id int 主键自增,order_no varchar(32) 唯一,user_id int,account_id int,amount decimal(18,2),status tinyint 默认 0 代表 0 提现中 1 提现成功 2 已驳回,admin_remark varchar(255),create_time int,handle_time int。

`app_redpacket`:口令红包表。字段:id int 主键自增,user_id int 发红包用户 ID,keyword varchar(50) 红包口令,total_amount decimal(18,2),total_count int,remain_amount decimal(18,2),remain_count int,status tinyint 默认 1 代表 1 进行中 0 已领完,create_time int。

`app_redpacket_record`:红包领取记录表。字段:id int 主键自增,redpacket_id int,user_id int,amount decimal(18,2),create_time int。

`app_config`:系统配置表。字段:id int 主键自增,config_key varchar(100) 唯一,config_value text,remark varchar(255),create_time int,update_time int。注意不要直接使用未转义的 `key` 作为字段名,避免 MySQL 关键字问题。预置配置项:mail_apikey 空,mail_smtp=smtp.163.com,mail_smtp_user=cloud123pro@163.com,mail_smtp_password=JAp6zF3EWV7MNKCq,mail_smtp_port=465,mail_webname=ONE,pay_gateway=https://pay.123vps.cn,pay_pid=e2148016ee1459d1c7d0754eb389046b,pay_key=03a80bdecc82c4062d250cde6fb9d9978f2c49084a1f21a74c95b86c869fda94,jwt_secret 自动生成一串安全随机字符串,jwt_expire=2592000,app_version=1.0.0,app_version_code=100,app_update_content 空,app_force_update=0,app_download_url 空,notice_marquee 空,notice_content 空,user_notice_content 空。

## 第三方邮件接口

邮件 API 地址:`https://api.nanyinet.com/api/gateway/mailer/api.php`,请求方式 POST,Body 类型 form-data 或 application/x-www-form-urlencoded。请求参数:apikey 从后台配置 `mail_apikey` 读取,email 收件人邮箱,content 邮件内容支持 HTML,title 邮件标题,webname 从 `mail_webname` 读取,smtp=`smtp.163.com`,smtp_user=`cloud123pro@163.com`,smtp_password=`JAp6zF3EWV7MNKCq`,smtp_port=`465`。验证码邮件内容示例:`您的验证码：123456，5分钟内有效，请勿泄露。`。请封装 `MailService::send($email,$title,$content)`。发送验证码逻辑:验证邮箱格式,同一邮箱同一场景 60 秒内不能重复发送,验证码为 6 位数字,有效期 5 分钟,验证通过后标记为已使用。

## 第三方易支付接口

支付网关:`https://pay.123vps.cn`,商户 PID:`e2148016ee1459d1c7d0754eb389046b`,商户密钥:`03a80bdecc82c4062d250cde6fb9d9978f2c49084a1f21a74c95b86c869fda94`。支持支付方式:`alipay` 和 `wxpay`。App 创建充值订单时传 `pay_type`,只能是 `alipay` 或 `wxpay`,后端请求易支付时把 `pay_type` 映射到易支付参数 `type`。请使用易支付标准接口,优先使用 `mapi.php` 返回支付二维码或跳转链接给 App。易支付请求参数至少包含:pid,type,out_trade_no,notify_url,return_url,name,money,sign,sign_type。异步通知地址固定为 `https://moe.123vps.cn/api/pay/notify`,同步跳转地址为 `https://moe.123vps.cn/pay/return`。签名规则:除 `sign`、`sign_type` 外所有参数按参数名 ASCII 升序排序,拼接成 `a=b&c=d` 格式,末尾直接拼接商户密钥,MD5 后转小写,`sign_type=MD5`。支付回调逻辑:接收易支付异步通知,校验签名,校验订单是否存在,校验金额是否一致,校验 `trade_status=TRADE_SUCCESS`,保证幂等同一个订单只允许入账一次,把 `app_recharge_order.status` 改为 1,记录 `trade_no` 和 `pay_time`,给用户余额增加对应金额,写入 `app_wallet_log` 且 type=`recharge`,成功输出纯文本 `success`,失败输出纯文本 `fail`。

## App 端 API 接口

`POST /api/send_code`:发送邮箱验证码,无需登录。参数:email,scene。scene 只能是 register/login/reset/bind/paypwd。逻辑:校验邮箱格式和 scene,同一邮箱同一场景 60 秒内不能重复发送,生成 6 位数字验证码,写入 `app_email_code`,调用邮件 API 发送。

`POST /api/register`:注册,无需登录。参数:email,password,password_confirm,code。逻辑:校验邮箱未注册,校验两次密码一致,校验邮箱验证码 scene=register,自动生成唯一用户名,自动生成唯一 18 位钱包地址,初始余额 0.00,注册成功返回 JWT token 和用户信息。

`POST /api/login`:登录,无需登录。支持两种登录方式。密码登录参数:type=password,account 用户名或邮箱,password。验证码登录参数:type=code,email,code。逻辑:密码登录支持用户名或邮箱,验证码登录校验 scene=login,用户被封禁禁止登录,登录成功返回 JWT token 和用户信息。

`POST /api/reset_password`:找回密码,无需登录。参数:email,code,new_password,new_password_confirm。逻辑:校验验证码 scene=reset,校验两次新密码一致,更新登录密码。

`GET /api/user/info`:获取当前用户信息,需要登录。返回 id,username,email,balance,wallet_address,has_pay_password,status。

`POST /api/user/bind_email`:换绑邮箱,需要登录。参数:new_email,code。逻辑:校验验证码 scene=bind,校验新邮箱未被其他用户绑定,更新用户邮箱。

`POST /api/user/set_pay_password`:设置或修改支付密码/提现密码,需要登录。参数:email_code,login_password,new_pay_password,new_pay_password_confirm。逻辑:校验当前用户邮箱验证码 scene=paypwd,校验当前登录密码,校验两次新支付密码一致,新支付密码用 bcrypt 存储,成功后返回 has_pay_password=true。

`POST /api/recharge/create`:创建充值订单/购买 ONE 币,需要登录。参数:amount,pay_type。pay_type 只能是 alipay 或 wxpay。逻辑:校验金额大于 0,生成唯一订单号,写入 `app_recharge_order`,请求易支付接口,返回支付链接或二维码数据给 App。返回字段:order_no,amount,pay_type,pay_url,qrcode,raw。

`POST /api/pay/notify`:易支付异步回调,无需 JWT 鉴权。逻辑:必须校验易支付签名,校验订单金额,处理订单幂等,支付成功后给用户增加余额,成功返回纯文本 success,失败返回 fail。

`GET /api/recharge/list`:获取充值记录,需要登录。参数:page,limit,status。返回 list,total。列表字段:order_no,amount,pay_type,status,trade_no,pay_time,create_time。

`GET /api/wallet/log`:钱包流水,需要登录。参数:page,limit,direction,type。direction=add 查金额大于 0,direction=reduce 查金额小于 0,默认返回当前用户全部流水。返回字段:type,amount,balance_after,remark,related_id,create_time。

`POST /api/transfer`:转账,需要登录。参数:to_wallet_address,amount,pay_password。逻辑:校验当前用户已设置支付密码,校验支付密码,校验金额大于 0,校验对方钱包地址存在,不允许转给自己,校验余额充足,使用数据库事务,当前用户扣款并写 `transfer_out` 流水,对方用户加款并写 `transfer_in` 流水。返回 amount,to_wallet_address,balance。

`GET /api/transfer/list`:转账记录,需要登录。参数:page,limit。逻辑:查询当前用户 `app_wallet_log` 中 type 为 `transfer_in` 和 `transfer_out` 的记录。

`GET /api/withdraw/account/list`:提现账号列表,需要登录。返回当前用户绑定的提现账号列表,字段:id,type,real_name,account,qrcode,create_time。

`POST /api/withdraw/account/add`:新增提现账号,需要登录。参数:type,real_name,account,qrcode。type 只能是 wechat 或 alipay。逻辑:校验 real_name/account/qrcode 不为空,绑定到当前登录用户。

`POST /api/upload`:上传图片,需要登录。参数:file。要求:仅允许 jpg,jpeg,png,webp,文件大小建议限制 5MB 以内,上传到 `public/uploads/app/`,返回完整 URL,用于提现收款码。

`POST /api/withdraw/create`:发起提现,需要登录。参数:account_id,amount,pay_password。逻辑:校验提现账号属于当前用户,校验当前用户已设置支付密码,校验支付密码,校验金额大于 0,校验余额充足,使用数据库事务,先扣除用户余额,写钱包流水 type=withdraw 且金额为负数,创建提现订单 status=0 提现中,后台管理员后续人工审核打款。

`GET /api/withdraw/list`:提现记录,需要登录。参数:page,limit,status。返回 summary 和 list。summary 包含 available 当前钱包余额,withdrawing 提现中订单金额合计,success 提现成功订单金额合计。list 字段:order_no,amount,status,account,admin_remark,create_time,handle_time。

`POST /api/redpacket/send`:发口令红包,需要登录。参数:keyword,total_amount,total_count,pay_password。逻辑:校验已设置支付密码,校验支付密码,校验口令不能为空,总金额大于 0,个数大于 0,总金额不能小于红包个数 * 0.01,余额充足,使用事务扣除发红包用户余额,创建红包记录,写钱包流水 type=redpacket_send 金额为负数。

`POST /api/redpacket/grab`:领取口令红包,需要登录。参数:keyword。逻辑:根据口令查找进行中的红包,不能领取自己发的红包,同一个用户同一个红包只能领取一次,如果剩最后一个红包直接领取剩余金额,否则随机分配金额最少 0.01,使用数据库事务,增加领取用户余额,减少红包剩余金额和剩余个数,写领取记录,写钱包流水 type=redpacket_grab,剩余个数为 0 时红包状态改为已领完。返回 amount,balance。

`GET /api/version/check`:检查 App 更新,无需登录。返回 version,version_code,update_content,force_update,download_url。App 每次启动都调用,如果服务端 version_code 大于 App 本地 versionCode,App 弹出更新弹窗,force_update=1 时只能显示立即更新不能显示取消按钮。

`GET /api/home`:首页聚合数据,需要登录。返回 user.balance,user.wallet_address,withdraw_summary.available,withdraw_summary.withdrawing,withdraw_summary.success,notice_marquee。

`GET /api/notice`:公告内容,无需登录。返回 title,content,marquee,update_time。content 从 app_config 的 notice_content 读取,marquee 从 notice_marquee 读取。

`GET /api/user_notice`:用户须知,无需登录。返回 title,content,update_time。content 从 app_config 的 user_notice_content 读取。

## EasyAdmin 后台功能

请在 EasyAdmin 后台新增菜单和页面,适配当前源码已有后台风格。

用户管理:用户列表,搜索用户名/邮箱/钱包地址,展示 id、用户名、邮箱、余额、钱包地址、状态、注册时间,支持封禁/解封用户,支持后台手动增加余额和减少余额。余额操作必须填写金额和备注,金额必须大于 0,减少余额时必须校验用户余额充足,必须使用数据库事务,必须写入 `app_wallet_log` 且 type=admin,增加余额 amount 为正数,减少余额 amount 为负数。

钱包流水:只读列表,可按用户、类型、时间筛选,展示用户、流水类型、金额、变动后余额、备注、关联业务 ID、创建时间。

充值订单:只读列表,可按用户、订单号、支付方式、状态、时间筛选,展示订单号、用户、金额、支付方式 alipay/wxpay、状态、易支付平台订单号、支付时间、创建时间。

提现订单审核:提现订单列表,可按用户、订单号、状态、时间筛选,展示用户、提现账号类型、提现姓名、提现账号、收款码图片、提现金额、状态、备注、创建时间、审核时间。审核通过:只允许状态为 0 提现中的订单通过,改为 1 提现成功,记录 handle_time 和 admin_remark,不再重复扣款,因为用户发起提现时已经扣过。审核驳回:只允许状态为 0 提现中的订单驳回,改为 2 已驳回,把提现金额退回用户余额,写钱包流水 type=withdraw_refund 金额为正数,记录 handle_time 和 admin_remark,必须使用数据库事务。

提现账号管理:查看所有用户绑定的提现账号,展示用户、类型、姓名、账号、收款码图片、创建时间,可按用户、类型搜索。

红包管理:红包列表和红包领取记录列表,可查看发红包用户、口令、总金额、总个数、剩余金额、剩余个数、状态、创建时间,可查看领取用户、领取金额、领取时间。

系统配置:后台做一个配置页,可以修改邮件 apikey、邮件网站名称、SMTP 服务器、SMTP 用户名、SMTP 授权码、SMTP 端口、易支付网关、易支付 PID、易支付密钥、JWT 密钥、JWT 过期时间、App 最新版本号、App 最新 versionCode、App 更新内容、是否强制更新、App 下载地址、首页跑马灯公告、公告内容、用户须知内容,保存到 `app_config` 表。

转账记录:只读列表,查询 `app_wallet_log` 中 type 为 `transfer_in`、`transfer_out` 的记录,可按用户、钱包地址、时间筛选。

## 部署和环境适配要求

因为当前仓库已经是解压后的 EasyAdmin 源码,请直接读取当前 GitHub 仓库源码,不需要解压任何压缩包。先检查项目结构,确认 EasyAdmin 使用的 ThinkPHP 版本、后台控制器目录、路由文件位置、数据库配置方式。检查 `composer.json`,判断当前依赖是否支持 PHP 8.4。如果不支持,给出修改后的 composer.json 依赖建议和命令。提供 `.env` 或数据库配置示例:数据库类型 mysql,数据库版本 MySQL 5.7,数据库字符集 utf8mb4,数据库端口 3306。提供 Nginx 1.30 配置示例:域名 `moe.123vps.cn`,站点根目录指向项目 `public` 目录,配置 ThinkPHP 伪静态。提供 PHP 8.4 所需扩展清单:pdo_mysql,mbstring,openssl,curl,fileinfo,gd,json。提供项目目录权限建议:runtime,public/uploads,EasyAdmin 需要写入的缓存目录。不要破坏原后台登录、菜单、权限系统。新增后台功能必须适配原 EasyAdmin 风格。支付回调地址固定为 `https://moe.123vps.cn/api/pay/notify`。邮件接口、易支付接口、App API 地址都按正式域名 `https://moe.123vps.cn` 生成。

## 最终交付文档要求

完成代码开发后,请在项目根目录创建 `DEVELOPMENT_REPORT.md`,不要只在聊天窗口输出。文档内容必须包含:修改文件、新增文件、完整建表 SQL、后台菜单添加方法、API 接口列表、Nginx 配置、PHP8.4/Composer 兼容说明、部署命令、测试流程、核心接口 curl 示例、邮件接口和易支付配置说明、后台配置项说明、常见问题排查。文档要能让我按步骤完成部署、导入数据库、配置 Nginx 并测试 API。
```

再给你提供一个初始的文档，上面的是修改过的
# 给 AI 的开发 Prompt(后端 API + EasyAdmin 后台)

## 项目背景

当前 AI 后台已经绑定 GitHub 仓库,仓库里已经是解压后的 EasyAdmin 源码,不是压缩包。请直接读取当前仓库代码,先检查项目结构,再在现有 EasyAdmin 源码基础上完成后端 API 和后台功能改造。

不要重新创建新项目,不要脱离现有 EasyAdmin 目录结构。必须基于当前仓库源码开发。

服务器环境:

- 域名:`https://moe.123vps.cn`
- API 基础地址:`https://moe.123vps.cn/api`
- PHP:`8.4`
- MySQL:`5.7`
- Nginx:`1.30`
- 框架:EasyAdmin / ThinkPHP
- 数据库字符集:`utf8mb4`
- Web 根目录必须指向项目的 `public` 目录
- 所有 App 接口前缀统一为 `/api/`

请先完成以下检查:

1. 读取当前仓库源码。
2. 检查项目目录结构。
3. 检查 `composer.json`。
4. 确认 EasyAdmin 使用的 ThinkPHP 版本。
5. 确认后台控制器目录、模型目录、路由文件位置、数据库配置方式。
6. 确认 EasyAdmin 后台菜单和权限的添加方式。
7. 在现有源码上改造,不要破坏原后台登录、菜单、权限系统。
8. 如果原项目没有 API 模块,请新建 API 控制器目录。
9. 所有新增代码必须适配现有 EasyAdmin 后台风格。

请注意兼容 PHP 8.4 和 MySQL 5.7:

- PHP 代码不能使用已废弃或容易在 PHP 8.4 报错的写法。
- 如果当前依赖不兼容 PHP 8.4,请指出需要修改的 composer 依赖版本,并给出可执行命令。
- MySQL 5.7 不要使用 MySQL 8 才支持的语法。
- 不要使用窗口函数、CTE 等 MySQL 8 特性。
- 建表 SQL 使用 InnoDB + utf8mb4。
- 金额字段统一使用 `decimal(18,2)`。
- 涉及金额变动的操作必须使用数据库事务。

---

## 基础要求

所有 App 端接口统一返回 JSON:

```json
{
  "code": 0,
  "msg": "成功",
  "data": {}
}
```
## 质量优先要求

本次开发不限制代码行数、不限制回复字数、不限制实现时间,只考虑最终代码质量和功能完整度。

要求:

- 不要为了简短而省略必要代码。
- 不要只给思路,必须给可落地的完整实现。
- 不要用伪代码代替真实代码。
- 不要跳过错误处理、参数校验、事务处理、权限校验和安全处理。
- 不要因为内容较长就省略文件。
- 如果一个文件请逐个说明文件路径和完整代码。
- 所有新增和修改都必须能在当前 EasyAdmin 源码中实际运行。
- 优先保证稳定、安全、可维护,其次才考虑代码简洁。
- 如果发现现有源码结构和我的描述不一致,请以仓库实际结构为准,但必须说明原因和调整方案。

规则:

- `code=0` 表示成功。
- 非 0 表示失败。
- 除注册、登录、找回密码、发送验证码、版本检查、支付回调外,其他接口都需要 JWT 鉴权。
- JWT Token 放在请求头:

```text
Authorization: Bearer <token>
```

账号安全:

- 登录密码使用 `password_hash` / bcrypt 存储。
- 支付密码使用 `password_hash` / bcrypt 存储。
- 不允许存明文密码。
- 用户状态为封禁时禁止登录和使用需要登录的接口。

金额规则:

- 钱包余额就是 ONE 币。
- 1 ONE = 1 元。
- 充值、提现、转账、红包都使用同一余额。
- 金额展示精确到分。

---

## 一、数据库表结构

请新增以下表,表名前缀使用 `app_`。

### 1. `app_user` 用户表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | 用户 ID |
| username | varchar(50) 唯一 | 用户名,注册时自动生成 |
| email | varchar(100) 唯一 | 邮箱 |
| password | varchar(255) | 登录密码,bcrypt |
| pay_password | varchar(255) | 支付密码,bcrypt,可为空 |
| balance | decimal(18,2) 默认 0.00 | ONE 余额 |
| wallet_address | varchar(18) 唯一 | 18 位钱包地址,随机英文和数字 |
| status | tinyint 默认 1 | 1 正常,0 封禁 |
| create_time | int | 创建时间 |
| update_time | int | 更新时间 |

### 2. `app_email_code` 邮箱验证码表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | |
| email | varchar(100) | 邮箱 |
| code | varchar(10) | 6 位数字验证码 |
| scene | varchar(20) | register / login / reset / bind / paypwd |
| expire_time | int | 过期时间,5 分钟有效 |
| used | tinyint 默认 0 | 0 未使用,1 已使用 |
| create_time | int | 创建时间 |

### 3. `app_wallet_log` 钱包流水表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | |
| user_id | int | 用户 ID |
| type | varchar(30) | recharge / withdraw / withdraw_refund / transfer_in / transfer_out / redpacket_send / redpacket_grab / admin |
| amount | decimal(18,2) | 正数增加,负数减少 |
| balance_after | decimal(18,2) | 变动后余额 |
| remark | varchar(255) | 备注 |
| related_id | int 默认 0 | 关联业务 ID |
| create_time | int | 创建时间 |

### 4. `app_recharge_order` 充值订单表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | |
| order_no | varchar(32) 唯一 | 商户订单号 |
| user_id | int | 用户 ID |
| amount | decimal(18,2) | 充值金额 |
| pay_type | varchar(10) | alipay / wxpay |
| status | tinyint 默认 0 | 0 待支付,1 已支付 |
| trade_no | varchar(64) | 易支付平台订单号 |
| pay_time | int | 支付时间 |
| create_time | int | 创建时间 |
| update_time | int | 更新时间 |

### 5. `app_withdraw_account` 提现账号表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | |
| user_id | int | 用户 ID |
| type | varchar(10) | wechat / alipay |
| real_name | varchar(50) | 提现姓名 |
| account | varchar(100) | 提现账号 |
| qrcode | varchar(255) | 收款码图片地址 |
| create_time | int | 创建时间 |
| update_time | int | 更新时间 |

### 6. `app_withdraw_order` 提现订单表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | |
| order_no | varchar(32) 唯一 | 提现订单号 |
| user_id | int | 用户 ID |
| account_id | int | 提现账号 ID |
| amount | decimal(18,2) | 提现金额 |
| status | tinyint 默认 0 | 0 提现中,1 提现成功,2 已驳回 |
| admin_remark | varchar(255) | 后台审核备注 |
| create_time | int | 创建时间 |
| handle_time | int | 审核时间 |

### 7. `app_redpacket` 口令红包表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | |
| user_id | int | 发红包用户 ID |
| keyword | varchar(50) | 红包口令 |
| total_amount | decimal(18,2) | 总金额 |
| total_count | int | 总个数 |
| remain_amount | decimal(18,2) | 剩余金额 |
| remain_count | int | 剩余个数 |
| status | tinyint 默认 1 | 1 进行中,0 已领完 |
| create_time | int | 创建时间 |

### 8. `app_redpacket_record` 红包领取记录表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | |
| redpacket_id | int | 红包 ID |
| user_id | int | 领取用户 ID |
| amount | decimal(18,2) | 领取金额 |
| create_time | int | 创建时间 |

### 9. `app_config` 系统配置表

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int 主键自增 | |
| key | varchar(100) 唯一 | 配置键 |
| value | text | 配置值 |
| remark | varchar(255) | 备注 |
| create_time | int | 创建时间 |
| update_time | int | 更新时间 |

预置配置项:

```text
mail_apikey = 
mail_smtp = smtp.163.com
mail_smtp_user = cloud123pro@163.com
mail_smtp_password = JAp6zF3EWV7MNKCq
mail_smtp_port = 465
mail_webname = ONE

pay_gateway = https://pay.123vps.cn
pay_pid = e2148016ee1459d1c7d0754eb389046b
pay_key = 03a80bdecc82c4062d250cde6fb9d9978f2c49084a1f21a74c95b86c869fda94

app_version = 1.0.0
app_version_code = 100
app_update_content = 
app_force_update = 0
app_download_url = 

notice_marquee = 
```

---

## 二、第三方接口对接

### 1. 邮件发送接口

邮件 API 地址:

```text
https://api.nanyinet.com/api/gateway/mailer/api.php
```

请求方式:

```text
POST
```

Body 类型:

```text
form-data 或 application/x-www-form-urlencoded
```

请求参数:

| 参数 | 说明 |
|---|---|
| apikey | 从后台配置 `mail_apikey` 读取 |
| email | 收件人邮箱 |
| content | 邮件内容,支持 HTML |
| title | 邮件标题 |
| webname | 网站名称,从 `mail_webname` 读取 |
| smtp | `smtp.163.com` |
| smtp_user | `cloud123pro@163.com` |
| smtp_password | `JAp6zF3EWV7MNKCq` |
| smtp_port | `465` |

验证码邮件内容示例:

```html
您的验证码：123456，5分钟内有效，请勿泄露。
```

请封装 `MailService::send($email, $title, $content)`。

发送验证码逻辑:

- 验证邮箱格式。
- 同一邮箱同一场景 60 秒内不能重复发送。
- 验证码为 6 位数字。
- 验证码有效期 5 分钟。
- 验证通过后标记为已使用。

---

### 2. 易支付接口

支付网关:

```text
https://pay.123vps.cn
```

商户 PID:

```text
e2148016ee1459d1c7d0754eb389046b
```

商户密钥:

```text
03a80bdecc82c4062d250cde6fb9d9978f2c49084a1f21a74c95b86c869fda94
```

支持支付方式:

```text
alipay
wxpay
```

App 创建充值订单时传 `pay_type`:

- `alipay` 支付宝
- `wxpay` 微信支付

后端请求易支付时,把 `pay_type` 映射到易支付参数 `type`。

请使用易支付标准接口,优先使用 `mapi.php` 返回支付二维码或跳转链接给 App。

易支付请求参数至少包含:

| 参数 | 说明 |
|---|---|
| pid | 商户 PID |
| type | alipay 或 wxpay |
| out_trade_no | 商户订单号 |
| notify_url | 异步通知地址 |
| return_url | 同步跳转地址 |
| name | 商品名称,如 `购买ONE币` |
| money | 支付金额 |
| sign | 签名 |
| sign_type | MD5 |

异步通知地址:

```text
https://moe.123vps.cn/api/pay/notify
```

同步跳转地址:

```text
https://moe.123vps.cn/pay/return
```

签名规则:

1. 除 `sign`、`sign_type` 外,所有参数按参数名 ASCII 升序排序。
2. 拼接成 `a=b&c=d` 格式。
3. 末尾直接拼接商户密钥。
4. MD5 后转小写。
5. `sign_type=MD5`。

支付回调逻辑:

- 接收易支付异步通知。
- 校验签名。
- 校验订单是否存在。
- 校验金额是否一致。
- 校验 `trade_status=TRADE_SUCCESS`。
- 保证幂等,同一个订单只允许入账一次。
- 把 `app_recharge_order.status` 改为 `1`。
- 记录 `trade_no` 和 `pay_time`。
- 给用户余额增加对应金额。
- 写入 `app_wallet_log`,type=`recharge`。
- 成功后输出纯文本:

```text
success
```

失败输出:

```text
fail
```

---

## 三、App 端 API 接口

除注册、登录、找回密码、发送验证码、版本检查、支付回调外,其他接口都需要 JWT 鉴权。

### 账号模块

#### 1. 发送邮箱验证码

```text
POST /api/send_code
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| email | 是 | 邮箱 |
| scene | 是 | register / login / reset / bind / paypwd |

逻辑:

- 校验邮箱格式。
- 校验 scene 是否合法。
- 同一邮箱同一场景 60 秒内不能重复发送。
- 生成 6 位数字验证码。
- 写入 `app_email_code`。
- 调用邮件 API 发送。

---

#### 2. 注册

```text
POST /api/register
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| email | 是 | 邮箱 |
| password | 是 | 登录密码 |
| password_confirm | 是 | 重复登录密码 |
| code | 是 | 邮箱验证码 |

逻辑:

- 校验邮箱未注册。
- 校验两次密码一致。
- 校验邮箱验证码 scene=`register`。
- 自动生成唯一用户名。
- 自动生成唯一 18 位钱包地址,由大小写字母和数字组成。
- 初始余额为 `0.00`。
- 注册成功后返回 JWT token 和用户信息。

---

#### 3. 登录

```text
POST /api/login
```

支持两种登录方式。

密码登录参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| type | 是 | password |
| account | 是 | 用户名或邮箱 |
| password | 是 | 登录密码 |

验证码登录参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| type | 是 | code |
| email | 是 | 邮箱 |
| code | 是 | 邮箱验证码 |

逻辑:

- 密码登录支持用户名或邮箱。
- 验证码登录校验 scene=`login`。
- 用户被封禁时禁止登录。
- 登录成功返回 JWT token 和用户信息。

---

#### 4. 找回密码

```text
POST /api/reset_password
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| email | 是 | 邮箱 |
| code | 是 | 邮箱验证码 |
| new_password | 是 | 新密码 |
| new_password_confirm | 是 | 重复新密码 |

逻辑:

- 校验验证码 scene=`reset`。
- 校验两次新密码一致。
- 更新登录密码。

---

#### 5. 获取用户信息

```text
GET /api/user/info
```

返回:

```json
{
  "id": 1,
  "username": "user123456",
  "email": "test@qq.com",
  "balance": "0.00",
  "wallet_address": "A1b2C3d4E5f6G7h8I9",
  "has_pay_password": false,
  "status": 1
}
```

---

#### 6. 换绑邮箱

```text
POST /api/user/bind_email
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| new_email | 是 | 新邮箱 |
| code | 是 | 新邮箱验证码 |

逻辑:

- 校验验证码 scene=`bind`。
- 校验新邮箱未被其他用户绑定。
- 更新用户邮箱。

---

#### 7. 设置或修改支付密码

```text
POST /api/user/set_pay_password
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| email_code | 是 | 邮箱验证码 |
| new_pay_password | 是 | 新支付密码 |

逻辑:

- 校验当前用户邮箱的验证码 scene=`paypwd`。
- 新支付密码用 bcrypt 存储。
- 成功后返回是否已设置支付密码。

---

### 钱包和 ONE 币模块

#### 8. 创建充值订单

```text
POST /api/recharge/create
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| amount | 是 | 充值金额 |
| pay_type | 是 | alipay / wxpay |

逻辑:

- 校验金额大于 0。
- 校验 `pay_type` 只能是 `alipay` 或 `wxpay`。
- 生成唯一订单号。
- 写入 `app_recharge_order`。
- 请求易支付接口。
- 返回支付链接或二维码数据给 App。

返回示例:

```json
{
  "order_no": "RC202606080001",
  "amount": "10.00",
  "pay_type": "alipay",
  "pay_url": "https://pay.123vps.cn/xxx",
  "qrcode": "https://pay.123vps.cn/xxx.png"
}
```

---

#### 9. 易支付异步回调

```text
POST /api/pay/notify
```

说明:

- 不需要 JWT 鉴权。
- 必须校验易支付签名。
- 必须校验订单金额。
- 必须处理订单幂等。
- 支付成功后给用户增加余额。
- 成功返回纯文本 `success`。
- 失败返回纯文本 `fail`。

---

#### 10. 获取充值记录

```text
GET /api/recharge/list
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| page | 否 | 页码 |
| limit | 否 | 每页数量 |
| status | 否 | 0 待支付,1 已支付 |

返回:

```json
{
  "list": [
    {
      "order_no": "RC202606080001",
      "amount": "10.00",
      "pay_type": "alipay",
      "status": 1,
      "trade_no": "2026060812345678",
      "pay_time": 1780000000,
      "create_time": 1780000000
    }
  ],
  "total": 1
}
```

---

#### 11. 钱包流水

```text
GET /api/wallet/log
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| page | 否 | 页码 |
| limit | 否 | 每页数量 |
| direction | 否 | add / reduce |
| type | 否 | 流水类型 |

说明:

- `direction=add` 只查金额大于 0 的记录。
- `direction=reduce` 只查金额小于 0 的记录。
- 默认返回当前用户全部流水。

返回字段:

```json
{
  "list": [
    {
      "type": "recharge",
      "amount": "10.00",
      "balance_after": "10.00",
      "remark": "购买ONE币",
      "create_time": 1780000000
    }
  ],
  "total": 1
}
```

---

#### 12. 转账

```text
POST /api/transfer
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| to_wallet_address | 是 | 对方钱包地址 |
| amount | 是 | 转账金额 |
| pay_password | 是 | 支付密码 |

逻辑:

- 校验当前用户是否已设置支付密码。
- 校验支付密码。
- 校验金额大于 0。
- 校验对方钱包地址存在。
- 不允许转给自己。
- 校验余额充足。
- 使用数据库事务。
- 当前用户扣款,写 `transfer_out` 流水。
- 对方用户加款,写 `transfer_in` 流水。

返回:

```json
{
  "amount": "5.00",
  "to_wallet_address": "A1b2C3d4E5f6G7h8I9",
  "balance": "95.00"
}
```

---

#### 13. 转账记录

```text
GET /api/transfer/list
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| page | 否 | 页码 |
| limit | 否 | 每页数量 |

逻辑:

- 查询当前用户 `app_wallet_log` 中 type 为 `transfer_in` 和 `transfer_out` 的记录。

---

### 提现模块

#### 14. 提现账号列表

```text
GET /api/withdraw/account/list
```

返回:

```json
{
  "list": [
    {
      "id": 1,
      "type": "alipay",
      "real_name": "张三",
      "account": "test@alipay.com",
      "qrcode": "https://moe.123vps.cn/uploads/qrcode/xxx.png",
      "create_time": 1780000000
    }
  ]
}
```

---

#### 15. 新增提现账号

```text
POST /api/withdraw/account/add
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| type | 是 | wechat / alipay |
| real_name | 是 | 提现姓名 |
| account | 是 | 提现账号 |
| qrcode | 是 | 收款码图片地址 |

逻辑:

- `type` 只能是 `wechat` 或 `alipay`。
- `real_name` 不能为空。
- `account` 不能为空。
- `qrcode` 不能为空。
- 绑定到当前登录用户。

---

#### 16. 上传图片

```text
POST /api/upload
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| file | 是 | 图片文件 |

要求:

- 需要 JWT 鉴权。
- 仅允许 jpg、jpeg、png、webp。
- 限制文件大小,建议 5MB 以内。
- 上传到 `public/uploads/app/`。
- 返回完整 URL。

返回:

```json
{
  "url": "https://moe.123vps.cn/uploads/app/xxx.png"
}
```

---

#### 17. 发起提现

```text
POST /api/withdraw/create
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| account_id | 是 | 提现账号 ID |
| amount | 是 | 提现金额 |
| pay_password | 是 | 支付密码 |

逻辑:

- 校验提现账号属于当前用户。
- 校验当前用户是否设置支付密码。
- 校验支付密码。
- 校验金额大于 0。
- 校验余额充足。
- 使用数据库事务。
- 先扣除用户余额。
- 写钱包流水,type=`withdraw`,金额为负数。
- 创建提现订单,status=`0` 提现中。
- 后台管理员后续人工审核打款。

---

#### 18. 提现记录

```text
GET /api/withdraw/list
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| page | 否 | 页码 |
| limit | 否 | 每页数量 |
| status | 否 | 0 提现中,1 提现成功,2 已驳回 |

返回:

```json
{
  "summary": {
    "available": "90.00",
    "withdrawing": "10.00",
    "success": "100.00"
  },
  "list": [
    {
      "order_no": "WD202606080001",
      "amount": "10.00",
      "status": 0,
      "account": {
        "type": "alipay",
        "real_name": "张三",
        "account": "test@alipay.com",
        "qrcode": "https://moe.123vps.cn/uploads/app/xxx.png"
      },
      "admin_remark": "",
      "create_time": 1780000000,
      "handle_time": 0
    }
  ],
  "total": 1
}
```

说明:

- `available` 为当前钱包余额。
- `withdrawing` 为提现中订单金额合计。
- `success` 为提现成功订单金额合计。

---

### 红包模块

#### 19. 发口令红包

```text
POST /api/redpacket/send
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| keyword | 是 | 红包口令 |
| total_amount | 是 | 红包总金额 |
| total_count | 是 | 红包个数 |
| pay_password | 是 | 支付密码 |

逻辑:

- 校验已设置支付密码。
- 校验支付密码。
- 校验口令不能为空。
- 校验总金额大于 0。
- 校验个数大于 0。
- 校验总金额不能小于红包个数 * 0.01。
- 校验余额充足。
- 使用事务扣除发红包用户余额。
- 创建红包记录。
- 写钱包流水,type=`redpacket_send`,金额为负数。

---

#### 20. 领取口令红包

```text
POST /api/redpacket/grab
```

参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| keyword | 是 | 红包口令 |

逻辑:

- 根据口令查找进行中的红包。
- 不能领取自己发的红包。
- 同一个用户同一个红包只能领取一次。
- 如果剩最后一个红包,直接领取剩余金额。
- 否则随机分配金额,最少 0.01。
- 使用数据库事务。
- 增加领取用户余额。
- 减少红包剩余金额和剩余个数。
- 写领取记录。
- 写钱包流水,type=`redpacket_grab`。
- 剩余个数为 0 时把红包状态改为已领完。

返回:

```json
{
  "amount": "0.66",
  "balance": "20.66"
}
```

---

### 版本和公告模块

#### 21. 检查 App 更新

```text
GET /api/version/check
```

不需要登录。

返回:

```json
{
  "version": "1.0.0",
  "version_code": 100,
  "update_content": "修复已知问题",
  "force_update": 0,
  "download_url": "https://moe.123vps.cn/download/app.apk"
}
```

说明:

- App 每次启动都调用。
- 如果服务端 `version_code` 大于 App 本地 versionCode,App 弹出更新弹窗。
- `force_update=1` 时,App 只能显示立即更新,不能显示取消按钮。

---

#### 22. 首页聚合数据

```text
GET /api/home
```

需要登录。

返回:

```json
{
  "user": {
    "balance": "90.00",
    "wallet_address": "A1b2C3d4E5f6G7h8I9"
  },
  "withdraw_summary": {
    "available": "90.00",
    "withdrawing": "10.00",
    "success": "100.00"
  },
  "notice_marquee": "欢迎使用 ONE"
}
```

---

## 四、EasyAdmin 后台功能

请在 EasyAdmin 后台新增以下菜单和页面,适配当前源码已有后台风格。

### 1. 用户管理

功能:

- 用户列表。
- 搜索:用户名、邮箱、钱包地址。
- 展示:id、用户名、邮箱、余额、钱包地址、状态、注册时间。
- 封禁/解封用户。
- 后台手动增加余额。
- 后台手动减少余额。

余额操作要求:

- 必须填写金额和备注。
- 金额必须大于 0。
- 减少余额时必须校验用户余额充足。
- 必须使用数据库事务。
- 必须写入 `app_wallet_log`,type=`admin`。
- 增加余额 amount 为正数。
- 减少余额 amount 为负数。

---

### 2. 钱包流水

功能:

- 只读列表。
- 可按用户、类型、时间筛选。
- 展示用户、流水类型、金额、变动后余额、备注、关联业务 ID、创建时间。

---

### 3. 充值订单

功能:

- 只读列表。
- 可按用户、订单号、支付方式、状态、时间筛选。
- 展示订单号、用户、金额、支付方式(alipay/wxpay)、状态、易支付平台订单号、支付时间、创建时间。

---

### 4. 提现订单审核

功能:

- 提现订单列表。
- 可按用户、订单号、状态、时间筛选。
- 展示用户、提现账号类型、提现姓名、提现账号、收款码图片、提现金额、状态、备注、创建时间、审核时间。

审核操作:

1. 通过:
   - 只允许状态为 `0 提现中` 的订单通过。
   - 改为 `1 提现成功`。
   - 记录 `handle_time` 和 `admin_remark`。
   - 不再重复扣款,因为用户发起提现时已经扣过。

2. 驳回:
   - 只允许状态为 `0 提现中` 的订单驳回。
   - 改为 `2 已驳回`。
   - 把提现金额退回用户余额。
   - 写钱包流水,type=`withdraw_refund`,金额为正数。
   - 记录 `handle_time` 和 `admin_remark`。
   - 必须使用数据库事务。

---

### 5. 提现账号管理

功能:

- 查看所有用户绑定的提现账号。
- 展示用户、类型、姓名、账号、收款码图片、创建时间。
- 可按用户、类型搜索。

---

### 6. 红包管理

功能:

- 红包列表。
- 红包领取记录列表。
- 可查看发红包用户、口令、总金额、总个数、剩余金额、剩余个数、状态、创建时间。
- 可查看领取用户、领取金额、领取时间。

---

### 7. 系统配置

功能:

后台做一个配置页,可以修改以下配置:

- 邮件 apikey
- 邮件网站名称
- SMTP 服务器
- SMTP 用户名
- SMTP 授权码
- SMTP 端口
- 易支付网关
- 易支付 PID
- 易支付密钥
- App 最新版本号
- App 最新 versionCode
- App 更新内容
- 是否强制更新
- App 下载地址
- 首页跑马灯公告

保存到 `app_config` 表。

---

### 8. 转账记录

功能:

- 只读列表。
- 查询 `app_wallet_log` 中 type 为 `transfer_in`、`transfer_out` 的记录。
- 可按用户、钱包地址、时间筛选。

---

## 五、部署和环境适配要求

因为当前仓库已经是解压后的 EasyAdmin 源码,请按下面流程处理:

1. 直接读取当前 GitHub 仓库源码。
2. 不需要解压任何压缩包。
3. 先检查项目结构,确认 EasyAdmin 使用的 ThinkPHP 版本、后台控制器目录、路由文件位置、数据库配置方式。
4. 检查 `composer.json`,判断当前依赖是否支持 PHP 8.4。如果不支持,给出修改后的 composer.json 依赖建议和命令。
5. 提供 `.env` 或数据库配置示例:
   - 数据库类型:mysql
   - 数据库版本:MySQL 5.7
   - 数据库字符集:utf8mb4
   - 数据库端口:3306
6. 提供 Nginx 1.30 配置示例:
   - 域名:`moe.123vps.cn`
   - 站点根目录指向项目 `public` 目录
   - 配置 ThinkPHP 伪静态
7. 提供 PHP 8.4 所需扩展清单:
   - pdo_mysql
   - mbstring
   - openssl
   - curl
   - fileinfo
   - gd
   - json
8. 提供项目目录权限建议:
   - `runtime`
   - `public/uploads`
   - EasyAdmin 需要写入的缓存目录
9. 不要破坏原后台登录、菜单、权限系统。
10. 新增后台功能必须适配原 EasyAdmin 风格。
11. 支付回调地址固定为:

```text
https://moe.123vps.cn/api/pay/notify
```

12. 邮件接口、易支付接口、App API 地址都按正式域名 `https://moe.123vps.cn` 生成。

---

## 六、最终请输出

请完成代码开发后,在项目根目录新增一份交付文档:

1. 修改了哪些文件。
2. 新增了哪些文件。
3. 完整建表 SQL。
4. 后台菜单添加方法。
5. API 接口列表。
6. Nginx 配置。
7. Composer / PHP 8.4 兼容处理说明。
8. 部署命令。
9. 测试流程。
10. 
每个核心接口的 curl 测试示例。
11. 第三方接口配置说明,包括邮件接口和易支付接口。
12. 后台系统配置项说明。
13. 常见问题排查。
要求:
• 文档必须写入项目根目录。
• 文件名必须是  DEVELOPMENT_REPORT.md 。
• 不要只在聊天窗口里输出,必须创建这个 Markdown 文件。
• 内容要能让我按文档一步步部署、导入数据库、配置 Nginx、测试 API。