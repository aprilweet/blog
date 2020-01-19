# 效果展示

### 迷迷蒙蒙明明灭灭
> 记录生活，学习技术

![欢迎扫一扫](https://user-images.githubusercontent.com/10728431/53088139-dd191400-3543-11e9-99b7-a5dfb4dceeff.jpg)

# 部署说明
blog v2.0.0 采用微信小程序云开发架构，无需自建服务器。

1. 初始化云存储
   1. 新建文件夹：mmAvatar,mmImage
   2. 配置权限：*所有用户可读，仅创建者及管理员可写*
2. 初始化云数据库
   1. 新建集合：mmUser,mmArticle,mmComment,mmLike,mmMe,mmAnalytics,mmDouban
   2. 配置权限：目前只有云函数访问云数据库，小程序不直接访问，所以设置为最小权限*仅管理员可读写*即可，或者保持默认
   3. 配置索引：目前数据量较小，不必配置索引，待数据量增长后再考虑
   4. 初始化云数据库集合mmMe，可将图片传入云存储mmImage中
3. 部署云函数
   1. 上传tcb目录中的云函数：mmApi,mmDouban。选择*云端安装依赖*上传
   2. 配置云函数：环境变量，超时时间，触发方式等
      * mmApi
        * 超时时间：3秒
        * 环境变量：readOnly=false
      * mmDouban
        * 超时时间：20秒
        * 环境变量：account=*你的豆瓣域名*
        * 上传触发器：*EveryHour*，每小时整点运行
      * mmSearchSubmit
        * 超时时间：20秒
        * 上传触发器：*EveryDay*，每天零点运行
4. 测试发布小程序
   1. 修改src/config.js中的tcbEnv为云开发环境ID
   2. 在 *微信公众平台-小程序-开发-开发设置-服务器域名-downloadFile合法域名* 中设置*https://wx.qlogo.cn*
   3. 如果有需要，可以在 *微信公众平台-小程序-统计-自定义分析* 中新建自定义事件，事件字段请参考小程序代码
   4. 编译小程序
   5. 发布小程序

# 升级方案
此方案适用从blog v1升级到blog v2。在完成上述部署后，再进行数据迁移。在数据迁移前，可以考虑将之前的小程序停止服务，防止新数据产生

1. 数据库迁移
   1. 利用server的migrate.py工具将已有数据导出成各个文件
   2. 将文件分别导入对应的云数据库集合
2. 图片迁移
   * 利用云函数migrate，将用户头像和文章图片下载并上传到云存储，并更新到云数据库。注意不要上传云端运行，而是在本地调试模式中执行：
     * 修改index.js中的cloud.init()为指定环境
     * 由于可能需要较长时间，云端可能超时。本地调试，自定义超时0（不超时）
     * 并发连接数可能超出云端平台限制，本地调试则没有限制
     * cloud.uploadFile和http.IncomingMessage流式配合，在node.js 8(云端默认环境)中不能正常运行，本地调试可以指定node.js 11（目前正常）
     * 如果任何原因失败，重试直到输出 *User update successfully* 和 *Article update successfully*
3. 如果之前自定义分析中使用了整型的字段，而新版小程序中部分字段（如用户ID，文章ID）为字符串，需要适配修改并发布
4. 移除不必要的小程序配置的可信域名

# 版本说明
* 目前blog包含子模块miniapp和server，三者之间的版本号互相独立
* 微信公众平台小程序发布版本号对应miniapp的tag
* blog的tag是整个系统的版本

# 发布文章
文章信息存储在云数据库的mmArticle集合中，发布一篇文章，也即在其中添加一条记录，每条记录有以下字段，其中部分为可选（没有或者设置为null）：
* abstract：string，可选。文章摘要，显示在文章列表页面，markdown文本
* abstractInFile：boolean，可选。true表明abstract字段为markdown文本的云存储fileID，markdown文本会从对应云存储文件读取
* author：string，可选。文章原作者，展示时会覆盖userID指定的作者
* body：string，可选。文章正文，显示在文章页面，markdown文本
* bodyInFile：boolean，可选。与abstractInFile相似
* classification：string，可选。文章分类
* createTime：date，必选。发布时间，影响显示顺序
* deleted：boolean，必选。文章是否已经删除，删除后不会显示
* sourceLink：string，可选。转载文章的链接
* tags：array of string，可选。文章的标签列表
* title：string，必选。文章标题
* updateTime：date，可选。文章最近修改时间，目前不影响展示顺序
* userID：string，可选。文章发布者ID，必须在mmUser中存在（也即使用过本小程序），如果author字段为空，则展示此user的昵称为文章作者
* visible：boolean，必选。文章是否可见。false表明此文章只对管理员可见

