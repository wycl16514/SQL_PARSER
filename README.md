我们接上节内容继续完成SQL解释器的代码解析工作。下面我们实现对update语句的解析，其语法如下：
UpdateCmd -> INSERT | DELETE | MODIFY | CREATE
Create -> CreateTable | CreateView | CreateIndex
Insert -> INSERT INTO ID LEFT_PARAS FieldList RIGHT_PARAS VALUES LEFT_PARS ConstList RIGHT_PARAS
FieldList -> Field  ( COMMA  FieldList)?
ConstList -> Constant ( COMMA ConstList)?
Delete -> DELETE FROM ID [WHERE Predicate)?
Modify -> UPDATE ID SET Field ASSIGN_OPERATOR Expression (WHERE Predicate)?
CreateTable -> CREATE TABLE ID (FieldDefs)?
FieldDefs -> FieldDef ( COMMA FieldDefs)?
FieldDef -> ID TypeDef 
TypeDef -> INT | VARCHAT LEFT_PARAS NUM RIGHT_PARAS
CreateView -> CREATE VIEW ID AS Query 
CreateIndex -> CREATE INDEX ID ON ID LEFT_PARAS Field RIGHT_PARAS

我们对上面的语法做一些基本说明：
UpdateCmd -> INSERT | DELETE | MODIFY | CREATE
这句语法表明SQL语言中用于更新表的语句一定由insert, delete, modify , create等几个命令开始。insert 语句由关键字insert开始，然后跟着insert into两个关键字，接着是左括号，跟着是由列名(column)组成的字符串，他们之间由逗号隔开，然后跟着右括号，接着是关键字VALUES，然后是左括号，接着是一系列常量和逗号组成的序列，最后以又括号结尾，其他语法大家可以参照SQL相关命令来理解，下面我们看看代码的实现，继续在parser.go中添加如下代码：

```go
func (p *SQLParser) UpdateCmd() interface{} {
	tok, err := p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}

	if tok.Tag == lexer.INSERT {
		p.sqlLexer.ReverseScan()
		return p.Insert()
	} else if tok.Tag == lexer.DELETE {
		p.sqlLexer.ReverseScan()
		return p.Delete()
	} else if tok.Tag == lexer.UPDATE {
		p.sqlLexer.ReverseScan()
		return p.Update()
	} else {
		p.sqlLexer.ReverseScan()
		return p.Create()
	}
}

func (p *SQLParser) Create() interface{} {
	tok, err := p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}
	if tok.Tag != lexer.CREATE {
		panic("token is not create")
	}

	tok, err = p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}

	if tok.Tag == lexer.TABLE {
		return p.CreateTable()
	} else if tok.Tag == lexer.VIEW {
		return p.CreateView()
	} else {
		return p.CreateIndex()
	}
}

func (p *SQLParser) CreateView() interface{} {
	return nil
}

func (p *SQLParser) CreateIndex() interface{} {
	return nil
}

func (p *SQLParser) CreateTable() interface{} {
	tok, err := p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}
	if tok.Tag != lexer.ID {
		panic("token should be ID for table name")
	}

	tblName := p.sqlLexer.Lexeme
	tok, err = p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}
	if tok.Tag != lexer.LEFT_BRACKET {
		panic("missing left bracket")
	}
	sch := p.FieldDefs()
	tok, err = p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}
	if tok.Tag != lexer.RIGHT_BRACKET {
		panic("missing right bracket")
	}

	return NewCreateTableData(tblName, sch)
}

func (p *SQLParser) FieldDefs() *record_manager.Schema {
	schema := p.FieldDef()
	tok, err := p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}
	if tok.Tag == lexer.COMMA {
		schema2 := p.FieldDefs()
		schema.AddAll(schema2)
	} else {
		p.sqlLexer.ReverseScan()
	}

	return schema
}

func (p *SQLParser) FieldDef() *record_manager.Schema {
	_, fldName := p.Field()
	return p.FieldType(fldName)
}

func (p *SQLParser) FieldType(fldName string) *record_manager.Schema {
	schema := record_manager.NewSchema()
	tok, err := p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}

	if tok.Tag == lexer.INT {
		schema.AddIntField(fldName)
	} else if tok.Tag == lexer.VARCHAR {
		tok, err := p.sqlLexer.Scan()
		if err != nil {
			panic(err)
		}
		if tok.Tag != lexer.LEFT_BRACKET {
			panic("missing left bracket")
		}

		tok, err = p.sqlLexer.Scan()
		if err != nil {
			panic(err)
		}

		if tok.Tag != lexer.NUM {
			panic("it is not a number for varchar")
		}

		num := p.sqlLexer.Lexeme
		fldLen, err := strconv.Atoi(num)
		if err != nil {
			panic(err)
		}
		schema.AddStringField(fldName, fldLen)

		tok, err = p.sqlLexer.Scan()
		if err != nil {
			panic(err)
		}
		if tok.Tag != lexer.RIGHT_BRACKET {
			panic("missing right bracket")
		}
	}

	return schema
}
```
在上面代码中我们需要定义一个CreateTableData结构，因此增加一个create_data.go文件，添加代码如下：
```go
package parser

import (
	"record_manager"
)

type CreateTableData struct {
	tblName string
	sch     *record_manager.Schema
}

func NewCreateTableData(tblName string, sch *record_manager.Schema) *CreateTableData {
	return &CreateTableData{
		tblName: tblName,
		sch:     sch,
	}
}

func (c *CreateTableData) TableName() string {
	return c.tblName
}

func (c *CreateTableData) NewSchema() *record_manager.Schema {
	return c.sch
}

```
最后我们在main.go中添加代码，调用上面的代码实现：
```go
package main

//import (
//	bmg "buffer_manager"
//	fm "file_manager"
//	"fmt"
//	lm "log_manager"
//	"math/rand"
//	mm "metadata_management"
//	record_mgr "record_manager"
//	"tx"
//)

import (
	"parser"
)

func main() {
	sql := "create table person (PersonID int, LastName varchar(255), FirstName varchar(255)," +
		"Address varchar(255), City varchar(255) )"
	sqlParser := parser.NewSQLParser(sql)
	sqlParser.UpdateCmd()

}
```

