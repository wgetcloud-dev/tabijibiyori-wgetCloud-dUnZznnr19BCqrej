
## SQLite数据库


HarmonyOS的关系型数据库基于SQLite
导入模块



```
import { relationalStore } from '@kit.ArkData';

```

实现步骤：


1. 获取RdbStore对象，用于创建数据库，数据表，以及数据库升级等操作



```
let storeConfig = {
  name: 'Poetry.db',  //数据库文件名
  securityLevel: relationalStore.SecurityLevel.S1,  //安全级别
  encrypt: false, //是否加密，可选，默认不加密
  customDir: '', 	//自定义路径，可选，目录：context.databaseDir + '/rdb/' + customDir
  isReadOnly: false, //是否已只读方式打开，可选，默认false
}

relationalStore.getRdbStore(this.context, this.storeConfig)
  .then(store => {
    //创建表
    store.executeSql('sql')
    //判断版本
    store.version
  })
  .catch((err: Error) => {
    
  })

```

2. 插入数据



```
let data :ValuesBucket={
  name:"zhangsan",
  age: 23,
}
store.insert("tableName",data).then((rowId)=>{
  //操作成功返回rowId,否则返回-1
})

store.batchInsert() //用于插入批量数据

```

3. 修改，删除数据：通过组件提供的谓词(Predicates)修改或删除组件



```
let data :ValuesBucket={
  name:"zhangsan",
  age: 26,
}
let predicates = new relationalStore.RdbPredicates("tableName")
predicates.equalTo("name","zhangsan")
//更新数据
store.update(data,predicates).then((value)=>{

})

//删除数据
store.delete(predicates).then((value)=>{

})

```

4. 查询数据



```
let predicates = new relationalStore.RdbPredicates("tableName")
predicates.equalTo("name","zhangsan")
store.query(predicates).then((resultSet)=>{
  while (resultSet.goToNextRow()){
    const name = resultSet.getString(resultSet.getColumnIndex("name"))
    const age = resultSet.getLong(resultSet.getColumnIndex("age"))
  }
  resultSet.close()
})
//也可以通过下面接口使用sql查询
store.querySql(sql: string, bindArgs?: Array<ValueType>): Promise<ResultSet>;

```

5. 备份数据和恢复数据



```
//备份数据
store.backup("backup.db")
//恢复数据
store.restore("backup.db")

```

## SmartDB


SmartDB与Android中的room组件类似，可以简化我们数据库操作的步骤，使代码更易维护。
安装和导入模块



```
//安装模块
ohpm install @liushengyi/smartdb
//导入模块
import sql from "@liushengyi/smartdb"

```

定义数据结构：



```
export class Poetry {
  @sql.SqlColumn(sql.ColumnType.TEXT)
  uuid?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  title?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  dynasty?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  author?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  introduction?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  text?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  textAlign?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  translation?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  rectify?: string
  @sql.SqlColumn(sql.ColumnType.TEXT)
  searchKey?: string
}

```

执行数据库操作：进行增删改查、事务操作



```
export class PoetryDao {
  public static TABLE_NAME = "Poetry"
  public static SQL_CREATE_TABLE =
    "CREATE TABLE IF NOT EXISTS `Poetry` (`uuid` TEXT NOT NULL, `title` TEXT, `dynasty` TEXT, `author` TEXT, `introduction` TEXT, `text` TEXT, `textAlign` TEXT, `translation` TEXT, `rectify` TEXT, `searchKey` TEXT, PRIMARY KEY(`uuid`))"

  @sql.SqlQuery(`select * from ${PoetryDao.TABLE_NAME} where uuid=#{uuid}`)
  @sql.ReturnType(Poetry)
  queryOne(@sql.Param('uuid') uuid: string): Promise<Poetry> {
    return sql.PromiseNull()
  }

  @sql.SqlQuery(`select count(*) from ${PoetryDao.TABLE_NAME} `)
  @sql.ReturnType(Number)
  queryCount(): Promise<Number> {
    return sql.PromiseNull()
  }

  @sql.SqlInsert(`insert into ${PoetryDao.TABLE_NAME} values (#{data.uuid},#{data.title},#{data.dynasty},#{data.author},#{data.introduction},#{data.text},#{data.textAlign},#{data.translation},#{data.rectify},#{data.searchKey})`)
  insert(@sql.Param('data') data: Poetry): Promise<void> {
    return sql.PromiseNull()
  }
  
  @sql.Transactional()
  async insertPoetryAll(list: Poetry[]) {
    for (let item of list) {
      await this.insert(item)
    }
  }
}

```

数据库管理：创建数据库、数据库升级



```
export class DatabaseManager {
  static readonly DATABASE_VERSION = 1
  static readonly DATABASE_NAME = 'poetry.db'

  static init(context: Context) {
    sql.dbHelper.initDb(context,
      DatabaseManager.DATABASE_NAME,
      DatabaseManager.DATABASE_VERSION,
      new DbOpenHelperImpl()
    )
  }
}

class DbOpenHelperImpl extends sql.DbOpenHelper {
  //创建数据库
  onCreate(db: relationalStore.RdbStore): void {
    db.executeSql(PoetryDao.SQL_CREATE_TABLE)
  }

  //升级数据
  onUpgrade(db: relationalStore.RdbStore, oldVersion: number, newVersion: number): void {

  }
}

```

最后在app启动的时候调用初始化方法



```
export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    hilog.info(0x0000, 'testTag', '%{public}s', 'Ability onCreate');
    DatabaseManager.init(this.context)
  }
}

```

## 数据初始化和使用


将json格式的数据插入到数据库中



```
poetryDao: PoetryDao = new PoetryDao()

new Promise(async (resolve, reject) => {
  try {
    let count = await this.poetryDao.queryCount()
    if (!count) {
      let list = await (await import("resources/rawfile/poetry.json")).default
      let poetryArray: Array<Poetry> = []
      for (let item of list) {
        let poetry = item as Poetry
        poetry.uuid = util.generateRandomUUID()
        if (poetry.rectify) {
          poetry.rectify = JSON.stringify(poetry.rectify)
        }
        poetryArray.push(poetry)
      }
      this.poetryDao.insertPoetryAll(poetryArray)
      resolve(true)
    } else {
      resolve(false)
    }
  } catch (e) {
    reject(e)
  }
})

```

读取数据



```
//读取所有数据
new PoetryDao().queryList()
  .then((value) => {

  })
//读取一条数据
new PoetryDao().queryOne("id")
  .then((value) => {

  })

```



---


本文的技术设计和实现都是基于作者工作中的经验总结，如有错误，请留言指正，谢谢。


 本博客参考[樱花宇宙官网](https://yzygzn.com)。转载请注明出处！
