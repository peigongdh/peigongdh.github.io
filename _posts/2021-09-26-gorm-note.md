---
layout: post
title:  "gorm 笔记"
date:   2021-09-26 00:00:00 +0800
categories: cs
tag: gorm
---

## gorm在事务中避免条件污染

使用Session.WithConditions = false避免

```go
tx = tx.Table(new(official.TermLevel).TableName()).Session(&gorm.Session{WithConditions: false})
```

注意：此处存在bug，session会过滤所有条件，在链式中应该前置，以上面为例，假设存在分表Table无法工作

```go
tx = tx.Session(&gorm.Session{WithConditions: false}).Table(new(official.TermLevel).TableName())
```

Model应该配合Update使用，待验证

## gorm中使用外键preload

```go
type Post struct {
	ID               int64              `gorm:"column:id" json:"id" form:"id"`
	AppId            string             `gorm:"column:app_id" json:"app_id" form:"app_id"`
	PostAuthorName   string             `gorm:"column:post_author_name" json:"post_author_name" form:"post_author_name"`
	PostDate         time.Time          `gorm:"column:post_date" json:"post_date" form:"post_date"`
	PostStatus       string             `gorm:"column:post_status" json:"post_status" form:"post_status"`
	Icon             string             `gorm:"column:icon" json:"icon" form:"icon"`
	Desc             string             `gorm:"column:desc" json:"desc" form:"desc"`
	PostTitle        string             `gorm:"column:post_title" json:"post_title" form:"post_title"`
	PostContent      string             `gorm:"column:post_content" json:"post_content" form:"post_content"`
	TermRelationship []TermRelationship `gorm:"ForeignKey:ObjectId;AssociationForeignKey:ID"`
	CreatedAt        time.Time          `gorm:"column:created_at" json:"created_at" form:"created_at"`
	UpdatedAt        time.Time          `gorm:"column:updated_at" json:"updated_at" form:"updated_at"`
}

type TermRelationship struct {
	ID        int64     `gorm:"column:id" json:"id" form:"id"`
	ObjectId  int64     `gorm:"column:object_id" json:"object_id" form:"object_id"`
	TermId    int64     `gorm:"column:term_id" json:"term_id" form:"term_id"`
	OrderType string    `gorm:"column:order_type" json:"order_type" form:"order_type"`
	Term      Term      `gorm:"ForeignKey:TermId;AssociationForeignKey:ID"`
	Post      Post      `gorm:"ForeignKey:ObjectId;AssociationForeignKey:ID"`
	CreatedAt time.Time `gorm:"column:created_at" json:"created_at" form:"created_at"`
	UpdatedAt time.Time `gorm:"column:updated_at" json:"updated_at" form:"updated_at"`
}

type Term struct {
	ID        int64     `gorm:"column:id" json:"id" form:"id"`
	AppId     string    `gorm:"column:app_id" json:"app_id" form:"app_id"`
	Taxonomy  string    `gorm:"column:taxonomy" json:"taxonomy" form:"taxonomy"`
	Level     int64     `gorm:"column:level" json:"level" form:"level"`
	Name      string    `gorm:"column:name" json:"name" form:"name"`
	Icon      string    `gorm:"column:icon" json:"icon" form:"icon"`
	Desc      string    `gorm:"column:desc" json:"desc" form:"desc"`
	CreatedAt time.Time `gorm:"column:created_at" json:"created_at" form:"created_at"`
	UpdatedAt time.Time `gorm:"column:updated_at" json:"updated_at" form:"updated_at"`
}
```

在Post中
post和term的关系存储在term_relationship中，所以应该指定外键为objectId：gorm:"ForeignKey:ObjectId;AssociationForeignKey:ID"

在TermRelationship中
自己的字段TermId，ObjectId也是外键

```go
type TermLevel struct {
	ID             int64     `gorm:"column:id" json:"id" form:"id"`
	TermId         int64     `gorm:"column:term_id" json:"term_id" form:"term_id"`
	ParentId       int64     `gorm:"column:parent_id" json:"parent_id" form:"parent_id"`
	TermById       Term      `gorm:"ForeignKey:TermId;AssociationForeignKey:ID"`
	TermByParentId Term      `gorm:"ForeignKey:ParentId;AssociationForeignKey:ID"`
	CreatedAt      time.Time `gorm:"column:created_at" json:"created_at" form:"created_at"`
	UpdatedAt      time.Time `gorm:"column:updated_at" json:"updated_at" form:"updated_at"`
}

// Preload("TermById")
// Preload("TermByParentId")
```

同时可以类似上面为TermId和ParentId指定同一个表的外键，只是需要在preload时分别加载

## 动态表名

Gorm建议在动态表名使用Scopes实现：

```go
func UserTable(user User) func (tx *gorm.DB) *gorm.DB {
  return func (tx *gorm.DB) *gorm.DB {
    if user.Admin {
      return tx.Table("admin_users")
    }

    return tx.Table("users")
  }
}

db.Scopes(UserTable(user)).Create(&user)
```
