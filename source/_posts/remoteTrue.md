---
title: Rails單頁選單顯示表單實作(二)
date: 2017-06-13 00:32:10
featured_image: /images/remotetrue.gif
excerpt: 前一篇我們用標準的HTTP GET更新表單，取代了傳統的HTTP POST，因此只需要一個action。 但這個方法問題很多，因此在這一篇中，我們使用了rails的ajax。只要在表單中放在remote true的參數，就會用rails的ajax call，不但程式碼變簡單，也更符合一般網頁的使用習慣，當然，這還不是最好的做法。
tags: [Rails, ajax, javascript]
---

# 前言

在選擇網頁後端語言時，我不斷強調，只要不選擇某些過時的語言都可以，因為網頁語言最重的是在前端，而前端當然就是javascript。不過如果你有幸身在一個大的團隊，前端有人處理了，只要進行後端的開發，那你使用rails提供的jquery library也就夠了。延續上一篇[Rails單頁選單顯示表單實作(一)](http://josh.hu/2017/06/12/samePageForm/)中的作法，當使用者選擇選單中的項目，並且按下送出時，我們把回應action的原來HTTP GET反應，換成HTTP POST，並且把原來回應對應的Template從html換成javascript，再讓這個javascript來更新div而不是整個網頁，直接使用rails內建的ajax call即可。

# 準備工作

你只要有上一篇[Rails單頁選單顯示表單實作(一)](http://josh.hu/2017/06/12/samePageForm/)中準備好的project即可，我們就先切換到新的git branch。
```shell
git checkout -b remotetrue
```

# 要修改的地方

這邊要修改的地方有幾個

* 原來`index`這個action的view。
* 在controller中新增一POST的action。
* 對應到這個POST action的Template本來應該是html，換成javascript。

我們就來看看：
## 修改cities controller

這邊主要是增加一個`create`的action，因為我們使用了標準的RestFUL的語法，因此對應到`/cities`的POST方法名稱就是`create`。

```ruby
# app/controllers/cities_controller.rb
class CitiesController < ApplicationController

  def index
    @countries = Country.all
    @cities = City.where(country_id: params[:country_id])
  end

  def create
    @cities = City.where(country_id: params[:country_id])
  end

end
```

** 說明**

`create`很簡單，就是從使用者的選單中讀取選中的`country_id`，然後再到model中尋找對應該國家碼的城市。

## 修改index這個view

我們要修改的檔案為`app/view/cities/index.html.erb`，程式碼如下：
```javascript
<h2>選擇國家</h2>
<%= form_tag "/cities", remote: true do %>
  <%= select_tag :country_id, options_from_collection_for_select(@countries, "id","name") %>
  <%= button_tag "查詢" %>
<% end %>
<ul id="show_items">
  <%= render @cities %>
</ul>
<%= debug(params) if Rails.env.development? %>
```

** 說明**

這邊最主要的就是第2行中的`remote: true`這個敘述。在rails中，當我們在表單中放入`remote: true`之後，rails就會自動把這個表單在controller中所對應到的action(本例為cities_controller的create)，其所`respond_to`的檔案類型設定為javascript。本來這個action的template應該是`create.html.erb`，但有了`remote: true`之後，對應到的template就變成了`create.js.erb`了。

你當然可以在`create.js.erb`中再度指定新的html，但這就多此一舉了。使用這種機制最常見的作法就是更新原來view中的某一HTML元件。因此我們在上面的程式碼第6行，幫顯示城市的地方加了一個ul的id`show_items`，然後我們只要到`create.js.erb`這個javscript中更新`index.html`中`show_items`這個`<ul>`，你只要記得，我們通常使用javascript啟動ajax call去伺服器端要資料(data)，而不是要頁面(page)，載入速度就會很快，並且可以做到很多動態的效果。

## 新增`create.js.erb`

接下來就是新增一個對應到`create`這個action的template，這邊就是`app/view/cities/create.js.erb`。程式碼如下：
```javascript
$('#show_items').html("<%= j render @cities %>");
```

** 說明 **
這是一個標準的javascript內嵌rails指令，短短一行，使用了jquery的html函數，而更新的內容，正是我們要求的`render @cities`原來這一塊。

# 整個完整流程

接下來是整個完整流程

1. 使用者在瀏覽器中輸入`http://192.168.1.105:4000/cities`，就是在向後端的伺服器發出GET HTTP。
2. 伺服器檢查使用者的請求，去查routes是否存在這個請求。
3. 伺服器發現這個請求對應的是'cities#index`這個action，因此執行這個action中的動作。
4. 執行`index`這個action之後，把變數代表的值丟到對應的template `index.html.erb`中
5. `index.html.erb`把變數代表的值換掉其中的變數，成為`index.html`。
6. `index.html`傳回使用者瀏覽器執行。
7. 使用者在瀏覽器中從選單選擇國家，並且按下傳送按鈕。
8. 此時瀏覽器發現使用者發出了HTTP POST的請求，並且因為有`data-remote`這個選項，因此認定這是一個ajax call。
9. 伺服器端查路由，找HTTP POST對應到的是`cities#create`這個action。
10. 去執行`cities_controller`中的'create` action。執行之後，把變數代表的值丟到對應的template
11. 由於是ajax call，因此template不再是html，而是javascript，就是`create.js.erb`。
12. 把變數代表的值在`create.js.erb`中換成正常值，生成`create.js`。
13. 把`create.js`傳回使用者的瀏覽器。
14. 使用者瀏覽器接收到`create.js`，並且在瀏覽器中執行。
15. 執行的程式碼是`$('#show_items').html('<li>北京</li><li>上海</li><li>廣州</li>');`。因此就直接更新`<ul id="show_items">`這個部分。

![](/images/remotetrue.gif)
