#【iOS开发】python解析sqlite数据库

文｜远风foreverwind

##1. 引言

在iOS开发调试过程中，打印日志是最常使用的手段，但是打印日志有时候无法提供详细的信息用于问题的分析，例如一般的消息数据由于数据量较大，打印日志时只会选择将一些关键信息打印出来，信息不够完整。因此，如果能将存入sqlite数据库中的数据导出，对于问题的分析定位解决将会十分有帮助。

##2. python解析sqlite基本数据类型<sup>[[1]](https://docs.python.org/2/library/sqlite3.html)</sup>

2.1 数据库基本操作

- 数据库连接

```
import sqlite3
conn = sqlite3.connect('example.db')  
```

- 数据库连接对象基本操作

```
cur = conn.cursor()    //创建cursor
conn.commit()          //事务提交
conn.rollback()        //事务回滚
conn.close()           //关闭一个数据库链接
```

- 数据库cursor基本操作

```
cur.execute()    //执行sql语句
cur.fetchone()   //从结果中取出一条记录
cur.fetchall()   //从结果中取出所有记录
cur.close()      //关闭cursor
```

2.2 获取数据库中的表

sqlite数据库中都包含一个特殊的表sqlite_master，其表结构为：

```
CREATE TABLE sqlite_master (
type TEXT,
name TEXT,
tbl_name TEXT,
rootpage INTEGER,
sql TEXT);
```

可以利用此表来查询数据库中所有的表：

```
tables = cur.execute("select name from sqlite_master where type = 'table' order by name").fetchall()
```

2.3 获取表结构

```
columns = cur.execute("PRAGMA table_info(%s)" % table).fetchall()
```

2.4 获取表中的数据

```
data = cur.execute("select * from %s" % table).fetchall()
```

##3. iOS序列化存储

在iOS开发中，经常将可能会扩展或者根据不同类型存储不同数据的数据库字段定义为blob类型，采用`NSKeyedArchiver`对具有差异性的数据进行序列化，转为NSData数据进行存储。

例如消息的数据表结构可定义为：

```
CREATE TABLE message (
msgId integer primary key,
msgType int,
extensionData blob);
```

其中extensionData就可以根据消息的类型（例如文本、图片、多媒体等），存储不同的数据；同时extensionData中的数据还可以在不改变表结构的条件下，进行其自身的数据扩展。

假设图片消息类定义为：

```
@interface PicMsg : NSObject
@property (nonatomic, assign) NSUInteger msgType;
@property (nonatomic, strong) PicInfo *picInfo;
@end
```

其中PicInfo为图片基本信息：

```
@interface PicInfo : NSObject<NSCoding>
@property (nonatomic, assign) NSUInteger picType;
@property (nonatomic, assign) NSUInteger fileSize;
@property (nonatomic, assign) NSUInteger picWidth;
@property (nonatomic, assign) NSUInteger picHeight;
@end
```

```
@implementation PicInfo
-(void)encodeWithCoder:(NSCoder *)coder
{
    [coder encodeObject:@(self.picType) forKey:@"picType"];
    [coder encodeObject:@(self.fileSize) forKey:@"fileSize"];
    [coder encodeObject:@(self.picWidth) forKey:@"picWidth"];
    [coder encodeObject:@(self.picHeight) forKey:@"picHeight"];
}
-(id)initWithCoder:(NSCoder *)decoder
{
    if (self = [super init]) {
        self.msgType = [[decoder decodeObjectForKey:@"picType"] unsignedIntegerValue];
        self.fileSize = [[decoder decodeObjectForKey:@"fileSize"] unsignedIntegerValue];
        self.picWidth = [[decoder decodeObjectForKey:@"picWidth"] unsignedIntegerValue];
        self.picHeight = [[decoder decodeObjectForKey:@"picHeight"] unsignedIntegerValue];
    }
    return self;
}
@end
```

这样在存入数据库时就可以调用`[NSKeyedArchiver archivedDataWithRootObject:self.picFileInfo];`，将picInfo转化为NSData存入extensionData字段里。

##4. python+pyobjc<sup>[[2]](https://pypi.python.org/pypi/pyobjc)</sup>解析序列化数据

4.1 pyobjc

>PyObjC is a bridge between Python and Objective-C. It allows full featured Cocoa applications to be written in pure Python. It is also easy to use other frameworks containing Objective-C class libraries from Python and to mix in Objective-C, C and C++ source.

在使用pyobjc和iOS相关框架时，只需要import相关模块即可，例如Foundation框架：

```
from Foundation import *
```
 
4.2 pyobjc解析序列化数据

- 定义对应类

要对打包数据进行解包，需要用python定义与OC相对应的类。
例如第3节中提到的`PicInfo`，改用python定义为:

```
class PicInfo(NSObject):
    def init(self):
        self = super(PicInfo, self).init()
        if self is None:
            print "init None"
        return self
    def initWithCoder_(self, encoder):
        self = super(PicInfo, self).init()
        if self is None:
            print "initWithCoder_ None"
        self.picType = encoder.decodeObjectForKey_("picType")
        self.fileSize = encoder.decodeObjectForKey_("fileSize")
        self.picWidth = encoder.decodeObjectForKey_("picWidth")
        self.picHeight = encoder.decodeObjectForKey_("picHeight")
        return self
    def encodeWithCoder_(self, decoder):
        decoder.encodeObject_forKey_(self.picType, "picType")
        decoder.encodeObject_forKey_(self.fileSize, "fileSize")
        decoder.encodeObject_forKey_(self.picWidth, "picWidth")
        decoder.encodeObject_forKey_(self.picHeight, "picHeight")
```

- 数据的序列化与反序列化

```
picInfo = PicInfo.alloc().init()
picInfo.picType = 1
picInfo.fileSize = 1024
picInfo.picWidth = 200
picInfo.picHeight = 100
// 序列化
archivedData = NSKeyedArchiver.archivedDataWithRootObject_(picInfo)
// 反序列化
unarchivedPicInfo = NSKeyedUnarchiver.unarchiveObjectWithData_(archivedData)
```

##5. 结语

python+pyobjc不仅仅可以在iOS开发时进行数据的序列化和反序列化，利用pyobjc可以在Mac OS上使用纯python开发Cocoa GUI应用，功能非常强大。

##参考资料

>[[1]sqlite3 DB-API 2.0 interface for SQLite databases](https://docs.python.org/2/library/sqlite3.html)  
[[2]pyobjc module](https://pypi.python.org/pypi/pyobjc)  
[[3]pyobjc package documentation](http://pythonhosted.org/pyobjc/)
