# iOS编程标准化之MVC组件化实践

#### **痛点**

- 维护和修改别人代码比较痛苦，因为团队每个人设计不一致

- bugs较多，容易出错

- 开发效率太慢



#### **设计目标**

- 高效，高质量开发
- 尽量将所有设计标准化，形成文档，整个团队设计一致
- 统一事件发送及处理接口，方便查找**查找事件处理**，方便后期维护。
- 简化事件传递
- 减轻Controller负担




#### **标准化方向**

- MVC
  - 目录结构
  - 命名
  - 架构
- 



#### **目录划分及文件命名规范**

  ```
  User
  ├── Security
  │   ├── Gesture
  │   │   ├── View
  │   │   │   ├── PARSUserSecurityGestureView.swift
  │   │   │   └── PARSUserSecurityGestureHeaderView.swift
  │   │   ├── Controller
  │   │   │   └── PARSUserSecurityGestureController.swift
  │   │   └── Model
  │   │   │   ├── PARSUserSecurityGestureProvider.swift
  │   │   │   ├── PARSUserSecurityGestureRequest.swift
  │   │   │   └── PARSUserSecurityGestureManager.swift
  │   ├── Password
  │   │   ├── View
  │   │   ├── Controller
  │   │   └── Model
  │   │       ├── Manager
  │   │       │   └── PARSUserSecurityPasswordManager.swift
  │   │       ├── Request
  │   │       │   └── PARSUserSecurityPasswordRequest.swift
  │   │       └── Provider
  │   │           └── PARSUserSecurityPasswordProvider.swift
  │   ├── ...
  ├── Info
  │   ├── ...
  ```

  #### 

  - Controller：PARSUserSecurityGestureController

  - Controller.View: PARSUserSecurityGestureView

  - Model:PARSUserSecurityGestureManager

    Model: PARSUserSecurityGestureManager，负责封装并组装其它的Model类，比如PARSUserSecurityGestureRequest和PARSUserSecurityGestureProvider等

    ​            存储： PARSUserSecurityGestureProvider

    ​            网络请求：PARSUserSecurityGestureRequest

    **说明**:这样一眼很容易就找到Controller、View及Model的对应关系

    ​

#### **减轻Controller负担**

- 单一职责、迪米特原则：只负责View和Model之间的通讯，也即承担起中介者模式中中介作用
- 明确View和Model的职责：View负责和用户交互，必须将所有细节封装起来；model负责数据处理，也必须将所有细节封装起来。比如PARSUserSecurityGestureView将所有的view的细节都封装起来了；而PARSUserSecurityGestureManager将所有的model细节 都封装起来；PARSUserSecurityGestureController不需要关心。


   问题来了：由于View通常是由这么多子View组合起来的，子View和Controller的通讯比较麻烦，如何解决？请查看**简化事件传递及处理**及**减少View更新时层次传递**



#### **View规范**

- 每个模块的常量叫PARSUserSecurityGestureConst

#### **减少View更新时层次传递**

实现步骤：

1. 为每个View的更新方法申明一个tag来标记此方法

   ```
   // 申明tag
   enum PARSUserInfoTag{
     PARSUserInfoTagName = 1 // 用户名
   }
   ```

2. View初始化时调用`parsAddSelector:forTag:`方法将tag和方法绑定。

   **说明：**系统会自动取到二级模块名字区分不同tag。 如下示例，用户信息模块的View名字叫`PARSUserInfoView.m`，调用`parsAddSelector:forTag:`时会取出`UserInfo`作为分类模块名，将`PARSUserInfoTagName`归类到`UserInfo`模块，避免tag重复，`UserInfo`模块不允许tag值重复；

   ```
   // PARSUserInfoView.m
   - (void)awakeFromNib{
       // 添加tag对应的方法给Controller调用
       [self parsAddSelector:@selector(updateName:) forTag:PARSUserInfoTagName];
   }
   // 更新用户名字函数
   - (void)setButtonTitleWithParam:(id<PARSViewParamProtocol> _Nonnull)param{
       self.labelName.text = param.data;
   }
   ```

3. Controller调用`parsUpdateViewWithParam:`

   **说明：**因为参数中带tag，就可以通过tag找到相应的方法；参数传递只要遵守PARSViewParamProtocol协议即可，也即可以自定义参数。

   ```
   //  PARSUserInfoViewController.m
   - (void)viewDidLoad{
     [super viewDidLoad];
     // 更新用户名
     [self parsUpdateViewWithParam:[PARSViewParam<NSString*> paramWithData:@"张三" tag:tag]];
   }
   ```



#### **简化事件传递及统一事件处理入口**

实现步骤：

1. 为每个事件申明一个ActionTag

   **说明：**ActionTag的作用域为Controller，即不同的Controller事件的值是可以重复的。

   ```
   // PARSUserActionTag.h
   enum PARSUserActionTag{
      PARSUserActionTagChangeAvatar = 1, // 修改头像
   };
   ```

2. View初始化时为事件添加响应

   ```
   // PARSUserInfoView.m
   - (void)awakeFromNib{
       // 为修改头像按钮添加事件响应
       [self.button parsAddTouchUpInsideActionWithTag:PARSUserActionTagChangeAvatar];
   }
   ```

3. Controller实现`PARSViewActionsHandlerViewController`协议

   **说明：** Controller实现`viewEventsHandlerTable`方法，此方法返回字典，key为第1步申明的tag，value可以是selector或者block

   ```
   // PARSUserInfoViewController.m实现PARSViewActionsHandlerViewController协议
   - (NSDictionary<NSNumber*, id>*) viewEventsHandlerTable{
       return @{
                @(PARSUserActionTagChangeAvatar):PARSSelectorToString(changeAvatar:),
               };
   }

   - (void)changeAvatar:(id <PARSViewActionParam>)param
   {
      // 处理 PARSUserActionTagChangeAvatar 事件
   }
   ```

   示例：如上所示，只要在PARSUserActionTag.h头文件里，就可以查找到所有事件处理，增加代码可读性。

