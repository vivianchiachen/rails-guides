**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON http://guides.rubyonrails.org.**

Active Record 基礎
====================

本篇是Active Record 的基礎介紹。

讀完本篇之後，你將會知道：

* 什麼是 Object Relational Mapping 和 Active Record 以及如何在 Rails 中使用它們。
* Active Record 是如何運用在 Model-View-Controller 中。
* 如何運用 Active Record Model 去操作儲存在關聯式資料庫中的資料。
* Active Record 的命名慣例。
* 資料庫遷移、驗證與回呼的概念。

--------------------------------------------------------------------------

什麼是 Active Record?
----------------------

Active Record 是 [MVC](http://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) 中的 M （Model），是整個系統中負責呈現商業資料與邏輯的層級。 Active Record使需要長期儲存在資料庫中的資料更容易地被新增與使用。Active Record的執行模式本身就是一種 Object Relational Mapping 系統的形式。

### Active Record 的模式

可參考 Matin Fowler 所著的 _Patterns of Enterprise Application Architecture_ 中對 [Active Record 模式的描述](http://www.martinfowler.com/eaaCatalog/activeRecord.html)。在 Active Record 中，物件帶有長久性的資料與行為。因此 Active Record 確保存取邏輯為物件的一部分，進而教導使用者如何寫入與讀取資料庫中的資料。


### Object Relational Mapping

Object Relational Mapping ，通常簡稱 ORM，是一種將應用程式中大量物件，對應到關係式管理資料庫中表格的一種技巧。使用 ORM，能輕易地將應用程式中物件的屬性與關聯性進行存取，省去了寫 SQL指令的需要，總地減少了許多存取資料時需要的程式碼。

### Active Record 作為 ORM 的框架

Active Record 提供了我們許多機制，其中重要的功能有：

* 表示 Model 與其資料。
* 表示這些 Model 之間的關聯性。
* 表示相關 Model 之間的等級繼承關係。
* 存入資料庫之前將資料進行驗證。
* 使資料庫呈現物件導向的運作。



在 Active Record 中的慣例勝於設定
----------------------------------------------

當我們利用其他程式語言或框架寫應用程式的時候，通常需要寫許多設定的程式碼，對於 ORM 這個框架而言也是如此。然而，若我們能依循 Rails 的慣例，在新增 Active Record Model 時就不需要太多的設定（有些情況下根本不需設定）。主要的概念是，如果每一次在設定應用程式時都是使用相同的設定，那麼這就是預設的設定方式。因此，只有無法依循標準慣例時，才會需要寫入特定的設定。

### 命名慣例

在預設的情況下，Active Record 會根據某種命名的慣例，將 Model 與資料表的對應關聯建立起來，Rails 會將類別名稱轉為複數來找到對應的資料表。所以 `Book` 對應的資料表便是 **books**。Rails 的單複數轉換機制非常地強大，它能複數化（和單數化）所有規則或不規則的字。當名稱由兩個以上的字組成時，Model名稱將會依循 Ruby 命名慣例，使用駝峰式命名，而資料表名稱則必須利用底線分隔。舉例來說：

* 資料表 - 複數形，由底線分隔多個單字 (e.g., `book_clubs`).
* Model 類別 - 單數形，第一個字大寫 (e.g., `BookClub`).

| Model / Class    | Table / Schema |
| ---------------- | -------------- |
| `Article`        | `articles`     |
| `LineItem`       | `line_items`   |
| `Deer`           | `deers`        |
| `Mouse`          | `mice`         |
| `Person`         | `people`       |


### 資料表綱要慣例

Active Record 會根據欄位用途來命名資料表中的欄位。
  
* **外鍵** - 此欄位依據單數形加上 _id 的形式命名，(如： `item_id`, `order_id`)。Active Record 會依`singularized_table_name_id` 形式的欄位名稱來建立 Model 之間的關聯。
* **主鍵** - Active Record 預設會使用一個名為 id 的整數欄位作為資料表的主鍵。當使用 [Active Record 遷移](active_record_migrations.html) 來建立資料表時，這個欄位就會自動生成。

下列是其他選擇性的欄位名稱，能增加更多不同的功能到 Active Record 中：

* `created_at` - 自動設定此欄位為首次建立此紀錄的時間與日期。
* `updated_at` - 自動設定此欄位為每次更新紀錄時的時間與日期。
* `lock_version` - 加入 [optimistic
  locking](http://api.rubyonrails.org/classes/ActiveRecord/Locking.html) 功能到 Model 中。
* `type` - 表示 Model 使用 [單表繼承](http://api.rubyonrails.org/classes/ActiveRecord/Base.html#class-ActiveRecord::Base-label-Single+table+inheritance) 的功能。
* `(association_name)_type` - 儲存
  [多態關聯](association_basics.html#polymorphic-associations) 的資料類型。
* `(table_name)_count` - 用來快取關聯物件的數量。 比如說，在 `Article` model 裡的 `comments_count` 就會為每篇文章快取 `Comment` 的數量。

NOTE: 雖然這些欄位名稱是選擇性的，但實際上是 Active Record 的保留字，除非需要使用到這些額外的功能，不然盡量不要使用這些保留字。比如說，`type` 是用來設計單表繼承資料表的。如果你沒有用到單表繼承資料表，可使用具相同功能的詞 "context" 代替，還是能準確的描述你所建置的模型。 

新增 Active Record Models
-----------------------------

新增 Active Record Model 非常簡單。只需要建立一個 ActiveRecord::Base 的子類別即可：

```ruby
class Product < ApplicationRecord
end
```

這會新增一個 `Product` Model，對應到資料庫中 `products` 的資料表。資料表中的每一列，皆會對應到 Model 實體的屬性。假設 products 以下面的 SQL 語句新建而成：

```sql
CREATE TABLE products (
   id int(11) NOT NULL auto_increment,
   name varchar(255),
   PRIMARY KEY  (id)
);
```

按照上述的資料表綱要，可以寫出如下的程式碼：

```ruby
p = Product.new
p.name = "Some Book"
puts p.name # "Some Book"
```

覆寫命名慣例
---------------------------------

要是需要不同於 Active Record 原有的命名慣例時怎麼辦？又或者 Rails 應用程式使用的資料來自舊的資料庫？沒問題，你也能輕易地覆寫預設的慣例。

可以使用 ActiveRecord::Base.table_name= 方法來指定對應的資料表名稱：

```ruby
class Product < ApplicationRecord
  self.table_name = "my_products"
end
```

如果修改了資料表名稱，在測試裡會需要使用 set_fixture_class 來手動定義 fixture 的類別名稱。

```ruby
class ProductTest < ActiveSupport::TestCase
  set_fixture_class my_products: Product
  fixtures :my_products
  ...
end
```

你也能使用 `ActiveRecord::Base.primary_key=` 方法覆寫資料表中的主鍵欄位。

```ruby
class Product < ApplicationRecord
  self.primary_key = "product_id"
end
```

CRUD: 讀寫資料
------------------------------

CRUD 是四種操作資料方法的簡稱：**C**reate, **R**ead, **U**pdate and **D**elete，分別是新增、讀取、更新與刪除。 Active Record 自動為應用程式新增處理資料表所需要的方法。

### 新增

Active Record 物件能從 hash 或區塊 (block) 中被建立，或者先建立之後在設定屬性也可以。 `new` 會回傳一個新的物件，而 `create` 會回傳一個物件後並存在資料庫中。


舉例來說，`User` 這個 Model 有 `name` 和 `occupation` 兩個屬性，使用 `create` 就能在資料庫裡新增並儲存一個新的紀錄，如下：

```ruby
user = User.create(name: "David", occupation: "Code Artist")
```

使用 `new` 這個方法，一個物件會被實體化，但不會被儲存，如下：

```ruby
user = User.new
user.name = "David"
user.occupation = "Code Artist"
```

呼叫 `user.save` 會將該筆紀錄存入資料庫裡。

最後，若是使用區塊的話，會將 User.new 實體化出來的物件放入區塊裡，對個別屬性作設定

```ruby
user = User.new do |u|
  u.name = "David"
  u.occupation = "Code Artist"
end
```

### 讀取

Active Record 提供了豐富的 API 來存取資料庫裡的資料。下列是 Active Record 提供的幾個讀取資料的方法：

```ruby
# return a collection with all users
users = User.all
```

```ruby
# return the first user
user = User.first
```

```ruby
# return the first user named David
david = User.find_by(name: 'David')
```

```ruby
# find all users named David who are Code Artists and sort by created_at in reverse chronological order
users = User.where(name: 'David', occupation: 'Code Artist').order(created_at: :desc)
```

你能在 [Active Record
Query Interface](active_record_querying.html) 中找到更多關於查詢 Active Record model 的內容。

### Update

一旦 Active Record 的物件被讀取出來了，可以修改它的屬性之後再存回資料庫裡。

```ruby
user = User.find_by(name: 'David')
user.name = 'Dave'
user.save
```

一個較簡易的方式是利用 hash 來對應要修改的屬性，如下：

```ruby
user = User.find_by(name: 'David')
user.update(name: 'Dave')
```

若要一次更新大量的資料屬性，上述的方法是最好的。若是要批量更新多筆記錄，可以使用類別方法：update_all：

```ruby
User.update_all "max_login_attempts = 3, must_change_password = 'true'"
```

### Delete

同樣地， Active Record 裡的物件也能從資料庫裡被移除。

```ruby
user = User.find_by(name: 'David')
user.destroy
```

驗證
-----------

Active Record 允許資料在存入資料庫前，先驗證資料的狀態。驗證資料有許多方法，
如：驗證一個屬性的值是不是空的、是不是單一的、是不是已經在資料庫裡或者是否依據特定的書寫格式....等等。

在我們將持久性資料寫入資料庫時，驗證是一個非常重要的問題，因為 `save` 與 `update` 會將驗證納入執行之中：當驗證失敗時，它會回傳 `false`，且不會對資料庫進行任何操作。上述三個方法皆有對應的 BANG 方法：save! 以及 update!，這比原本的方法更嚴格些，一旦失敗會直接拋出 ActiveRecord::RecordInvalid 的異常。用個簡單例子來說明：

```ruby
class User < ApplicationRecord
  validates :name, presence: true
end

user = User.new
user.save  # => false
user.save! # => ActiveRecord::RecordInvalid: Validation failed: Name can't be blank
```

你能在 [Active Record Validations
guide](active_record_validations.html) 中瞭解更多關於驗證的內容。

回呼（Callbacks）
---------

Active Record 回呼能允許你在 Model 生命週期裡對特定事件附加程式碼。這能讓你可以在特定事件發生時，執行特定的程式碼。比如說在資料庫中新增、更新或刪除等。了解更多關於回呼的內容，請參考：[Active Record Callbacks guide](active_record_callbacks.html)。

Migrations
----------

Rails 提供了管理資料庫綱要的 DSL，稱為「遷移」。遷移存在檔案裡，可以透過 `rake` 對所有支援 Active Record 的資料庫進行執行。以下是如何新建一張資料表：

```ruby
class CreatePublications < ActiveRecord::Migration[5.0]
  def change
    create_table :publications do |t|
      t.string :title
      t.text :description
      t.references :publication_type
      t.integer :publisher_id
      t.string :publisher_type
      t.boolean :single_issue

      t.timestamps
    end
    add_index :publications, :publication_type_id
  end
end
```

Rails 會持續追蹤所有提交到資料庫上的檔案，且提供回滾功能。要真正建立一張新的資料表，需要執行 `rails db:migrate` ，若是要回滾則執行 `rails db:rollback`。

注意以上的程式碼適用於各種資料庫，不論是 MySQL, PostgreSQL, Oracle 都可以。了解更多關於遷移的內容，請參考 [Active Record Migrations guide](active_record_migrations.html)。