在main中，我们定义了一个create table的sql语句，然后调用UpdateCmd接口实现语法解析，大家可以在b站搜索”coding迪斯尼“，查看代码的调试演示视频，由于上面语法解析的逻辑稍微复杂和繁琐，因此通过视频来跟踪代码的单步调试过程才能更简单省力的理解实现逻辑。

下面我们看看insert语句的解析实现，在parser.go中添加代码如下：
```go
func (p *SQLParser) checkWordTag(wordTag lexer.Tag) {
	tok, err := p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}
	if tok.Tag != wordTag {
		panic("token is not match")
	}
}

func (p *SQLParser) isMatchTag(wordTag lexer.Tag) bool {
	tok, err := p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}

	if tok.Tag == wordTag {
		return true
	} else {
		p.sqlLexer.ReverseScan()
		return false
	}
}


func (p *SQLParser) fieldList() []string {
	L := make([]string, 0)
	_, field := p.Field()
	L = append(L, field)
	if p.isMatchTag(lexer.COMMA) {
		fields := p.fieldList()
		L = append(L, fields...)
	}

	return L
}

func (p *SQLParser) constList() []*query.Constant {
	L := make([]*query.Constant, 0)
	L = append(L, p.Constant())
	if p.isMatchTag(lexer.COMMA) {
		consts := p.constList()
		L = append(L, consts...)
	}

	return L
}

func (p *SQLParser) Insert() interface{} {
	/*
		根据语法规则：Insert -> INSERT INTO ID LEFT_PARAS FieldList RIGHT_PARAS VALUES LEFT_PARS ConstList RIGHT_PARAS
		我们首先要匹配四个关键字，分别为insert, into, id, 左括号,
		然后就是一系列由逗号隔开的field,
		接着就是右括号，然后是关键字values
		接着是常量序列，最后以右括号结尾
	*/
	p.checkWordTag(lexer.INSERT)
	p.checkWordTag(lexer.INTO)
	p.checkWordTag(lexer.ID)
	tblName := p.sqlLexer.Lexeme
	p.checkWordTag(lexer.LEFT_BRACKET)
	flds := p.fieldList()
	p.checkWordTag(lexer.RIGHT_BRACKET)
	p.checkWordTag(lexer.VALUES)
	p.checkWordTag(lexer.LEFT_BRACKET)
	vals := p.constList()
	p.checkWordTag(lexer.RIGHT_BRACKET)

	return NewInsertData(tblName, flds, vals)
}

```
我们调用上面代码测试一下解析效果：
```go
func main() {
	
	sql := "INSERT INTO Customers (CustomerName, ContactName, Address, City, PostalCode, Country) " +
		"VALUES (\"Cardinal\", \"Tom B. Erichsen\", \"Skagen 21\", \"Stavanger\", 4006, \"Norway\")"
	sqlParser := parser.NewSQLParser(sql)
	sqlParser.UpdateCmd()

} 
```
请大家在b站搜索coding迪斯尼，通过视频调试演示的方式能更直白和有效的了解代码逻辑。接下来我们看看 create 命令如何创建 view 和 index 两个对象，首先我们看看 view 的创建，根据 create view 的语法：
```go
CreateView -> CREATE VIEW ID AS QUERY
```
首先我们要判断语句的前两个 token 是否对应 关键字 CREATE, VIEW，然后接着的token 必须是 ID类型，然后跟着关键字 AS,最后我们调用 QUERY 对应的解析规则来解析后面的字符串，我们看看代码实现，在 parser.go 中添加如下代码：
```go
func (p *SQLParser) CreateView() interface{} {
	p.checkWordTag(lexer.ID)
	viewName := p.sqlLexer.Lexeme
	p.checkWordTag(lexer.AS)
	qd := p.Query()

	vd := NewViewData(viewName, qd)
	vdDef := fmt.Sprintf("vd def: %s", vd.ToString())
	fmt.Println(vdDef)
	return vd
}
```
然后新增文件 create_view.go，添加如下代码：
```
package parser

import "fmt"

type ViewData struct {
	viewName  string
	queryData *QueryData
}

func NewViewData(viewName string, qd *QueryData) *ViewData {
	return &ViewData{
		viewName:  viewName,
		queryData: qd,
	}
}

func (v *ViewData) ViewName() string {
	return v.viewName
}

func (v *ViewData) ViewDef() string {
	return v.queryData.ToString()
}

func (v *ViewData) ToString() string {
	s := fmt.Sprintf("view name %s, viewe def: %s", v.viewName, v.ViewDef())
	return s
}

```
最后我们在 main.go 中添加如下测试代码：
```
func main() {
	//sql := "create table person (PersonID int, LastName varchar(255), FirstName varchar(255)," +
	//	"Address varchar(255), City varchar(255) )"

	sql := "create view Customer as select CustomerName, ContactName from customers where country=\"China\""
	sqlParser := parser.NewSQLParser(sql)
	sqlParser.UpdateCmd()

}
```
上面代码运行后结果如下：
```
vd def: view name Customer, viewe def: select CustomerName, ContactName, from customers, where  and country=China
```
更详细的内容请在 b 站搜索 coding 迪斯尼。下面我们看看索引创建的语法解析，其对应的语法为：
```
CreateIndex -> CREATE INDEX ID ON ID LEFT_BRACKET Field RIGHT_BRACKET
```
从语法规则可以看出，在解析时我们需要判断语句必须以 CREATE INDEX 这两个关键字开头，然后接着的字符串要能满足 ID 的定义，然后又需要跟着关键字 ON, 然后跟着的字符串要满足 ID 定义，接下来读入的字符必须是左括号，然后接着的内容要满足 Field 的定义，最后要以右括号结尾，我们看看代码实现在 parser.go 中添加如下代码：