#### **UITableViewController、UICollectionViewController、或者带UITableView、UICollectionView控制器编码规范**
实现步骤：

1. 实现自定义的PARSCustomTableViewCell，并实现PARSCellUpdateDataProtocol更新协议

   ```
   extension PARSImageRightCollectionViewCell:PARSCellUpdateDataProtocol{
       public func update(data: Any, indexPath:IndexPath) {
           if let zone = data as? PARSLifeHomeModuleZoneItem{
               self.textLabel?.text = zone.title
               self.detailTextLabel?.text = zone.subTitle
               if nil != zone.imageUrl{
                   self.imageView?.sd_setImage(with: URL(string:zone.imageUrl!))
               }
           }
       }
   }
   ```

2. Controller有TableView则实现PARSTableViewDataSourceDelegateProtocol

   协议;CollectionView则实现PARSCollectionViewDataSourceDelegateProtocol协议;Header或者Footer实现```PARSDataSourceUpdateHeaderFooterProtocol```

   **说明:**该协议用于取数据时，根据数据类型映射对应的Cell

   ```
   // 数据映射到Cells
   - (NSDictionary<NSString*, Class>*)dataMapCells{
       return @{PARSSampleData.parsIdentifier:PARSImageRightCollectionViewCell.class};
   }

   // 数据映射到HeaderView
   - (NSDictionary<NSString*, Class>*)dataMapHeaders{
       return @{PARSDataHeader.parsIdentifier:PARSHeaderCollectionViewCell.class};
   }

   // 数据映射到footerView
   - (NSDictionary<NSString*, Class>*)dataMapFooters{
       return @{PARSDataFooter.parsIdentifier:PARSHeaderCollectionViewCell.class};
   }
   ```

3. 加载的数据格式化成需要显示的格式，并使用其初始化PARSSectionsData

   ```
   // 如果cell中又嵌套了tableview或者cellectionview，则格式化成数组
       PARSSampleData* sample = [[PARSSampleData alloc] init];
       NSArray* items = @[sample, [sample copy], [sample copy], [sample copy], [sample copy], [sample copy], [sample copy]];
       PARSOCSection* section0 = [PARSOCSection sectionWithItems:items
                                                          header:@"我是头"
                                                          footer:@"我是尾"];
       NSArray* items1 = [items arrayByAddingObjectsFromArray:items];
       PARSOCSection* section1 = [PARSOCSection sectionWithItems:@[items1]
                                                          footer:@"我是尾1"];
   PARSOCSectionsData *data = [PARSOCSectionsData sectionsDataWithSections: section0, section1 , nil];
   ```

4. 加载数据

   ```
       [self.collectionView parsSetDataSourceWithData:[PARSOCSectionsData
                                                  sectionsDataWithSections:section0, section1, nil]];
   ```

   这时候会取collectionView的controller做为遵守PARSCollectionViewDataSourceDelegateProtocol协议的类

5. TableView和CollectionView需要在Controller里自已实现高度等其它delegate

   ```
   // tableview
   - (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{
       return 44.0;
   }
   // collectionview
   - (CGSize)collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout*)collectionViewLayout sizeForItemAtIndexPath:(NSIndexPath *)indexPath {
       return CGSizeMake(80, 80);
    }
   ```

   ​
#### **埋点规范**

解决模块内，B和C类都继承于A类，但B和C类的埋点不一样问题。

1. 实现PARSDataAnalyticsProtocol方法sendDataAnalyticsWithTag:(int)tag params:(NSDictionary*)dict
2. 在Controller的头文件中申明埋点的标签
3. B类和C类实现各自的sendDataAnalyticsWithTag:(int)tag params:(NSDictionary*)dict方法

#### **业务流程规范**

每个子模块都有一个Flow类，比如用户手势密码模块会PARSUserGestureFlow类，此类中的每个函数包含一个完整的业务流程，也即在一个函数中能看到完整的业务流程，而不是流程分散在各个模块中，跳来跳去。

```
RACSignal* PGShareKitFlowShare(PGShareKitBLLGetSharInfo getParamBlock)
{
   return [[[/* 加载配置数据 */
       [PGShareKitLoadConfigSignal()
       /* 过滤没安装的 */
        map:^id(NSArray<id<PGSKServiceInfo>> *services) {
            return [[services.rac_sequence filter:^BOOL(id value) {
                return PGShareKitServiceCanShare(value);
            }]
                    array];
    }]/* 让用户选择分享平台 */
     flattenMap:^RACStream *(NSArray<id<PGSKServiceInfo>> *services) {
         return PGShareKitCreateShareSelectorSignal(services);
    }]
      /* 获取分享的参数数据，必须时进行转换，比如twitter和微博 */
     flattenMap:^RACStream *(NSObject<PGSKServiceInfo>* serviceInfo) {
         return PGShareKitCreateShareDataAndTransformIfNeedSignal(getParamBlock, serviceInfo);
    }]
     /* 分享到社交平台 */
     flattenMap:^RACStream *(RACTuple* tuple) {
         return PGShareKitCreateShareSignal(tuple.first, tuple.second, (PGSKServiceSupportedDataType)[tuple.third integerValue]);
     }];
}
```

### **未来展望**

- 引入redux框架
- model接口标准化
- 去除ViewActionsHandlerViewController事件处理中心，所有消息直接从View发往Store进行处理，代码将进一步简化。
- 上和下拉刷新