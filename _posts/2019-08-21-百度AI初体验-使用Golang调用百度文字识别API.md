---
layout:     post
title:      百度AI初体验-使用Golang调用百度文字识别API
subtitle:   OCR Demo
date:       2019-08-21
author:     JC
header-img: img/OCR.jpg
catalog: true
tags:
    - OCR
    - Golang
    - Baidu AI
---

## 开通API

通过ai.baidu.com或cloud.baidu.com进行登录，进入“管理控制台”后，在“已开通服务”板块下的“文字识别”中申请开通API。

## API使用流程

1、调用获取access token的接口，需提供客户API Key和Secret Key；

2、调用具体api，需提供access token；

3、access token可以反复使用，但有一定的有效期，失效后需要重新获取。

## Demo代码

	package main

	import (
		"encoding/base64"
		"encoding/json"
		"fmt"
		"github.com/asaskevich/govalidator"
		"github.com/bitly/go-simplejson"
		"io/ioutil"
		"net/http"
		"net/url"
		"strings"
	)

	const (
		clientID     = "cmnLxxxxVgyaEf"           			// API Key
		clientSecret = "zPkV7xxxxxZH77Q2"  					// Secret Key
	)

	type Session struct {
		AccessToken   string `json:"access_token"`
		RefreshToken  string `json:"refresh_token"`
		SessionKey    string `json:"session_key"`
		SessionSecret string `json:"session_secret"`
		Scope         string `json:"scope"`
		ExpiresIn     int    `json:"expires_in"`
	}


	func main() {
		token, err := accessToken(clientID, clientSecret)
		if err != nil {
			fmt.Println("accessToken Error: ", err)
			return
		}
		fmt.Println("Token:", *token)

		img, err := ioutil.ReadFile("test.JPG")//官方提示JPG成功率较高
		if err != nil {
			fmt.Println("ReadFile Error: ", err)
			return
		}

		err = ImageToText(*token, img)
		if err != nil {
			fmt.Println("ImageToText Error: ", err)
			return
		}
	}

	func accessToken(id string, secret string) (token *string, err error) {
		apiURL := fmt.Sprintf("https://aip.baidubce.com/oauth/2.0/token?grant_type=client_credentials&client_id=%s&client_secret=%s", id, secret)
		resp, err := http.Get(apiURL)
		if err != nil {
			fmt.Println("HTTP Get Error")
			return nil, err
		}
		defer resp.Body.Close()
		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			fmt.Println("Read Response Body Error")
			return nil, err
		}
		var session Session
		err = json.Unmarshal(body, &session)
		if err != nil {
			fmt.Println("Unmarshal Session Json Error")
			return nil, err
		}
		token = &session.AccessToken
		return token, nil
	}	
		
	func ImageToText(token string, image []byte) error {
		apiURL := fmt.Sprintf("https://aip.baidubce.com/rest/2.0/ocr/v1/accurate_basic?access_token=%s", token)
		param := "image=" + url.QueryEscape(base64.StdEncoding.EncodeToString(image))
		resp, err := http.Post(apiURL, "application/x-www-form-urlencoded", strings.NewReader(param))
		if err != nil {
			return err
		}
		defer resp.Body.Close()
		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			fmt.Println("Read Response Body Error")
			return err
		}
		
		json, err := simplejson.NewJson([]byte(body))
		if err != nil {
			fmt.Println("Unmarshal Result Json Error")
			return err
		}
		rows,err := json.Get("words_result").Array()
		if err != nil{
			fmt.Println("Json Get Error: ",err)
		}
		//fmt.Println(rows)
		//这里对结果潦草处理了下便于显示
		for k,_ := range rows{
			words := govalidator.ToString(rows[k])
			words = string([]rune(words)[10:])
			words = strings.Trim(words, "]")
			fmt.Println(words)
		}
		
		return nil
}

![avatar](img/ocr/ocr1.png)

![avatar](img/ocr/ocr2.png)





