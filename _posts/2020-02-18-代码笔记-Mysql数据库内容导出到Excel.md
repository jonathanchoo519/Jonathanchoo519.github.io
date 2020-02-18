---
layout:     post
title:      代码笔记-MySQL数据库内容导出到Excel
date:       2020-02-18
author:     JC
header-img: img/golang.jpg
catalog: false
tags:
    - golang
    - excelize
---

项目中有时后台需要根据筛选条件将数据导出到EXCEL，主要还是对Go的反射和结构体tag的运用吧，代码如下：

### struct声明及gorm查询
```
type VillageRecord struct {
	gorm.Model
	Area           string	`xlsx:"A-行政区划"`
	TownName       string	`xlsx:"B-乡镇名称"`
	VillageName    string	`xlsx:"C-行政村名称"`
	Name           string	`xlsx:"D-村民姓名"`
	IDCard         string	`xlsx:"E-身份证号"`
	Phone          string	`xlsx:"F-手机号"`
	CarNum         string	`xlsx:"G-车牌号"`
	...
}
func FindAllInfo()(result []*VillageRecord,err error){
	db := gormNew()
	err = db.Find(&result).Error
	return result, err
}
```
### 数据导出
```
func Export2Excel(){
	xlsx := excelize.NewFile()
	index := xlsx.NewSheet("Sheet1")
	result, _ := models.FindAllInfo()

	//PS:便于理解，这里说明是按列写，不是按行写
	//遍历结果集
	for i, t := range result {
		//取出每条结果结构体中的属性集d
		d := reflect.TypeOf(t).Elem()
		//遍历属性集d，注意如果有gorm.model 则index需要从1开始遍历，否则从0
		for j := 1; j < d.NumField(); j++ {
			// 设置表头
			if i == 0 {
				//根据结构体中绑定的tag，根据分隔符，拿到列号
				column := strings.Split(d.Field(j).Tag.Get("xlsx"), "-")[0]
				//同理拿到列名
				name := strings.Split(d.Field(j).Tag.Get("xlsx"), "-")[1]
				// 设置表头
				xlsx.SetCellValue("Sheet1", fmt.Sprintf("%s%d", column, i+1), name)
			}

			// 设置内容
			column := strings.Split(d.Field(j).Tag.Get("xlsx"), "-")[0]
			switch d.Field(j).Type.String() {
			//根据属性的类型进行断言，写入
			case "string":
				xlsx.SetCellValue("Sheet1", fmt.Sprintf("%s%d", column, i+2), reflect.ValueOf(t).Elem().Field(j).String())
			case "int32":
				xlsx.SetCellValue("Sheet1", fmt.Sprintf("%s%d", column, i+2), reflect.ValueOf(t).Elem().Field(j).Int())
			case "int64":
				xlsx.SetCellValue("Sheet1", fmt.Sprintf("%s%d", column, i+2), reflect.ValueOf(t).Elem().Field(j).Int())
			case "bool":
				xlsx.SetCellValue("Sheet1", fmt.Sprintf("%s%d", column, i+2), reflect.ValueOf(t).Elem().Field(j).Bool())
			case "float32":
				xlsx.SetCellValue("Sheet1", fmt.Sprintf("%s%d", column, i+2), reflect.ValueOf(t).Elem().Field(j).Float())
			case "float64":
				xlsx.SetCellValue("Sheet1", fmt.Sprintf("%s%d", column, i+2), reflect.ValueOf(t).Elem().Field(j).Float())
			}
		}
	}

	xlsx.SetActiveSheet(index)
	// 保存到xlsx中

	err := xlsx.SaveAs("test_write.xlsx")
	if err != nil {
		fmt.Println(err)
	}
}
```