# QQMusicBySwift

## 在此只模仿了歌手列表界面与播放界面
  
## 1.搭建项目的结构
* 1.划分项目功能模块, 创建文件夹结构
    * 1. 项目总共划分为两个大模块
     * 音乐列表
       * 主要负责展示音乐列表, 当点击某一个音乐时, 就播放对应音乐, 停止其他音乐播放
     * 音乐详情
      * 主要负责展示音乐详情, 包含音乐名称, 歌手, 专辑图片, 歌词, 进度, 以及控制逻辑
    * 2.文件夹结构创建如下
  
  ![示例](http://ww4.sinaimg.cn/large/006nRIk4jw1f3yoxhh7b0j30h80bcwfe.jpg)
  
* 2.拖入必要的资源文件和工具类,以及第三方框架

  ```
      *  1. APP图标, 启动图片等等
          App Icons on iPad and iPhone
      *  2. 拖入音乐文件, 专辑图片文件, 歌词文件等必备资源文件
      *  3. 一些第三方框架可以等使用时再拖入
  ```
  
* 3.根据界面跳转逻辑, 搭建storyboard, 并创建好对应的控制器 
     * 1. 导航控制器为初始控制器, 其根控制器为 UITableViewController(QQ音乐列表控制器)
     * 2. 当点击QQ音乐列表控制器某一行时, 跳转到详情控制器 UIViewController;

##2. 实现QQ音乐列表功能
* 1.界面基本设置
    * 1.背景图片
    * 2.隐藏导航栏
    * 3.状态栏设置为白色

* 2.加载QQ列表数据

  ```
  不能将获取数据的实现逻辑在控制器中写, 这样不利于日后的维护和重用, 也不利于后期扩展
  ```
   * 1. 创建数据模型
      * 根据音乐列表的plist文件内容, 创建对应的音乐数据模型 JDYMusicModel 
   * 2.创建数据操作工具类
      * > 主要负责数据的获取, 和以后数据的操作;
      * > 此处提供, 供外界调用的获取数据的接口
      * > 请使用`闭包`将数据传递出去, 不要直接返回一个数组(因为后期如果改成从网络获取列表, 就懵了, 网络获取数据是异步的)
      * > 代码如下:

      ```
       /** 获取所有音乐列表接口 */
       class func getMusicList(result: (musicMs: [QQMusicModel])->()){
        
    
        //解析歌曲信息
        //1.获取文件路径,加载本地的plist文件
        guard let path = NSBundle.mainBundle().pathForResource("Musics.plist", ofType: nil) else {
            
            result(musicMs: [QQMusicModel]())
        
            return
        }
        
        //2.读取文件内容
        guard let dictArray = NSArray(contentsOfFile: path)else{
            
            result(musicMs: [QQMusicModel]())
            
            return
        }
        
        //3.解析:把字典转换成歌曲模型
        var models = [QQMusicModel]()
        
        for dict in dictArray {
            
            let model = QQMusicModel(dict: dict as! [String : AnyObject])
            
            models.append(model)
        }
        //4.返回结果粗去
        result(musicMs: models)
    }
      ```
      
   * 3.在表格控制器内, 调用数据操作类提供的接口, 加载数据并展示 

* 3.音乐列表界面展示
  * 自定义cell, 以便于后期扩充
* 4.预留好对接接口
  * **知道到时候在哪调用真正的外界播放接口, 停止接口. 为了统一管理**
  * 1. 播放接口
  * 2. 停止播放接口
* 5.实现音乐播放功能
  *  **千万不要把播放的业务实现逻辑直接写在控制器里面, 大哥, 会死人; 应当抽取一个工具类**
  *  **针对于音乐播放功能, 建议分为两层; 最底层负责单个音乐的播放,暂停,停止等操作; 上层则负责播放的业务逻辑, 比如上一首, 下一首, 随机播放, 顺序播放等等; 易于维护,重用, 扩展!**

 #### 1.封装单个音乐文件操作的工具类 (JDYAudioTool)
 1.**接口**

     ```
      var player: AVAudioPlayer?
    /*根据音频名称播放音频 */
    func playMusic(name: String)
    {
        //具体的播放实现
        //1.创建播放器
        guard let url = NSBundle.mainBundle().URLForResource(name, withExtension: nil) else {
            
            return
    
        }
        if url == player?.url
        {
            //播放的是同一首歌曲
            player?.play()
            return
        }
        
        do
        {
            player = try AVAudioPlayer(contentsOfURL: url)
        }catch
        {
            print(error)
            return
        }
        
        //2.准备播放
        player?.prepareToPlay()
        
        //3.开始播放
        player?.play()
    }
    
 2.接口
    /*暂停播放*/
    func pauseCurrentMusic() -> ()
    {
        player?.pause()
    }
 3.接口   
    /*恢复播放*/
    func resumeCurrentMusic() -> ()
    {
        player?.play()
    }
      ```
      
#### 2.封装多个音乐文件操作的工具类 (QQMusicOperationTool)
* 接口1

```
  //将此工具类设计成为一个单例; 因为会有很多界面使用; 而且多个界面操作的数据一致
    static let shareInstance = QQMusicOperationTool()
    
    /*音乐播放列表*/
    var musicMList: [QQMusicModel]?
    
    // 记录当前正在播放的索引
    var index = 0 {
        didSet {
            if musicMList != nil
            {
                
                if index < 0
                {
                    index = musicMList!.count - 1
                }
                if index > musicMList!.count - 1 {
                    index = 0
                }
                
            }
        }
    }
 //创建QQMusicTool工具类音乐播放列表
    let tool = QQMusicTool()
    
    /* 根据音频名称播放音频 */
    func playMusic(musicM: QQMusicModel) -> (){
        
        let fileName = musicM.filename ?? ""
        
        tool.playMusic(fileName)
        
        
        /*判断是否是播放列表*/
        if musicMList == nil
        {
            print("没有播放列表,只能播放一首歌曲")
            return
        }
        
        index = (musicMList?.indexOf(musicM))!
    }
    接口2
    /*开始播放音乐*/
    func playCurrentMusic() -> ()
    {
        tool.resumeCurrentMusic()
    }
    
    接口3
    /*暂停播放*/
    func pauseCurrentMusic() -> ()
    {
        tool.pauseCurrentMusic()
    }
```

####5. 在预留接口中, 调用工具类的对应接口, 然后测试; QQ音乐列表功能结束;

##3. QQ音乐详情界面实现
#####1. QQ音乐详情界面搭建
* 1. 分析界面结构, 选择合适控件搭建界面;
* 2. 注意将同一组子控件使用一个父控件进行包装, 方便添加约束布局;
* 3. 稍微不好构思的地方在于歌词界面和专辑界面的切换, 需要借助UIScrollView;
* 4. 关联属性和方法到对应的详情控制器, 方便后续的动画和赋值操作
 
#####2. 扩展音乐播放工具类接口, 实现播放业务逻辑, 并展示音乐详情
* 1.扩展多个音乐操作的工具类 (QQMusicOperationTool) 上一首, 下一首 等接口
   * 接口1
   
```
 /*下一首*/
    func nextMusic() -> ()
    {
        index += 1
        
        if let tempList = musicMList
        {
            //取出模型
            let musicM = tempList[index]
            //根据模型.播放音乐
            playMusic(musicM)
        }
    }
    
    /*上一首*/
    func preMusic() -> ()
    {
        index -= 1
        
        if let temoList = musicMList
        {
            //取出模型
            let musicM = temoList[index]
            //根据模型,播放音乐
            playMusic(musicM)
        }
    }

```

* 2.在控制器对应的关联方法中, 调用不同的播放接口, 进行测试
* 3.将需要展示的数据按 "刷新频率" 进行分类, 分别提供 "单次刷新" 和 "实时刷新" 方法
   * 需要根据不同的数据刷新频率, 采用不同的刷新策略
   * 例如: 如果实时刷新, 就可以使用NSTimer, 使用定时任务不断刷新, 展示最新数据; 

* 4.汇总所有需要刷新的字段, 根据字段, 创建歌曲播放信息数据模型; 此数据模型由 多个音乐操作的工具类 (QQMusicOperationTool) 统一提供
    * > 不要非常零散的单独获取, 到处拼凑;
    * > 之所以由 QQMusicOperationTool 统一提供歌曲播放信息数据模型,主要原因
       * 第一是因为此功能, 应划分到此类的业务逻辑中; 
       * 第二,只有这个类, 最了解当前音乐的播放信息;
* 5.直接从控制器预留的 "单次刷新" 和 "实时刷新" 刷新方法中, 从 QQMusicOperationTool 中获取最新的音乐播放数据;


#####3. 实时更新歌词, 并实现进度展示
* 1. 创建歌词数据模型 (QQLrcsMode)
* 2. 创建歌词解析工具类 (QQLrcDataTool), 负责解析不同歌曲对应的歌词文件
   * 接口如下:


