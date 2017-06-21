---
title: Rails單頁選單顯示表單實作(三)
featured_image: /images/jsresponse.gif
tags: [Rails, ajax, javascript]
excerpt: 繼上一篇使用了標準rails的remote true選項，來完成HTTP POST的ajax call。但我總覺得那個按下「查詢」按鈕的動作是多此一舉的。能不能只有選單，當你在選單中選擇了國家之後，自動就會更新下方的城市列表。在這個系列的第三篇，我們就來看看怎麼做，重點選是ajax call，只是這次我們自己寫，不用rails的。這篇文章著重javascript。
date: 2017-06-13 12:20:43
---
# 前言

雖然在上一篇[Rails單頁選單顯示表單實作(二)](http://josh.hu/2017/06/13/remoteTrue/)中我們用了rails提供的ajax call直接更新`index.html`這個網頁上`<ul div="show_items">`這個元素，但是筆者一直覺得那個查詢按鈕很多餘。應該是我們選擇了國家之後，自動就要列出這個國家的城市，可以少移動一個滑鼠及少按一個按鈕，也比較直覺，我們就來看看怎麼做。

# 準備工作

你只要有這一篇[Rails單頁選單顯示表單實作(一)](http://josh.hu/2017/06/12/samePageForm/)中準備好的project即可，我們就先切換到新的git branch。
```shell
git checkout -b jsresponse
```

# 要修改的地方

這邊要修改的地方有幾個，首先就是`index.html.erb`這個view。因為我們要監測選單變更的事件，並且把選單變更的事件綁定到一個jquery的函數，因此需要給這個選單一個id。此外我們當然要把按鈕拿掉，因此`index.html.erb`就變的非常簡單了，`app/view/cities/index.html.erb`的程式如下：
```javascript
<h2>選擇國家</h2>
<%= select_tag :country_id, options_from_collection_for_select(@countries, "id","name"), id: "filter" %>

<ul id="show_items">
  <%= render @cities %>
</ul>
<%= debug(params) if Rails.env.development? %>
```
**說明**

首先就在第2行，多了一個`id: "filter"`，這是等一下要更動的地方，其它則維持不變。

接下來我們在這個程式的最下方先暫時加一段javascript，如下：
```html
<h2>選擇國家</h2>
<%= select_tag :country_id, options_from_collection_for_select(@countries, "id","name"), id: "filter" %>

<ul id="show_items">
  <%= render @cities %>
</ul>
<%= debug(params) if Rails.env.development? %>

<script>
 $("#filter").change(function(){
   $.ajax({
     url: "<%= cities_path(:js) %>",
     type: 'POST',
     datatype: "script",
     data: {country_id: $(this).val() },
     success: function(data){
       console.log(data);
     }
   });
 });
</script>
```

**說明**

第9到21行是標準的javascript，用來綁定選單變動時執行的事件函數。第11到17行是這個函數的本身。

首先是10行，當`#filter`這個id的選單(即第2行)有變動時，即執行第11到17行的ajax呼叫。

第11到17行的是標準jquery呼叫ajax的語法，包括了幾個參數：
* `url`: 當發生ajax call時，這邊請求的頁面寫成eruby參數的格式，如`"<%= cities_path(:js) %>"`。
* `type`: `'POST'`，表示是用POST。
* `datatype`: 我們要求伺服器傳回javascript執行，因此這邊寫`"script"`。如果是要資料的話，可以是`"json"`。
* `data`: 這就是我們要傳回伺服器的資料，也就是rails中的`params`。這邊指定`country_id`這個值為`$(this).val()`，表示取這個選單中選中的值。
* `success`: 這是執行ajax call成功之後的執行函數，通常就會把從伺服器傳回來的資料一起傳到這函數。我們可以用一個`console.log`來印出傳回來的資料。

因此我們在選單變動時就直接執行ajax call，用HTTP POST去`/cities`所代表controller中的action來執行(也就是`create`這個action)。由於我們指定了`datatype`是script，因此`create`這個action對應到的template就會是javascript，因此就會去執行`create.js.erb`，之後就和上一篇的動作一樣了。

## `create`的回應

因為我們在上面的`datatype`已經指定了`script`，因此controller中的action自動會回應javascript，當然正確的寫法還是要在action中指定回應的格式，如下
```ruby
# app/controller/cities_controller.rb
 class CitiesController < ApplicationController
   def index
     @countries = Country.all
     @cities = City.where(country_id: params[:country_id])
   end
   def create
     @cities = City.where(country_id: params[:country_id])
     respond_to :js
   end
 end
```

其中第9行可寫可不寫，他自己會知道要去找`create.js.erb`。

# 整個完整流程

接下來是整個完整流程

1. 使用者在瀏覽器中輸入`http://192.168.1.105:4000/cities`，就是在向後端的伺服器發出GET HTTP。
2. 伺服器檢查使用者的請求，去查routes是否存在這個請求。
3. 伺服器發現這個請求對應的是`cities#index`這個action，因此執行這個action中的動作。
4. 執行`index`這個action之後，把變數代表的值丟到對應的template `index.html.erb`中
5. `index.html.erb`把變數代表的值換掉其中的變數，成為`index.html`。
6. `index.html`傳回使用者瀏覽器執行。
7. 使用者在瀏覽器中從選單選擇國家。
8. 此時觸發了`'#filter'`這個id的change事件，在使用者的瀏覽器上執行ajax call
9. 根據ajax call的參數，發現是一個POST事件，因此根據參數值去伺服器端
10. 伺服器端接到ajax POST的要求，去routes找，發現是`cities_controller.rb`中的`create`這個action。
11. 此action接收了ajax參數中的`country_id`值，找到城市。
12. 接下個因為ajax call是要求`script`型態，因此就去找`create.js.erb`，並且把其中的變數換成正常值。
13. 換完之後，就傳回`create.js`到使用者瀏覽器。
14. 使用者瀏覽器接收到資料，就執行這個javascript，就是更新`<ul id="show_items">`這個元素內的值。


![](/images/jsresponse.gif)

我們可以從上圖中看到從伺服器傳回來的值，就是完整的一段script，並且把其中的html都更新成城市了。
