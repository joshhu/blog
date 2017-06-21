---
title: Rails單頁選單顯示表單實作(一)
date: 2017-06-12 22:13:24
featured_image: /images/result.gif
tags: [Rails, ajax, javascript]
excerpt:  目前流行的單頁應用程式(SPA)大部分時間我是用node.js來做，但Rails也可以，只是做法不太一樣，這一篇先從最簡單的rails form做起。
---
# 前言

所謂的單頁表單，就是在同一頁面中，上面有一個選單，下面則是選擇區。當你在選單中選擇項目時，下方的選擇區會根據你在選單中的選項而有所變動。目前單頁應用程式(Single Page Appliation, SPA)非常流行，事實上製作SPA最棒的後端還是node.js，我們以後有機會會提到，在本篇文章中，我們使用rails來看看一個非常簡單的單頁表單作法，先用最簡單的http get的方法來完成，這應該是最簡單的rails應用了。

舉例來說，我們有三個國家，分別是美國、中國、日本，每個國家有自己城市如下：

中國
  --北京
  --上海
  --廣州

日本
  --東京
  --北海道
  --大阪

美國
  --洛杉磯
  --紐約
  --芝加哥

我們要的效果就是在選單中選擇國家，同一頁下方的顯示區就會顯示對應的城市。


# 準備環境

作業系統：Mac OS或是Ubuntu 16.04
ruby版本：2.3.0
rails版本：4.2.6
請遵照前面文章安裝rvm來管理不同版本的ruby以及gem set。

# 建立專案

```shell
mkdir updateForm
cd updateForm
echo 2.3.0 > .ruby-version
echo jobexam > .ruby-gemset
rails new .
bundle install
git init
```

其中`.ruby-version`和`.ruby-gemset`是`rvm`管理`ruby`以及`gem set`的專案資源檔，如果目錄中有這兩個檔案，只要進入該目錄，就會自動切換成這兩個檔案指定的`ruby`版本及`gemset`。讀者要注意的是`jobexam`這個`gemset`是之間我已經建立好的。讀者可以自行建立你的`gemset`，只要確定你的`rails`版本為`4.2.6`即可，之後用`rails new .`建立專案，`bundle install`安裝需要的`gem`。

# 建立model及準備樣本資料

我們需要兩個model，一個是國家，一個是城市，輸入下面指令建立

```shell
rails g model country name:string
rails g model city name:string country:reference
rake db:migrate
```

## 說明

首先建立一個model名稱為`country`，只有一個自訂屬性`name`即國家名稱。接下來建立另一個model為`city`，有一個自訂屬性`name`為城市名稱之為，我們要讓系統知道這個model是`country`這個model的子類別，即**一個`country`可以有多個`cities`**，因此加上`country:reference`。這句話的意思其實就是在`city`這個model中加上`belongs_to :country`。

另外我們要到`country`這個model中手動加入擁有多個`cities`的敘述，打開`app/models/country.rb`輸入如下：
```ruby
# app/model/country.rb
class Country < ApplicationRecord
  has_many :cities
end
```

你也可以打開`app/models/city.rb`來看，由於使用了`country:reference`的參數來建立model，因此已經會預設有`belongs_to :country`了。
```ruby
# app/model/city.rb
class City < ApplicationRecord
  belongs_to :country
end 
```


## 準備樣本資料

開啟`db/seeds.rb`，並且輸入資料如下：
```ruby
# db/seeds.rb

Country.delete_all

Country.create!(
  { id: 1,
    name: "美國"
  }
)
Country.create!(
  { id: 2,
    name: "中國"
  }
)
Country.create!(
  { id: 3,
    name: "日本"
  }
)

City.delete_all

City.create!(
  { id: 1,
    name: "洛杉磯",
    country_id: 1
  }
)
City.create!(
  { id: 2,
    name: "紐約",
    country_id: 1
  }
)
City.create!(
  { id: 3,
    name: "芝加哥",
    country_id: 1
  }
)
City.create!(
  { id: 4,
    name: "北京",
    country_id: 2
  }
)
City.create!(
  { id: 5,
    name: "上海",
    country_id: 2
  }
)
City.create!(
  { id: 6,
    name: "廣州",
    country_id: 2
  }
)
City.create!(
  { id: 7,
    name: "東京",
    country_id: 3
  }
)
City.create!(
  { id: 8,
    name: "北海道",
    country_id: 3
  }
)
City.create!(
  { id: 9,
    name: "大阪",
    country_id: 3
  }
)
```

# 建立Controllers

接下來要建立Controllers。由於我們只要顯示城市，因此只要建立`cities controller`以及在`config/routes.rb`中對應的路由。先建立controller。
```
rails g controller cities index create
```

接下來修改路由如下：
```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :cities, only: [:index, :create]
end
```

上面的路由中，我們使用標準的RestFUL資源型式，但由於只需要用到`GET/POST cities`，因此只用兩個`actions`即可。接下來就可以編輯controller了，如下：
```ruby
# app/controllers/cities_controller.rb
class CitiesController < ApplicationController
  def index
    @countries = Country.all
    @cities = City.where(country_id: params[:country_id])
  end
end
```

## 說明：

其中先將所有的國家讀到`@countries`這個變數中給選單用，然後設定`@cities`則是使用者選取選單之後，把國家代碼傳回這個`action`，再去資料庫取出符合國家代碼的城市。

# view

View有兩個部分，一個是`index`這個`action`對應的template，即`app/view/cities/index.html.erb`，另一個則是單獨要列出所有城市的`partial`。如下：

```html
<h2>選擇國家</h2>
<%= form_tag "/cities" ,method: :get do %>
  <%= select_tag :country_id, options_from_collection_for_select(@countries, "id","name")%>
  <%= button_tag "查詢" %>
<% end %>
<ul>
  <%= render @cities %>
</ul>
<%= debug(params) if Rails.env.development? %>
```

## 說明

第2行這邊不使用`form_for`而使用`form_tag`是為了簡化。首先要做的就是加上`method: :get`表示這個表單是要用`GET`，因此在`RestFUL`的動作中就還是會回到`cities#index`這個`action`中，因此我們的controller就沒有對應到`POST`的`create`這個`action`。

第3行使用了標準的`select_tag`，並且使用了`options_from_collection`把`@countries`放入`collection`中成為選單，並且以`id`為`html`的`value`，`name`為顯示出來的值。最後第4行放上一個按鈕送出。**注意**：在rails中，輸入表單一定要有一個送出的機制，因此這邊一定要放入一個`button_tag`，才會產生RestFUL的GET或POST動作。

接下來是第7行的`<%= render @cities %>`。這句敘述表示要將`@cities`這整個collection丟到一個叫做`_city.html.erb`的`partial`來顯示，而且在`_city.html.erb`中，你不需要使用`@cities.each do |city|`的迴圈來執行，這種型式的`partial`會自動把整個`@cities`iterate一遍。因此我們要建立這個`partial`，檔案為`app/view/cities/_city.html.erb`

```html
<li><%= city.name %></li>
```


接下來先將這個版本的程式commit，輸入`git add .`以及`git commit -am "init"`，然後直接啟動，輸入`rails s -b 0.0.0.0 -p 3001`，就可以登入看結果，請到該主機上輸入`http://localhost:3001/cities/`

![](/images/result.gif)


# 待改進處：

1.執行結果使用get到原方法，因此選單中的選項會回到最原始的預設值美國
2.使用get表單，因此雖然看起來是在同一頁，其實是有換頁，正確應該用ajax，就是只更新顯示城市的地方而不要換頁。
3.選擇國家後，應後就直接顯示城市，而不需要一個按鈕。

# 整個完整流程

接下來是整個完整流程

1. 使用者在瀏覽器中輸入`http://192.168.1.105:4000/cities`，就是在向後端的伺服器發出GET HTTP。
2. 伺服器檢查使用者的請求，去查routes是否存在這個請求。
3. 伺服器發現這個請求對應的是'cities#index`這個action，因此執行這個action中的動作。
4. 執行`index`這個action之後，把變數代表的值丟到對應的template `index.html.erb`中
5. `index.html.erb`把變數代表的值換掉其中的變數，成為`index.html`。
6. `index.html`傳回使用者瀏覽器執行。
7. 使用者從`index.html`中的選單選擇，並且按下按鈕。
8. 重複2-6的動作。

