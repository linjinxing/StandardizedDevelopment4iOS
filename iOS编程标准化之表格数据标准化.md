# iOS编程标准化之表格数据标准化实践

#### **痛点**

- 维护和修改别人代码比较痛苦，因为团队每个人设计不一致

- bugs较多，容易出错

- 开发效率太慢



#### **设计目标**

- 高效，高质量开发
- 尽量将所有设计标准化，形成文档，整个团队设计一致
- 统一数据源
- 统一Cell开发规范
- 减轻Controller负担




#### **标准化点**

- DataSource
- Cell



#### **DataSource标准化**

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