```
func (p *SQLParser) Create() interface{} {
	tok, err := p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}
	if tok.Tag != lexer.CREATE {
		panic("token is not create")
	}

	tok, err = p.sqlLexer.Scan()
	if err != nil {
		panic(err)
	}

	if tok.Tag == lexer.TABLE {
		return p.CreateTable()
	} else if tok.Tag == lexer.VIEW {
		return p.CreateView()
	} else if tok.Tag == lexer.INDEX {
		return p.CreateIndex()
	}

	panic("sql string with create should not end here")
}

func (p *SQLParser) CreateIndex() interface{} {
	p.checkWordTag(lexer.ID)
	idexName := p.sqlLexer.Lexeme
	p.checkWordTag(lexer.ON)
	p.checkWordTag(lexer.ID)
	tableName := p.sqlLexer.Lexeme
	p.checkWordTag(lexer.LEFT_BRACKET)
	_, fldName := p.Field()
	p.checkWordTag(lexer.RIGHT_BRACKET)

	idxData := NewIndexData(idexName, tableName, fldName)
	fmt.Printf("create index result: %s", idxData.ToString())
	return idxData
}
```
新建 create_index_data.go 文件，在里面添加代码如下：
```
package parser

import "fmt"

type IndexData struct {
	idxName string
	tblName string
	fldName string
}

func NewIndexData(idxName string, tblName string, fldName string) *IndexData {
	return &IndexData{
		idxName: idxName,
		tblName: tblName,
		fldName: fldName,
	}
}

func (i *IndexData) IdexName() string {
	return i.idxName
}

func (i *IndexData) tableName() string {
	return i.tblName
}

func (i *IndexData) fieldName() string {
	return i.fldName
}

func (i *IndexData) ToString() string {
	str := fmt.Sprintf("index name: %s, table name: %s, field name: %s", i.idxName, i.tblName, i.fldName)
	return str
}

```
在 main.go 中我们使用 sql 语句中的 create index 语句测试一下上面代码实现：
```
func main() {
	//sql := "create table person (PersonID int, LastName varchar(255), FirstName varchar(255)," +
	//	"Address varchar(255), City varchar(255) )"

	sql := "create index idxLastName on persons (lastname)"
	sqlParser := parser.NewSQLParser(sql)
	sqlParser.UpdateCmd()

}
```
上面代码运行后所得结果如下：
```
create index result: index name: idxLastName, table name: persons, field name: lastname
```
到这里所有有关 create 语句的解析就基本完成，更多的调试演示和代码逻辑的讲解，请在 b 站搜索 coding 迪斯尼
