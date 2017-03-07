**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON http://guides.rubyonrails.org.**

Active Record 關聯
==========================

本篇介紹 Active Record 的關聯功能。

讀完本篇你將會知道：

* 如何宣告 Active Record Model 之間的關聯。
* 如何了解不同型式的 Active Record 關聯。
* 如何使用關聯加入的方法。

--------------------------------------------------------------------------------

為何需要關聯？
-----------------

在 Rails 中，兩個 Active Record Models 是透過 _關聯_ 來銜接。為什麼 Model 之間需要關聯呢？因為它們讓一般的程式碼運作起來更精簡且更簡單。舉例來說，一個簡單的 Rails 應用程式包含了作者的 Model 和書的 Model，一個作者可以寫了很多本書。要是沒有關聯， Model 的宣告可能會長這樣：

```ruby
class Author < ApplicationRecord
end

class Book < ApplicationRecord
end
```

現在，假設我們想要加入一本新的書到該作者底下，我們就需要做一些像這樣的事：

```ruby
@book = Book.create(published_at: Time.now, author_id: @author.id)
```
或者，我們考慮要刪掉一個作者，也要確保所有該作者的書也一併被刪除：

```ruby
@books = Book.where(author_id: @author.id)
@books.each do |book|
  book.destroy
end
@author.destroy
```
有了 Active Record 的關聯，我們可以使這些和其他的運作更有效率，就是利用宣告關聯性的方式告訴 Rails 這兩個 Model 是有關聯的。以下是修正過後設定作者與書的程式碼：

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :destroy
end

class Book < ApplicationRecord
  belongs_to :author
end
```
有了這樣的改變之後，新增一本書到特定的作者就變得非常容易：

```ruby
@book = @author.books.create(published_at: Time.now)
```
刪除該作者與他所有的書變得 *更加* 容易：

```ruby
@author.destroy
```
想知道更多關於其他類型的關聯，在本篇的下一個部分會介紹。然後會再介紹一些該如何使用和關聯有關的技巧，最後，會介紹一個完整的關聯方法和選項參考內容。

關聯的類型
-------------------------
Rails 提供六種類型的關聯：

* `belongs_to`
* `has_one`
* `has_many`
* `has_many :through`
* `has_one :through`
* `has_and_belongs_to_many`

關聯是透過宏風格（macro-style）的語法，即宣告的形式來加入功能到 Model。舉例來說，利用 `belongs_to` 宣告 Model 之間的隸屬關係，你指示 Rails 去維持 [主鍵](https://en.wikipedia.org/wiki/Unique_key)-[外鍵](https://en.wikipedia.org/wiki/Foreign_key) 兩個實體 Model 之間的資訊，同時也加了許多實用的方法到 Model 中。

在本篇中你會學到如何宣告與使用不同的關聯，首先，先介紹各種關聯的應用場景。

###`belongs_to` 關聯

`belongs_to` 關聯會建立兩個 Model 之間一對一的關係，宣告一個 Model 實體「屬於」另一個 Model 實體。舉例來說，如果你的應用程式包含作者與書，且每一本書都只能對應到一位作者，你可以這樣宣告書的 Model：

```ruby
class Book < ApplicationRecord
  belongs_to :author
end
```

![belongs_to Association Diagram](images/belongs_to.png)

NOTE: `belongs_to` 宣告 _必須_ 使用單數形式。上例若使用複數形式，會報 "uninitialized constant Book::Authors" 錯誤。這是因為 Rails 使用關聯名稱來推出類別名稱。如果關聯名稱錯用複數，推斷出來的類別名稱自然也錯了。 

上例對應的遷移看起來會像是：

```ruby
class CreateBooks < ActiveRecord::Migration[5.0]
  def change
    create_table :authors do |t|
      t.string :name
      t.timestamps
    end

    create_table :books do |t|
      t.belongs_to :author, index: true
      t.datetime :published_at
      t.timestamps
    end
  end
end
```

### `has_one` 關聯

`has_one` 關連也會建立兩個 Model 之間一對一的關係，但是在語意上（和因果關係）也有些不同。`has_one` 宣告一個 Model 實體，含有或持有另一個 Model 的實體。舉例來說，如果每一個供應商在你的應用程式中都只有一個帳號，你能這樣宣告你的供應商 Model：

```ruby
class Supplier < ApplicationRecord
  has_one :account
end
```

![has_one Association Diagram](images/has_one.png)

上例對應的遷移看起來會像是：

```ruby
class CreateSuppliers < ActiveRecord::Migration[5.0]
  def change
    create_table :suppliers do |t|
      t.string :name
      t.timestamps
    end

    create_table :accounts do |t|
      t.belongs_to :supplier, index: true
      t.string :account_number
      t.timestamps
    end
  end
end
```

根據不同的使用情景，可能會需要新增一個唯一 index 或新增一個在供應商欄位中的外鍵，來約束帳號資料表。在這個情況下，欄位就會如下定義：

```ruby
create_table :accounts do |t|
  t.belongs_to :supplier, index: true, unique: true, foreign_key: true
  # ...
end
```


###`has_many` 關聯

`has_many` 關聯表示了兩個 Model 之間一對多的關係。常 `has_many` 另一邊對應的是 `belongs_to` 關聯。`has_many` 關聯宣告一個 Model 實體，有零個或多個另一個 Model 的實體。舉例來說，應用程式有作者與書兩個 Model，作者可有多本書，作者 Model 便如此宣告：

```ruby
class Author < ApplicationRecord
  has_many :books
end
```

NOTE: 宣告 `has_many` 關聯名稱採複數，如上。

![has_many Association Diagram](images/has_many.png)


上例對應的遷移看起來會像是：

```ruby
class CreateAuthors < ActiveRecord::Migration[5.0]
  def change
    create_table :authors do |t|
      t.string :name
      t.timestamps
    end

    create_table :books do |t|
      t.belongs_to :author, index: true
      t.datetime :published_at
      t.timestamps
    end
  end
end
```

###`has_many :through` 關聯

`has_many :through` 關聯通常用來建立兩個 Model 之間的多對多的關係。`has_many :through` 關聯透過（through）第三個 Model，宣告一個 Model 實體，可有零個或多個另一個 Model 的實體。舉個醫療的例子來說，「病患」需要透過「預約」來見「物理治療師」。相對應的宣告如下：

```ruby
class Physician < ApplicationRecord
  has_many :appointments
  has_many :patients, through: :appointments
end

class Appointment < ApplicationRecord
  belongs_to :physician
  belongs_to :patient
end

class Patient < ApplicationRecord
  has_many :appointments
  has_many :physicians, through: :appointments
end
```

![has_many :through Association Diagram](images/has_many_through.png)

上例對應的遷移看起來會像是：

```ruby
class CreateAppointments < ActiveRecord::Migration[5.0]
  def change
    create_table :physicians do |t|
      t.string :name
      t.timestamps
    end

    create_table :patients do |t|
      t.string :name
      t.timestamps
    end

    create_table :appointments do |t|
      t.belongs_to :physician, index: true
      t.belongs_to :patient, index: true
      t.datetime :appointment_date
      t.timestamps
    end
  end
end
```

連接 Model（Join Model）的集合可以透過 [`has_many` 關聯方法](#has-many-association-reference) 來管理。比如：

```ruby
physician.patients = patients
```

新建立的關聯物件會自動生成連接 Model (Join Model)，如果刪除了其中一個物件，也會刪除該連接欄位在資料庫中的記錄。

WARNING: 連接 Model 會直接自動刪除，不會觸發 destroy 回呼。

`has_many :through` 關聯用來簡化嵌套 `has_many` 也非常有幫助。舉例來說，如果一個文件中有許多章節、段落，想要從文件取得所有段落的集合。你可以這樣做：

```ruby
class Document < ApplicationRecord
  has_many :sections
  has_many :paragraphs, through: :sections
end

class Section < ApplicationRecord
  belongs_to :document
  has_many :paragraphs
end

class Paragraph < ApplicationRecord
  belongs_to :section
end
```

當你標明了 `through: :sections`， Rails 就會了解：

```ruby
@document.paragraphs
```

###`has_one :through` 關聯

`has_one :through` 關聯通常用來建立兩個 Model 之間的ㄧ對ㄧ的關係。`has_one :through` 關聯透過（through）第三個 Model，宣告一個 Model 實體可有另一個 Model 的實體。舉例來說，供應商有一個帳號，每個帳號有該帳號的歷史，相對應的宣告如下：

```ruby
class Supplier < ApplicationRecord
  has_one :account
  has_one :account_history, through: :account
end

class Account < ApplicationRecord
  belongs_to :supplier
  has_one :account_history
end

class AccountHistory < ApplicationRecord
  belongs_to :account
end
```

![has_one :through Association Diagram](images/has_one_through.png)


上例對應的遷移看起來會像是：

```ruby
class CreateAccountHistories < ActiveRecord::Migration[5.0]
  def change
    create_table :suppliers do |t|
      t.string :name
      t.timestamps
    end

    create_table :accounts do |t|
      t.belongs_to :supplier, index: true
      t.string :account_number
      t.timestamps
    end

    create_table :account_histories do |t|
      t.belongs_to :account, index: true
      t.integer :credit_rating
      t.timestamps
    end
  end
end
```

###`has_and_belongs_to_many` 關聯

`has_and_belongs_to_many` 關聯建立兩個 Model 之間直接的多對多關係，且沒有另外介入的 Model。舉例來說，應用程式有許多組件（Assembly），組件下有許多部件（Part），可以如此宣告：

```ruby
class Assembly < ApplicationRecord
  has_and_belongs_to_many :parts
end

class Part < ApplicationRecord
  has_and_belongs_to_many :assemblies
end
```

![has_and_belongs_to_many Association Diagram](images/habtm.png)

上例對應的遷移看起來會像是：

```ruby
class CreateAssembliesAndParts < ActiveRecord::Migration[5.0]
  def change
    create_table :assemblies do |t|
      t.string :name
      t.timestamps
    end

    create_table :parts do |t|
      t.string :part_number
      t.timestamps
    end

    create_table :assemblies_parts, id: false do |t|
      t.belongs_to :assembly, index: true
      t.belongs_to :part, index: true
    end
  end
end
```

###`belongs_to` 和 `has_one` 的應用

如果想建立兩個 Model 之間的一對一關係，一邊宣告 `belongs_to`，另一邊宣告 `has_one`。怎麼知道哪個 Model 要寫哪個？

差異在於外鍵該放在哪個 Model 中（外鍵放在宣告 `belongs_to` 關聯的資料表）。但實際上應該要考慮真實的語義。比如 `has_one` 關聯表示某物屬於你，也就是供應商有一個帳號，比帳號擁有供應商合理。所以正確的關聯應這麼宣告：

```ruby
class Supplier < ApplicationRecord
  has_one :account
end

class Account < ApplicationRecord
  belongs_to :supplier
end
```
上例對應的遷移看起來會像是：

```ruby
class CreateSuppliers < ActiveRecord::Migration[5.0]
  def change
    create_table :suppliers do |t|
      t.string  :name
      t.timestamps
    end

    create_table :accounts do |t|
      t.integer :supplier_id
      t.string  :account_number
      t.timestamps
    end

    add_index :accounts, :supplier_id
  end
end
```

NOTE: 使用 `t.integer :supplier_id` 可以讓外鍵的命名更顯著且明確。在現在 Rails 的版本中，你可以使用 `t.references :supplier` 抽象掉實作細節。

### `has_many :through` 和 `has_and_belongs_to_many` 的應用

Rails 提供了兩種不同宣告 Model 之間多對多關係的方法。比較簡單的方法是透過 `has_and_belongs_to_many` ，他可以讓你更直接地建立關聯：

```ruby
class Assembly < ApplicationRecord
  has_and_belongs_to_many :parts
end

class Part < ApplicationRecord
  has_and_belongs_to_many :assemblies
end
```

第二種方法是 `has_many :through` 來宣告多對多的關係，這則是「透過」連接 Model 建立關係，屬較不直接的方式：

```ruby
class Assembly < ApplicationRecord
  has_many :manifests
  has_many :parts, through: :manifests
end

class Manifest < ApplicationRecord
  belongs_to :assembly
  belongs_to :part
end

class Part < ApplicationRecord
  has_many :manifests
  has_many :assemblies, through: :manifests
end
```
根據經驗法則來看，當多對多關係中間的 Model 要獨立使用時，使用 `has_many :through`。若不需要多對多關係中間的 Model 做任何事時，保持簡單使用 `has_and_belongs_to_many`（但要記得在資料庫中建立連接資料表）。


當在連接 Model 中需要用到驗證、回呼或者其他屬性時，使用 `has_many :through` 較好。

### 多型關聯

一個較進階的關聯使用技巧是 _多型關聯_ 。有了多型關聯，在單個關聯裡，一個 Model 可屬於多個 Model。比如說，一個圖片 Model 可屬於員工 Model 或產品 Model。如下為宣告的範例：

```ruby
class Picture < ApplicationRecord
  belongs_to :imageable, polymorphic: true
end

class Employee < ApplicationRecord
  has_many :pictures, as: :imageable
end

class Product < ApplicationRecord
  has_many :pictures, as: :imageable
end
```

你可能可以想到一個多型 `belongs_to` 宣告會建立一個介面是所有的 Model 都可以用的，就前面 `Employee` Model 的例子而言，你可以透過 @employee.pictures 來取出所有圖片。

同樣地，你也能透過 `@product.pictures` 取得照片。

如果你有一個 `Picture` Model 的實體，你可以透過 `@picture.imageable` 得到他的父物件。但是首先必須先宣告一個外鍵（ \_id）欄位與類型（ \_type）欄位在這個 Model 中，用來宣告此 Model 擁有多型介面：

```ruby
class CreatePictures < ActiveRecord::Migration[5.0]
  def change
    create_table :pictures do |t|
      t.string  :name
      t.integer :imageable_id
      t.string  :imageable_type
      t.timestamps
    end

    add_index :pictures, [:imageable_type, :imageable_id]
  end
end
```

上例遷移可用 `t.references` 形式簡化：

```ruby
class CreatePictures < ActiveRecord::Migration[5.0]
  def change
    create_table :pictures do |t|
      t.string :name
      t.references :imageable, polymorphic: true, index: true
      t.timestamps
    end
  end
end
```

![Polymorphic Association Diagram](images/polymorphic.png)

###自連接

在設計一個資料 Model 時，你可能會發現一個 Model 有時會需要與自己有關係，比如說：你可能需要將所有的員工資料存在一張資料表裡，但又要能追蹤像是經理或下屬之間的關係。這種情況可以使用自連接（Self join）關聯：

```ruby
class Employee < ApplicationRecord
  has_many :subordinates, class_name: "Employee",
                          foreign_key: "manager_id"

  belongs_to :manager, class_name: "Employee"
end
```
這麼一來，就可以使用 @employee.subordinates 與 @employee.manager 來取出經理與下屬。

在遷移裡則是需要加入參照該 Model 本身的欄位：

```ruby
class CreateEmployees < ActiveRecord::Migration[5.0]
  def change
    create_table :employees do |t|
      t.references :manager, index: true
      t.timestamps
    end
  end
end
```

秘訣、技巧與注意事項
--------------------------

如果想要有效率地在你的 Rails 應用程式裡使用 Active Record 關聯性的話，這裡有一些事情你應該知道：

* 控制快取
* 避免命名衝突
* 更新資料庫綱要（schema）
* 控制關聯作用域（scope）
* 雙向關聯

### 控制快取

這裡所有的關聯性關係皆圍繞著快取打轉，這些方法會保留最近的查詢結果，以便之後查詢使用。快取甚至在方法之間共享，比如：

```ruby
author.books                 # 從資料庫中取出書
author.books.size            # 使用書的快取版本
author.books.empty?          # 使用書的快取版本
```

但要是應用程式的某部分更新了資料，因此想要重載快取呢？只需要在關聯方法中呼叫 `reload` 即可。

```ruby
author.books                 # 從資料庫中取出書
author.books.size            # 使用書的快取版本
author.books.reload.empty?    # 刪除書的快取版本
                                # 回到資料庫中
```

### 避免命名衝突

你並不能任意地為關聯性命名，因為在建立關聯時，會新增一個與關聯性名稱相同的方法到 Model 中，要是關聯性名稱與 `ActiveRecord::Base` 的實體方法相同的話，關聯性成稱會覆蓋掉`ActiveRecord::Base` 實體方法的名稱。例如  `attributes` 或 `connection` 就是不好的關聯名稱。

### 更新資料庫綱要

關聯性非常有幫助，但是他們並非魔法，維持資料庫綱並保持與關聯性的對應關係是開發者的責任。就 `belongs_to` 關聯而言，你需要新增外鍵，而 `has_and_belongs_to_many` 關聯你則是需要新增一個相對應的連接資料表。

#### 為 `belongs_to` 關聯建立外鍵

當你宣告一個 `belongs_to` 關聯時，你需要建立一個適當的外鍵，比如說下面這個 Model:

```ruby
class Book < ApplicationRecord
  belongs_to :author
end
```

這個宣告必須在書的資料表中有對應的外鍵宣告：

```ruby
class CreateBooks < ActiveRecord::Migration[5.0]
  def change
    create_table :books do |t|
      t.datetime :published_at
      t.string   :book_number
      t.integer  :author_id
    end

    add_index :books, :author_id
  end
end
```

如果在建立 Model 之後才宣告關聯，記得使用 `add_column` 遷移，來提供所需的外鍵。

#### 為 `has_and_belongs_to_many` 關聯建立連接資料表

如果你建立了 `has_and_belongs_to_many` 關聯，你需要非常明確地建立連接資料表，除非這個連接資料表是已經標示使用了`:join_table` 選項， 否則 Active Record 會以關聯的類別名稱，依照詞法先後順序來命名這張連接資料表。假設有 author 與 book 兩個 Model，則預設的連接表名稱會是 "authors_books"，因為在詞法順序當中，a 的地位高於 b。

WARNING: Model 名稱的優先順序是使用 `String` 的 `<=>` 來計算。這表示若字串不一樣長，比較最短長度時，兩個字串是相等的。但長字串的詞法順序則先於短字串。舉例來說，你可能認為 "paper\_boxes" 與 "papers" 這兩個資料表產生的連接表名稱是 "papers\_paper\_boxes"，因為 "paper\_boxes" 比 papers 長。但實際上是 "paper\_boxes\_papers"，因為在常見的編碼裡，'s'的詞法順序優於底線 '\_' 。


不論名字為何，必須要在適當的遷移中，手動產生連接表。比如下面的關聯範例：

```ruby
class Assembly < ApplicationRecord
  has_and_belongs_to_many :parts
end

class Part < ApplicationRecord
  has_and_belongs_to_many :assemblies
end
```

關聯要有效，必須寫一個遷移來建立 `assemblies_parts` 資料表。而且此表無主鍵：

```ruby
class CreateAssembliesPartsJoinTable < ActiveRecord::Migration[5.0]
  def change
    create_table :assemblies_parts, id: false do |t|
      t.integer :assembly_id
      t.integer :part_id
    end

    add_index :assemblies_parts, :assembly_id
    add_index :assemblies_parts, :part_id
  end
end
```

我們會傳入 `id: false` 到 `create_table` 之中是因為資料表並不代表一個 Model。而這就需要依靠關聯來運作，假使你在 `has_and_belongs_to_many` 關聯中，檢查到一個不正常的行為，如 ID 錯位、ID 衝突，很可能就是因為忘記去掉主鍵。

你也能使用 `create_join_table` 這個方法。

```ruby
class CreateAssembliesPartsJoinTable < ActiveRecord::Migration[5.0]
  def change
    create_join_table :assemblies, :parts do |t|
      t.index :assembly_id
      t.index :part_id
    end
  end
end
```

### 控制關聯作用域

在預設的情況下，關聯只會在目前模組的作用域裡尋找物件。這在模組裡宣告 Active Record Model 時很重要，比如：

```ruby
module MyApplication
  module Business
    class Supplier < ApplicationRecord
       has_one :account
    end

    class Account < ApplicationRecord
       belongs_to :supplier
    end
  end
end
```

這並不會有什麼問題，因為 `Supplier` 和 `Account` 這兩個類別都是在同一個作用域中被定義。但是下一個例子中的就 _不會_ 正常運作，因為 `Supplier` 和 `Account` 是在不同的工作域中被定義：

```ruby
module MyApplication
  module Business
    class Supplier < ApplicationRecord
       has_one :account
    end
  end

  module Billing
    class Account < ApplicationRecord
       belongs_to :supplier
    end
  end
end
```
要使在不同工作域被命名的 Model 和另一個 Model 有關聯時，就會需要明確地在宣告關聯時，標示完整的類別名稱：

```ruby
module MyApplication
  module Business
    class Supplier < ApplicationRecord
       has_one :account,
        class_name: "MyApplication::Billing::Account"
    end
  end

  module Billing
    class Account < ApplicationRecord
       belongs_to :supplier,
        class_name: "MyApplication::Business::Supplier"
    end
  end
end
```

### 雙向關聯

關聯雙向運作是很常見的需求，這需要在兩邊都宣告：

```ruby
class Author < ApplicationRecord
  has_many :books
end

class Book < ApplicationRecord
  belongs_to :author
end
```
在預設的情況下，Active Reocrd 並不知道這些關聯的連結性，這可能會導致複製物件的不同步：

```ruby
a = Author.first
b = a.books.first
a.first_name == b.author.first_name # => true
a.first_name = 'Manny'
a.first_name == b.author.first_name # => false
```
這會發生的原因是 `a` 和 `b.author` 在記憶體中是相同的資料兩種不同表示法，且改了一個並不會自動改另一個。 Active Record 提供了 `:inverse_of` 選項用來通知 Rails 關聯之間的關係：

```ruby
class Author < ApplicationRecord
  has_many :books, inverse_of: :author
end

class Book < ApplicationRecord
  belongs_to :author, inverse_of: :books
end
```
有了這些改變， Active Record 只會從作者物件中載入一個，避免不一致性並讓應用程式更有效率。

```ruby
a = Author.first
b = a.books.first
a.first_name == b.author.first_name # => true
a.first_name = 'Manny'
a.first_name == b.author.first_name # => true
```
這裡有一些關於使用 `inverse_of` 輔助的限制：

* 他們無法與 `:through` 關聯聯用。
* 他們無法與 `:polymorphic` 關聯聯用。
* 他們無法與 `:as` 關聯聯用。
* 對於 `belongs_to` 關聯，會忽略 `has_many` 所設定的 `inverse_of`。

每個關聯都會自動尋找反向的關聯然後設定 `:inverse_of` 選項(根據他們相關連的名字)。 大部分標準的關聯性名字都能被這麼做。然而，要是關聯性含有以下的選項名的話，就不會正確的自動生成反向：

* `:conditions`
* `:through`
* `:polymorphic`
* `:foreign_key`

關聯完整參考手冊
------------------------------
以下小節將完整地給出每種關聯的細節，包含關聯新增的方法、宣告時可用的選項。

### `belongs_to` 關聯參考

`belongs_to` 關連選項會建立與另一個 Model 一對一的關係。以資料庫的術語來說，此類別包含外鍵，如有另一個類別含有此外鍵，那麼你就應該使用 `has_one`。

#### 由 `belongs_to` 增加的方法

當你宣告一個 `belongs_to` 關聯，宣告的類別就會增加五個關聯方法：

* `association`
* `association=(associate)`
* `build_association(attributes = {})`
* `create_association(attributes = {})`
* `create_association!(attributes = {})`

在這些方法中 `association` 會被換成第一個傳給 `belongs_to` 的參數 symbol。舉例來說，如下的宣告：

```ruby
class Book < ApplicationRecord
  belongs_to :author
end
```
每一個 `Book` Model 實體都會有以下的方法：

```ruby
author
author=
build_author
create_author
create_author!
```
NOTE: 當初始化一個新的 `has_one` 或 `belongs_to` 關聯時你必須使用 `build_` 字首建立這個關聯，而不是使用 `association.build` 方法，因為該方法用在 `has_many` 或 `has_and_belongs_to_many` 關聯上。要新建並儲存，則使用 `create_` 字首。

##### `association`

`association` 方法會回傳一個關聯物件（如果有的話），若沒有找到關聯物件，則會回傳 `nil`。

```ruby
@author = @book.author
```
如果關聯物件已經從資料庫中取出，則會回傳此物件的快取版本，若要強制重新從資料庫中讀取，就要在父層物件傳入 `#reload`。

```ruby
@author = @book.reload.author
```

##### `association=(associate)`

`association=` 方法是指定一個關聯物件至這個物件中，背後的原理是，將主鍵從此物件取出然後將之設為關聯物件的外鍵。

```ruby
@book.author = @author
```

##### `build_association(attributes = {})`

`build_association` 方法會回傳一個新的關聯類型物件，這個物件透過傳入的屬性被實體化，同時會自動設定外鍵，但關聯物件並 _不會_ 被儲存至資料庫中。

```ruby
@author = @book.build_author(author_number: 123,
                                  author_name: "John Doe")
```

##### `create_association(attributes = {})`

`create_association(attributes = {})` 方法回傳一個新的物件，這個物件透過傳入的屬性被實體化，同時會自動設定外鍵，一旦通過所有 Model 的驗證規則時，便 _會_ 把此關聯物件存入資料庫。

```ruby
@author = @book.create_author(author_number: 123,
                                   author_name: "John Doe")
```

##### `create_association!(attributes = {})`

功能和上述的 `create_association` 相同，但是在驗證無效時會提出 `ActiveRecord::RecordInvalid`。


#### Options for `belongs_to`

Rails 聰明的預設值足以應付多數場景，但也有需要自訂 `belongs_to` 關聯的時候，這些自訂的關聯很容易能被建造出來，只要在建立關聯時傳入選項和工作區塊，便能達成，舉例來說：這些關聯有兩種選項：

```ruby
class Book < ApplicationRecord
  belongs_to :author, dependent: :destroy,
    counter_cache: true
end
```

The `belongs_to` association supports these options:

`belongs_to` 關聯支援以下的選項：

* `:autosave`
* `:class_name`
* `:counter_cache`
* `:dependent`
* `:foreign_key`
* `:primary_key`
* `:inverse_of`
* `:polymorphic`
* `:touch`
* `:validate`
* `:optional`

##### `:autosave`

你果你將 `:autosave` 的選項設為 `true`，Rails 會在儲存父物件時，自動保存子物件。若子物件標記為刪除，也會在儲存時自動刪除。

##### `:class_name`

如果另一個 Model 名稱並不能從關聯名中推論出來，你可以使用 `:class_name` 選項去輔助該 Model 的名字。舉例來說，如果一本書只屬於一個作者，但是事實上這個包含作者名的 Model 叫：`Patron`，你就能這樣設定：

```ruby
class Book < ApplicationRecord
  belongs_to :author, class_name: "Patron"
end
```

##### `:counter_cache`

`:counter_cache` 選項可以使尋找隸屬物件更有效率，就下面的例子而言：

```ruby
class Book < ApplicationRecord
  belongs_to :author
end
class Author < ApplicationRecord
  has_many :books
end
```

有了上面的宣告後，詢問 `@author.books.size` 值時，則對資料庫下一條 `COUNT(*)` 查詢指令。要避免此操作，可以加上一個 counter_cache 到 _被隸屬的_ Model 中:


```ruby
class Book < ApplicationRecord
  belongs_to :author, counter_cache: true
end
class Author < ApplicationRecord
  has_many :books
end
```

有了這個宣告， Rails 會保持這個快取是最新的，然後回傳這個值到 `size` 值中。

即使 `:counter_cache` 選項是在 `belongs_to` Model 中指定，但是真實的欄位必須被加入  _associated_ (`has_many`) Model。就上述的例子而言，你需要增加一個叫做 `books_count` 的欄位到 `Author` Model 中。

你可以在 `counter_cache` 宣告中客製欄位名稱，覆蓋掉預設的欄位名。舉例來說，用 `count_of_books` 而不是 `books_count`：

```ruby
class Book < ApplicationRecord
  belongs_to :author, counter_cache: :count_of_books
end
class Author < ApplicationRecord
  has_many :books
end
```
NOTE: 你只需要在關聯中的 `belongs_to` 邊，指定 `counter_cache` 的內容。

Counter Cache 欄位透過 `attr_readonly` 被加到關聯模型的唯讀列表裡。

##### `:dependent`

當關聯物件擁有者被刪除之後，如何處理該關聯物件：

* `:destroy` 讓關聯物件也同時被刪除。
* `:delete_all` 讓關聯物件直接從資料庫中被刪除（回呼不會被執行）。
* `:nullify` 讓外鍵被射成 `NULL` （回呼不會被執行）。
* `:restrict_with_exception` 如果有ㄧ關聯記錄，就會對擁有者拋出異常。
* `:restrict_with_error` 如果有一關聯物件，錯誤就會被加入到擁有者中。

WARNING: 不應該在和 `has_many` 相連結的 `belongs_to` 關聯中使用這個選項，會導致資料庫產生孤兒現象。


##### `:foreign_key`

照慣例而言，Rails 假定連接資料表的外鍵名稱為 Model 名稱加上 `_id`。然而，`:foreign_key` 選項能讓你直接設定外鍵的名字：

```ruby
class Book < ApplicationRecord
  belongs_to :author, class_name: "Patron",
                        foreign_key: "patron_id"
end
```
TIP: 在大多數的情況下，Rails 並不會新增一個外鍵的欄位給你，你必須特別地在遷移中標記。


##### `:primary_key`

按照慣例，Rails 是以 `id` 欄位放置主鍵的資料表。`:primary_key` 選項可以讓你具體指定一個不一樣的欄位。

舉例來說，現在我們擁有一個 `users` 的資料表而 `guid` 為該資料表的主鍵。如果我們希望一個獨立出來的 `todos` 資料表中，`guid` 欄位以 `user_id` 為外鍵，我們就可以利用 `primary_key` 達到這樣的效果：

```ruby
class User < ApplicationRecord
  self.primary_key = 'guid' # primary key is guid and not id
end

class Todo < ApplicationRecord
  belongs_to :user, primary_key: 'guid'
end
```

當我們執行 `@user.todos.create` 時， `@todo` 記錄就會有其 `user_id` 的值，如同 `@user` 中的 `guid` 值。

##### `:inverse_of`

`:inverse_of` 選項具體指明了 `has_many` 和 `has_one` 關聯中另一端（反向）的名字。此選項無法與 `:polymorphic` 選項連用。

```ruby
class Author < ApplicationRecord
  has_many :books, inverse_of: :author
end

class Book < ApplicationRecord
  belongs_to :author, inverse_of: :books
end
```

##### `:polymorphic`

當傳入 `true` 的選項到 `:polymorphic` 中時表示這是一個多型關聯。多型關聯在 <a href="#polymorphic-associations">前面我們已經介紹過</a>。

##### `:touch`

如果將 `:touch` 選項設為 `true`，那麼 `updated_at` 或者 `updated_on` 的時間戳記會自動更新成現在物件被儲存或刪除的時間：

```ruby
class Book < ApplicationRecord
  belongs_to :author, touch: true
end

class Author < ApplicationRecord
  has_many :books
end
```

在這個情況下，儲存或刪除一本都都會更新作者關聯項的時間戳記。你也能指定一個特定的時間戳屬性來更新：


```ruby
class Book < ApplicationRecord
  belongs_to :author, touch: :books_updated_at
end
```

##### `:validate`

如果你設定 `:validate` 選項為 `true` 的話，那麼不管何時進行儲存，該關聯物件都會被驗證。預設的情況下，這是設定 `false`：在儲存的時候，關聯物件並不會被驗證。

##### `:optional`

如果你設定 `:optional` 的選項為 `true`，那麼物件的存在性就不會被驗證。在預設的情況下，這個選項是設定為 `false`。

#### `belongs_to` 工作域

有時你可能希望能客製化 `belongs_to` 使用的查詢語句。可以透過傳入作用域區塊 (scope block) 來達到，比如：

```ruby
class Book < ApplicationRecord
  belongs_to :author, -> { where active: true },
                        dependent: :destroy
end
```

你可以在工作域區塊使用任何一種 [查詢方法](active_record_querying.html)。下面我們將會討論幾個：

* `where`
* `includes`
* `readonly`
* `select`

##### `where`

`where` 方法讓你指定關聯物件必須達到的條件。

```ruby
class book < ApplicationRecord
  belongs_to :author, -> { where active: true }
end
```

##### `includes`

你可以使用 `includes` 方法用來指定需要 Eager Loading 的第二層關聯。看看下面這個例子：

```ruby
class LineItem < ApplicationRecord
  belongs_to :book
end

class Book < ApplicationRecord
  belongs_to :author
  has_many :line_items
end

class Author < ApplicationRecord
  has_many :books
end
```
如果你很頻繁地直接從 line items (`@line_item.book.author`) 中取出作者，那你應該在 line items 關聯中含有作者讓你的程式碼更有效率。

```ruby
class LineItem < ApplicationRecord
  belongs_to :book, -> { includes :author }
end

class Book < ApplicationRecord
  belongs_to :author
  has_many :line_items
end

class Author < ApplicationRecord
  has_many :books
end
```
NOTE: 立即的關聯不需要使用 `includes`，比如 `Book belongs_to :author`，預設則會 Eager Loading 作者。

##### `readonly`

若使用 `readonly` 則關聯物件取出時為唯讀。

##### `select`

`select` 方法讓你覆寫用來讀取資料關聯物件的 SQL `SELECT` 語句。 Rails 預設是讀取所有的欄位。

TIP: 如果在 `belongs_to` 關聯上使用 `select` 方法，你也同時要設定 `:foreign_key` 選在確保正確的結果。

#### 檢查關聯物件是否存在？

你可以利用 `association.nil?` 方法檢查關聯物件是否存在。

```ruby
if @book.author.nil?
  @msg = "No author found for this book"
end
```

#### 物件何時被儲存？

指定一個物件到 `belongs_to` 關聯並 _不_ 會自動儲存該物件。也不會儲存該關聯物件。  

### `has_one` 關聯參考

`has_one` 關聯建立兩個 Model 之間的一對一關係。在資料庫的術語是，這個關聯並沒有外鍵，如果這個類別包含外鍵，那你應該使用 `belongs_to`。

#### `has_one` 關連新增的方法

當你宣告一個 `has_one` 關聯時，這個被宣告的類別會自動增加五種方法：

* `association`
* `association=(associate)`
* `build_association(attributes = {})`
* `create_association(attributes = {})`
* `create_association!(attributes = {})`

在所有的方法中，`association` 會換成作為第一個傳給 `has_one` 的參數符號。比如：

```ruby
class Supplier < ApplicationRecord
  has_one :account
end
```
現在每個 `Supplier` Model 的實體都會有這些方法：

```ruby
account
account=
build_account
create_account
create_account!
```
NOTE: 當初始化一個新的 `has_one` 或 `belongs_to` 關聯時，你必須使用 `build_` 字首來建立關聯，而不是使用 `association.build` 方法，因為那是用來建立 `has_many` 或 `has_and_belongs_to_many` 關聯的。要建立且存進資料庫，則使用 `create_` 字首。

##### `association`

`association` 方法會回傳一個關聯物件（如果有的話），若沒有找到關聯物件，則會回傳 `nil`。

```ruby
@account = @supplier.account
```
如果關聯物件已經從資料庫中取出，則會回傳此物件的快取版本，若要強制重新從資料庫中讀取，就要在父層物件傳入 `#reload`。

```ruby
@account = @supplier.reload.account
```

##### `association=(associate)`

`association=` 方法指定關聯物件到這個物件中，背後的做法是，將主鍵從此物件取出然後將之設為關聯物件的外鍵。

```ruby
@supplier.account = @account
```

##### `build_association(attributes = {})`

`build_association` 方法會回傳一個新的關聯類型物件，這個物件透過傳入的屬性被實體化，同時會自動設定外鍵，但關聯物件並 _不會_ 被儲存至資料庫中。

```ruby
@account = @supplier.build_account(terms: "Net 30")
```

##### `create_association(attributes = {})`

`create_association` 方法回傳一個關聯類型的新物件，這個物件透過傳入的屬性被實體化，同時會自動設定外鍵，一旦通過所有 Model 的驗證規則時，便 _會_ 把此關聯物件存入資料庫。

```ruby
@account = @supplier.create_account(terms: "Net 30")
```

##### `create_association!(attributes = {})`

與 `create_association` 方法相同，但在驗證失敗時會提出 `ActiveRecord::RecordInvalid`。


#### Options for `has_one`

Rails 聰明的預設值足以應付多數場景，但也有需要自訂 `has_one` 關聯的時候，這些自訂的關聯很容易能被建造出來，只要在建立關聯時傳入選項和工作區塊，便能達成，舉例來說，這些關聯有兩種選項：

```ruby
class Supplier < ApplicationRecord
  has_one :account, class_name: "Billing", dependent: :nullify
end
```

`has_one` 關聯支援以下的選項：

* `:as`
* `:autosave`
* `:class_name`
* `:dependent`
* `:foreign_key`
* `:inverse_of`
* `:primary_key`
* `:source`
* `:source_type`
* `:through`
* `:validate`

##### `:as`

設定 `:as` 選項表示這是一個多型關聯，在[本篇前面的小節](#polymorphic-associations)已經討論過。

##### `:autosave`

如果你將 `:autosave` 的選項設為 `true`，Rails 會在儲存父物件時，自動保存子物件。如子物件標記為刪除，也會在儲存時自動刪除。

##### `:class_name`

如果關聯 Model 的名稱推論不出來的時，你可以使用 `:class_name` 選項來指定名稱。舉例來說，如果一個供應商擁有一個帳戶，但是實際上含有帳戶的 Model 名叫做 `Billing`，你能這麼設定：

```ruby
class Supplier < ApplicationRecord
  has_one :account, class_name: "Billing"
end
```

##### `:dependent`

在刪除關聯物件擁有者時該如何處理關聯物件：

* `:destroy` 讓關聯物件也同時被刪除。
* `:delete_all` 讓關聯物件直接從資料庫中被刪除（回呼不會被執行）。
* `:nullify` 讓外鍵被射成 `NULL` （回呼不會被執行）。
* `:restrict_with_exception` 如果有一關聯記錄，就會對擁有者拋出異常。
* `:restrict_with_error` 如果有一關聯物件，錯誤就會被加入到擁有者中。

如果資料庫約束設定 `NOT NULL`，則不要使用 `:nullify` 選項。如果你沒有設定 `dependent` 去刪除這些關聯，會沒有辦法改變這些關聯物件，因為這些關聯物件在初始化時就設定允許 `NULL` 值。

##### `:foreign_key`

照慣例而言，Rails 假定連接資料表的外鍵名稱為 Model 名稱加上 `_id`。然而，`:foreign_key` 選項能讓你直接設定外鍵的名字：

```ruby
class Supplier < ApplicationRecord
  has_one :account, foreign_key: "supp_id"
end
```
TIP: 在大多數的情況下，Rails 並不會新增一個外鍵的欄位給你，你必須特別地在遷移中標記。


##### `:inverse_of`

`:inverse_of` 選項具體指明了 `belongs_to` 關聯中另一端的名字。此選項無法與 `:through` 或 `:as` 選項連用。

```ruby
class Supplier < ApplicationRecord
  has_one :account, inverse_of: :supplier
end

class Account < ApplicationRecord
  belongs_to :supplier, inverse_of: :account
end
```

##### `:primary_key`

按照慣例，Rails 是以 `id` 欄位放置主鍵的資料表。`:primary_key` 選項可以讓你具體指定一個不一樣的欄位。

##### `:source`

`:source` 選項標明了 `has_one :through` 關聯的來源關聯名稱。

##### `:source_type`

`:source_type` 選項標明了透過多型關聯的 `has_one :through` 關聯的來源類型。

##### `:through`

`:through` 選項用來指定下查詢的連接 Model。`has_one :through` 關聯在[前面已詳細介紹過](#the-has-one-through-association)。

##### `:validate`

如果你設定 `:validate` 選項為 `true` 的話，那麼不管何時進行儲存，該關聯物件都會被驗證。預設的情況下，這是設定 `false`：在儲存的時候，關聯物件並不會被驗證。

#### `has_one` 工作域

有時你可能希望能客製化 `has_one` 使用的查詢語句。可以透過傳入作用域區塊 (scope block) 來達到，比如：


```ruby
class Supplier < ApplicationRecord
  has_one :account, -> { where active: true }
end
```

你可以在工作域區塊使用任何一種 [查詢方法](active_record_querying.html)。下面我們將會討論幾個：

* `where`
* `includes`
* `readonly`
* `select`

##### `where`

`where` 方法讓你指定關聯物件必須達到的條件。

```ruby
class Supplier < ApplicationRecord
  has_one :account, -> { where "confirmed = 1" }
end
```

##### `includes`

你可以使用 `includes` 方法用來指定需要 Eager Loading 的第二層關聯。看看下面這個例子：

```ruby
class Supplier < ApplicationRecord
  has_one :account
end

class Account < ApplicationRecord
  belongs_to :supplier
  belongs_to :representative
end

class Representative < ApplicationRecord
  has_many :accounts
end
```
如果你很頻繁地直接從 suppliers (`@supplier.account.representative`) 中取出 representatives，那你應該在 suppliers 和 accounts 關聯中含有 representatives 以讓你的程式碼更有效率。

```ruby
class Supplier < ApplicationRecord
  has_one :account, -> { includes :representative }
end

class Account < ApplicationRecord
  belongs_to :supplier
  belongs_to :representative
end

class Representative < ApplicationRecord
  has_many :accounts
end
```

##### `readonly`

若使用 `readonly` 則關聯物件取出時為唯讀。

##### `select`

`select` 方法讓你覆寫用來讀取資料關聯物件的 SQL `SELECT` 語句。 Rails 預設是讀取所有的欄位。

#### 檢查關聯物件是否存在？

你可以利用 `association.nil?` 方法檢查關聯物件是否存在。

```ruby
if @supplier.account.nil?
  @msg = "No account found for this supplier"
end
```

#### 物件何時被儲存？

當你賦值一個物件到 `has_one` 關聯時，該物件會自動被儲存（這樣才能同時更新該外鍵）。除此之外，任何被取代的物件也都會自動被儲存，因為該外鍵也會同樣被改變。

如果驗證失敗時，則賦值敘述會回傳 `false`，賦值動作也會被取消。

若父物件（有 `has_one` 的 Model）尚未儲存（也就是 `new_record?` 回傳 `true`），則不會儲存子物件。只有在父物件儲存時，才會儲存子物件。

若你希望賦值一個物件到 `has_one` 關聯而不要儲存該物件的話，使用 `association.build` 方法。


### `has_many` 關聯參考

`has_many` 關聯建立兩個 Model 之間一對多的關聯，在資料庫的術語是，這個關聯中的另一個類別有外鍵會對應到這類別的實體。

#### Methods Added by `has_many`

當你宣告一個 `has_many` 關聯時，被宣告的類別會自動新增 16 種關聯方法：

* `collection`
* `collection<<(object, ...)`
* `collection.delete(object, ...)`
* `collection.destroy(object, ...)`
* `collection=(objects)`
* `collection_singular_ids`
* `collection_singular_ids=(ids)`
* `collection.clear`
* `collection.empty?`
* `collection.size`
* `collection.find(...)`
* `collection.where(...)`
* `collection.exists?(...)`
* `collection.build(attributes = {}, ...)`
* `collection.create(attributes = {})`
* `collection.create!(attributes = {})`

在所有的方法中，`collection` 會被換成第一個傳給 `has_many` 的參數 symbol，而 `collection_singular` 會被換成第一個傳給 `has_many` 的單數型參數 symbol。舉例來說，如下的宣告：

```ruby
class Author < ApplicationRecord
  has_many :books
end
```
每一個 `Author` Model 的實體都會有以下的方法：

```ruby
books
books<<(object, ...)
books.delete(object, ...)
books.destroy(object, ...)
books=(objects)
book_ids
book_ids=(ids)
books.clear
books.empty?
books.size
books.find(...)
books.where(...)
books.exists?(...)
books.build(attributes = {}, ...)
books.create(attributes = {})
books.create!(attributes = {})
```

##### `collection`

`collection` 方法會回傳含所有關聯物件的陣列。如果沒有關聯物件，則會回傳一個空陣列。

```ruby
@books = @author.books
```

##### `collection<<(object, ...)`

`collection<<` 方法會藉由設定外鍵到被加入物件的主鍵，來加一個或多個物件到關聯集合裡。

```ruby
@author.books << @book1
```

##### `collection.delete(object, ...)`

`collection.delete` 方法藉由將外鍵設定為 `NULL` 從關聯集合中移除一個或多個物件。


```ruby
@author.books.delete(@book1)
```

WARNING: 除此之外，關聯若是設定為 `dependent: :destroy` 物件就會被 destroy，若是設定為 `dependent: :delete_all` 則物件會被 delete。

##### `collection.destroy(object, ...)`

`collection.destroy` 方法藉由執行 `destroy` 從集合中移除一個或多個物件。

```ruby
@author.books.destroy(@book1)
```

WARNING: 物件 _都會_ 從資料庫中被刪除，不論 `:dependent` 選項設定為何。

##### `collection=(objects)`

`collection=` 方法更改集合內容，根據提供的物件來決定要刪除還是新增。

##### `collection_singular_ids`

`collection_singular_ids` 方法回傳一個陣列中各個物件的 id 。

```ruby
@book_ids = @author.book_ids
```

##### `collection_singular_ids=(ids)`

`collection_singular_ids=` 方法使集合只含有主鍵值的物件，根據所提供的主鍵值來決定要刪除還是新增。

##### `collection.clear`

`collection.clear` 方法會根據 `dependent` 設定的選項從集合中移除所有物件。若是沒有給予設定，則會根據預設的方式。`has_many :through` 關聯的預設方式是 `delete_all`，`has_many` 關聯的預設方式則是將所有外鍵設為 `NULL`。

```ruby
@author.books.clear
```

WARNING: 物件會被刪除如果將關聯設定為 `dependent: :destroy` 如同 `dependent: :delete_all`。

##### `collection.empty?`

`collection.empty?` 方法回傳 `true` 如果集合沒有包含任何關聯物件。

```erb
<% if @author.books.empty? %>
  No Books Found
<% end %>
```

##### `collection.size`

`collection.size` 方法回傳集合中物件的數量。

```ruby
@book_count = @author.books.size
```

##### `collection.find(...)`

`collection.find` 方法會在集合中尋找物件。這方法和 `ActiveRecord::Base.find` 使用相同的語法於選項。

```ruby
@available_books = @author.books.find(1)
```

##### `collection.where(...)`

`collection.where` 方法會在集合中根據提供的條件尋找物件，預設是惰性載入，只有在需要用到物件時才會去資料庫做查詢。

```ruby
@available_books = @author.books.where(available: true) # No query yet
@available_book = @available_books.first # Now the database will be queried
```

##### `collection.exists?(...)`

`collection.exists?` 方法檢查符合條件的物件是否在集合中，語法和選項與 [`ActiveRecord::Base.exists?`](http://api.rubyonrails.org/classes/ActiveRecord/FinderMethods.html#method-i-exists-3F) 相同。

##### `collection.build(attributes = {}, ...)`

`collection.build` 方法回傳單個或一個陣列的關聯類型新物件。物件被傳入的屬性初始化，且會自動設定外鍵，但是關聯物件並 _不_ 會儲存至資料庫中。 

```ruby
@book = @author.books.build(published_at: Time.now,
                                book_number: "A12345")

@books = @author.books.build([
  { published_at: Time.now, book_number: "A12346" },
  { published_at: Time.now, book_number: "A12347" }
])
```

##### `collection.create(attributes = {})`

`collection.create` 方法回傳單個或一個陣列的關聯類型新物件。這些物件被傳入的屬性初始化，且會自動設定外鍵。一旦它通過所有在關聯物件 Model 上的驗證，關聯物件便 _會_ 儲存至資料庫中。 

```ruby
@book = @author.books.create(published_at: Time.now,
                                 book_number: "A12345")

@books = @author.books.create([
  { published_at: Time.now, book_number: "A12346" },
  { published_at: Time.now, book_number: "A12347" }
])
```

##### `collection.create!(attributes = {})`

與 `collection.create` 方法相同，但在驗證失敗時會提出 `ActiveRecord::RecordInvalid`。

#### Options for `has_many`

Rails 聰明的預設值足以應付多數場景，但也有需要自訂 `has_many` 關聯的時候，這些自訂的關聯很容易能被建造出來，只要在建立關聯時傳入選項和工作區塊，便能達成，舉例來說：這些關聯有兩種選項：


```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :delete_all, validate: false
end
```

`has_many` 關聯支援下面的選項：

* `:as`
* `:autosave`
* `:class_name`
* `:counter_cache`
* `:dependent`
* `:foreign_key`
* `:inverse_of`
* `:primary_key`
* `:source`
* `:source_type`
* `:through`
* `:validate`

##### `:as`

設定 `:as` 選項表示這是一個多型關聯，在[本篇前面的小節](#polymorphic-associations)已經討論過。

##### `:autosave`

如果將 `:autosave` 的選項設定為 `true`， Rails 會在儲存父物件時，自動保存子物件。如子物件標記為刪除，也會在儲存時自動刪除。

##### `:class_name`

如果關聯 Model 的名稱推論不出來的時，你可以使用 `:class_name` 選項來指定名稱。舉例來說，如果一個作者擁有許多書，但是實際上含有書的 Model 名叫做 `Transaction`，你能這麼設定：

```ruby
class Author < ApplicationRecord
  has_many :books, class_name: "Transaction"
end
```

##### `:counter_cache`

`:counter_cache` 選項可以更有效的找出所屬物件的數量。 這個選項只有用在 [belongs_to association](#options-for-belongs-to) 自訂 `:counter_cache` 名稱時。

##### `:dependent`

在刪除關聯物件擁有者時該如何處理關聯物件：

* `:destroy` 讓關聯物件也同時被刪除。
* `:delete_all` 讓關聯物件直接從資料庫中被刪除（回呼不會被執行）。
* `:nullify` 讓外鍵被射成 `NULL` （回呼不會被執行）。
* `:restrict_with_exception` 如果有ㄧ關聯物件，就會對擁有者拋出異常。
* `:restrict_with_error` 如果有ㄧ關聯物件，錯誤就會被加入到擁有者中。

##### `:foreign_key`

照慣例而言，Rails 會假設擁有外鍵的外鍵都是以 `_id` 為字尾命名的。然而，`:foreign_key` 選項能讓你直接設定外鍵的名字：


```ruby
class Author < ApplicationRecord
  has_many :books, foreign_key: "cust_id"
end
```
TIP: 在任何情況下， Rails 並不會新增外鍵欄位給你，你必須在遷移裡明確地標明它們。

##### `:inverse_of`

`:inverse_of` 選項具體指明了 `belongs_to` 關聯中另一端的名字。此選項無法與 `:through` 或 `:as` 選項連用。

```ruby
class Author < ApplicationRecord
  has_many :books, inverse_of: :author
end

class Book < ApplicationRecord
  belongs_to :author, inverse_of: :books
end
```

##### `:primary_key`

按照慣例，Rails 是以 `id` 欄位放置主鍵的資料表。`:primary_key` 選項可以讓你具體指定一個不一樣的欄位。

舉例來說，現在我們擁有一個 `users` 的資料表而 `id` 為該資料表的主鍵，但同時也擁有 `guid` 的欄位。條件是，我們的 `todos` 資料表的 `guid` 欄位的值應該要與外鍵相同，而不是與 `id` 的值相同，我們可以這樣做以達到這樣的效果：

```ruby
class User < ApplicationRecord
  has_many :todos, primary_key: :guid
end
```

如果我們現在執行 `@todo = @user.todos.create`，那麼 `@todo` 記錄中的 `user_id` 值就會變成 `@user` 的 `guid` 值。

##### `:source`

`:source` 選項標明了 `has_many :through` 關聯的來源關聯名稱。要是來源關聯的名稱並不能自動從關聯名中被推論出來，才需要使用此選項。


##### `:source_type`

`:source_type` 選項標明了透過多型關聯的 `has_many :through` 關聯的來源類型。

##### `:through`

`:through` 選項用來指定下查詢的連接 Model。`has_many :through` 關聯在[前面已詳細介紹過](#the-has-one-through-association)。

##### `:validate`

如果你設定 `:validate` 選項為 `false` 的話，那麼不管何時進行儲存，該關聯物件都不會被驗證。預設的情況下，這是設定 `true`：在儲存的時候，關聯物件會被驗證。

#### `has_many` 工作域

有時你可能希望能客製化 `has_many` 使用的查詢語句。可以透過傳入作用域區塊 (scope block) 來達到，比如：

```ruby
class Author < ApplicationRecord
  has_many :books, -> { where processed: true }
end
```

你可以在工作域區塊使用任何一種 [查詢方法](active_record_querying.html)。下面我們將會討論幾個：

* `where`
* `extending`
* `group`
* `includes`
* `limit`
* `offset`
* `order`
* `readonly`
* `select`
* `distinct`

##### `where`

`where` 方法讓你指定關聯物件必須達到的條件。

```ruby
class Author < ApplicationRecord
  has_many :confirmed_books, -> { where "confirmed = 1" },
    class_name: "Book"
end
```
透過 Hash 建立條件： 

```ruby
class Author < ApplicationRecord
  has_many :confirmed_books, -> { where confirmed: true },
                              class_name: "Book"
end
```

若使用 Hash 型式的 `where` 選項，產生出來的記錄會自動使用 Hash 工作域。上例中，`@author.confirmed_books.create` 或 `@author.confirmed_books.build` 會建立 books 且 confirmed 的欄位為 true。

##### `extending`

`extending` 方法指定了一個模組的名稱，用來擴充 association proxy。關聯的擴充會[在之後的章節](#association-extensions) 談到。

##### `group`

`group` 方法提供用來對結果進行分組的屬性名稱，用在 SQL 的 `GROUP BY` 子句裡。

```ruby
class Author < ApplicationRecord
  has_many :line_items, -> { group 'books.id' },
                        through: :books
end
```

##### `includes`

你可以使用 `includes` 方法用來指定需要 Eager Loading 的第二層關聯。看看下面這個例子：

```ruby
class Author < ApplicationRecord
  has_many :books
end

class Book < ApplicationRecord
  belongs_to :author
  has_many :line_items
end

class LineItem < ApplicationRecord
  belongs_to :book
end
```

如果你很頻繁地直接從 authors `@author.books.line_items`) 中取出 line items，那你應該在關聯中含有 line items 讓你的程式碼更有效率。如下：

```ruby
class Author < ApplicationRecord
  has_many :books, -> { includes :line_items }
end

class Book < ApplicationRecord
  belongs_to :author
  has_many :line_items
end

class LineItem < ApplicationRecord
  belongs_to :book
end
```

##### `limit`

`limit` 方法讓你限制從關聯中被取出的物件總數。

```ruby
class Author < ApplicationRecord
  has_many :recent_books,
    -> { order('published_at desc').limit(100) },
    class_name: "Book",
end
```

##### `offset`

`offset` 方法讓你指定從關聯取出物件的偏移量。比如 `-> { offset(11) }` 會忽略前 11 個物件。

##### `order`

`order` 方法指定關聯物件取出後的排序方式（語法為 SQL 的 `ORDER BY` 子句）。

```ruby
class Author < ApplicationRecord
  has_many :books, -> { order "date_confirmed DESC" }
end
```

##### `readonly`

若使用 `readonly` 則關聯物件取出時為唯讀。

##### `select`

`select` 方法讓你覆寫用來讀取資料關聯物件的 SQL `SELECT` 語句。 Rails 預設是讀取所有的欄位。

WARNING: 如果你明確指定了自己的 `select`，記得要包含關聯 Model 的主鍵和外鍵的欄位。如果沒有這麼做， Rails 會出現錯誤。

##### `distinct`

`distinct` 方法能讓集合免於重複的物件。最好的情況是和 `:through` 選項一起使用。

```ruby
class Person < ApplicationRecord
  has_many :readings
  has_many :articles, through: :readings
end

person = Person.create(name: 'John')
article   = Article.create(name: 'a1')
person.articles << article
person.articles << article
person.articles.inspect # => [#<Article id: 5, name: "a1">, #<Article id: 5, name: "a1">]
Reading.all.inspect  # => [#<Reading id: 12, person_id: 5, article_id: 5>, #<Reading id: 13, person_id: 5, article_id: 5>]
```
在上述的情況中，總共有兩篇 readings。即使這兩篇是相同的文章，`person.articles` 還是兩篇都回傳了。

現在我們使用 `distinct`：

```ruby
class Person
  has_many :readings
  has_many :articles, -> { distinct }, through: :readings
end

person = Person.create(name: 'Honda')
article   = Article.create(name: 'a1')
person.articles << article
person.articles << article
person.articles.inspect # => [#<Article id: 7, name: "a1">]
Reading.all.inspect  # => [#<Reading id: 16, person_id: 7, article_id: 7>, #<Reading id: 17, person_id: 7, article_id: 7>]
```
上述的情況中，還是有兩篇 readings，但是 `person.articles` 只回傳了一篇，因為集合只載入唯一的記錄。

如果你想要確保所有的記錄在寫入關聯之前都是唯一的（以便之後使用關聯時，不會有重複的記錄），你應該加入一個 unique 索引在資料表中。舉例來說，如果你有一個資料表名為 `readings` 且想確保所有文章不重複，可加入下面這個遷移：

```ruby
add_index :readings, [:person_id, :article_id], unique: true
```

一旦有了這個 unique 索引，想企圖加入兩次 acticle 到 person 中， `ActiveRecord::RecordNotUnique` 就會出現錯誤。

```ruby
person = Person.create(name: 'Honda')
article = Article.create(name: 'a1')
person.articles << article
person.articles << article # => ActiveRecord::RecordNotUnique
```

若是使用 `include?` 來檢查唯一性可能會產生 race condition（競態條件）。不要使用 `include?` 來強制關聯中的唯一性。就上述的 article 而言，下面的程式碼可能就會形成競態條件，因為可能會同時有多個使用者嘗試加入文章。 

```ruby
person.articles << article unless person.articles.include?(article)
```

#### 物件何時被儲存？

當你賦值一個物件到 `has_many` 關聯時，該物件會自動被儲存（這樣才能同時更新該外鍵）。如果你在一個敘述中賦值多個物件，這些物件都會被儲存。

如果驗證失敗時，則賦值敘述會回傳 `false`，賦值動作也會被取消。

若父物件（有 `has_many` 的 Model）尚未儲存（也就是 `new_record?` 回傳 `true`），則不會儲存子物件。只有在父物件儲存時，才會儲存子物件。

若你希望賦值一個物件到 `has_many` 關聯而不要儲存該物件的話，使用 `association.build` 方法。

### `has_and_belongs_to_many` 關聯參考

`has_and_belongs_to_many` 關聯建立兩個 Model 之間多對多的關係。以資料庫的術語解釋，透過中介的連接資料表將兩個 Model 關聯起來，連接資料表記錄了兩個類別的外鍵。



#### Methods Added by `has_and_belongs_to_many`

當你宣告一個 `has_and_belongs_to_many` 關聯時，被宣告的類別會自動新增 16 種關聯方法：

* `collection`
* `collection<<(object, ...)`
* `collection.delete(object, ...)`
* `collection.destroy(object, ...)`
* `collection=(objects)`
* `collection_singular_ids`
* `collection_singular_ids=(ids)`
* `collection.clear`
* `collection.empty?`
* `collection.size`
* `collection.find(...)`
* `collection.where(...)`
* `collection.exists?(...)`
* `collection.build(attributes = {})`
* `collection.create(attributes = {})`
* `collection.create!(attributes = {})`

在所有的方法中，`collection` 會被換成第一個傳給 `has_and_belongs_to_many` 的參數 symbol，而 `collection_singular` 會被換成第一個傳給 `has_and_belongs_to_many` 的單數型參數 symbol。舉例來說，如下的宣告：

```ruby
class Part < ApplicationRecord
  has_and_belongs_to_many :assemblies
end
```

每個 `Part` Model 的實體都有以下的方法：

```ruby
assemblies
assemblies<<(object, ...)
assemblies.delete(object, ...)
assemblies.destroy(object, ...)
assemblies=(objects)
assembly_ids
assembly_ids=(ids)
assemblies.clear
assemblies.empty?
assemblies.size
assemblies.find(...)
assemblies.where(...)
assemblies.exists?(...)
assemblies.build(attributes = {}, ...)
assemblies.create(attributes = {})
assemblies.create!(attributes = {})
```

##### 額外的欄位方法

如果 `has_and_belongs_to_many` 關聯的連接資料表有除了外鍵之外的欄位，這些欄位會被當成屬性透過關聯被加入紀錄中。記錄回傳的這些額外屬性是唯讀的，因為 Rails 無法儲存這些屬性的變動。

WARNING: `has_and_belongs_to_many` 關聯的連接資料表裡使用額外欄位已棄用。若多對多關係需要如此複雜的行為，應該使用 `has_many :through` 關聯。

##### `collection`

`collection` 方法會回傳含有全部關聯物件的陣列。如果沒有關聯物件，則會回傳一個空陣列。

```ruby
@assemblies = @part.assemblies
```

##### `collection<<(object, ...)`

`collection<<` 方法會藉由新增記錄到連接資料表中，加一個或多個物件到關聯集合裡。

```ruby
@part.assemblies << @assembly1
```
NOTE: 這個方法是 `collection.concat` 與 `collection.push` 的別名。

##### `collection.delete(object, ...)`

`collection.delete` 方法藉由刪除連接資料表的記錄，從關聯集合中移除一個或多個物件。但並不會刪除物件本身。

```ruby
@part.assemblies.delete(@assembly1)
```
WARNING: 這並不會在連接資料表中觸發回呼。

##### `collection.destroy(object, ...)`

`collection.destroy` 方法藉由在連接資料表中執行 `destroy` 與回呼從集合中移除一個或多個物件。但並不會刪除物件本身。

```ruby
@part.assemblies.destroy(@assembly1)
```

##### `collection=(objects)`

`collection=` 方法更改集合內容，根據提供的物件來決定要刪除還是新增。

##### `collection_singular_ids`

`collection_singular_ids` 方法回傳一個陣列中各個物件的 id 。

```ruby
@assembly_ids = @part.assembly_ids
```

##### `collection_singular_ids=(ids)`

`collection_singular_ids=` 方法使集合中只含有主鍵值的物件，根據所提供的主鍵值來決定要刪除還是新增。

##### `collection.clear`

`collection.clear` 方法會藉由從連接資料表中刪除欄位來移除在集合中的物件。但這並不會刪除關聯物件本身。

##### `collection.empty?`

`collection.empty?` 方法回傳 `true` 如果集合沒有包含任何關聯物件。

```ruby
<% if @part.assemblies.empty? %>
  This part is not used in any assemblies
<% end %>
```

##### `collection.size`

`collection.size` 方法回傳集合中物件的數量。

```ruby
@assembly_count = @part.assemblies.size
```

##### `collection.find(...)`

`collection.find` 方法會在集合中尋找物件。這方法和 `ActiveRecord::Base.find` 使用相同的語法於選項。


```ruby
@assembly = @part.assemblies.find(1)
```

##### `collection.where(...)`

`collection.where` 方法會在集合中根據提供的條件尋找物件，預設是惰性載入，只有在需要用到物件時才會去資料庫做查詢。同時增加新的額外條件：物件必須在集合裡。

```ruby
@new_assemblies = @part.assemblies.where("created_at > ?", 2.days.ago)
```

##### `collection.exists?(...)`

`collection.exists?` 方法依提供的條件檢查物件是否在集合中，語法和選項與 [`ActiveRecord::Base.exists?`](http://api.rubyonrails.org/classes/ActiveRecord/FinderMethods.html#method-i-exists-3F) 相同。


##### `collection.build(attributes = {})`

`collection.build` 方法回傳單個或一個陣列的關聯類型新物件。這些物件會被傳入的屬性初始化，且會自動設定外鍵，但是關聯物件並 _不_ 會儲存至資料庫中。 

```ruby
@assembly = @part.assemblies.build({assembly_name: "Transmission housing"})
```

##### `collection.create(attributes = {})`

`collection.create` 方法回傳單個或一個陣列的關聯類型新物件。這些物件會被傳入的屬性初始化，且會自動設定外鍵。一旦它通過所有在關聯物件 Model 上所有的驗證，關聯物件便 _會_ 儲存至資料庫中。 

```ruby
@assembly = @part.assemblies.create({assembly_name: "Transmission housing"})
```

##### `collection.create!(attributes = {})`

功能和上述的 `collection.create` 相同，但是在驗證無效時會提出 `ActiveRecord::RecordInvalid`。


#### `has_and_belongs_to_many` 關聯可用選項

Rails 聰明的預設值足以應付多數場景，但也有需要自訂 `has_and_belongs_to_many` 關聯的時候，這些自訂的關聯很容易能被建造出來，只要在建立關聯時傳入選項和工作區塊，便能達成，舉例來說，這些關聯有兩種選項：


```ruby
class Parts < ApplicationRecord
  has_and_belongs_to_many :assemblies, -> { readonly },
                                       autosave: true
end
```
 `has_and_belongs_to_many` 關聯支援下面的選項：

* `:association_foreign_key`
* `:autosave`
* `:class_name`
* `:foreign_key`
* `:join_table`
* `:validate`

##### `:association_foreign_key`

照慣例而言，Rails 假定連接資料表的外鍵名稱為 Model 名稱加上 `_id`。然而，`:association_foreign_key` 選項能讓你直接設定外鍵名稱：

TIP: 若是要建立一個多對多的自連接，那麼 `:foreign_key` 和 `:association_foreign_key` 關聯選項非常好用。

```ruby
class User < ApplicationRecord
  has_and_belongs_to_many :friends,
      class_name: "User",
      foreign_key: "this_user_id",
      association_foreign_key: "other_user_id"
end
```

##### `:autosave`

你果你將 `:autosave` 的選項設為 `true`，Rails 會在儲存父物件時，自動保存子物件。如子物件標記為刪除，也會在儲存時自動刪除。

##### `:class_name`

如果關聯 Model 的名稱推論不出來的時，你可以使用 `:class_name` 選項來指定名稱。舉例來說，如果一個裝置含有許多零件，但是實際上含有零件的 Model 名叫做 `Gadget`，你能這麼設定：

```ruby
class Parts < ApplicationRecord
  has_and_belongs_to_many :assemblies, class_name: "Gadget"
end
```

##### `:foreign_key`

照慣例而言，Rails 假定連接資料表的外鍵名稱為 Model 名稱加上 `_id`。然而，`:foreign_key` 選項能讓你直接設定外鍵的名字：


```ruby
class User < ApplicationRecord
  has_and_belongs_to_many :friends,
      class_name: "User",
      foreign_key: "this_user_id",
      association_foreign_key: "other_user_id"
end
```

##### `:join_table`

連接資料表的預設名是根據詞序法推出，如果這不是你需要的，你可以利用 `:join_table` 選項覆寫掉預設。

##### `:validate`

如果你設定 `:validate` 選項為 `false` 的話，那麼不管何時進行儲存，該關聯物件都不會被驗證。預設的情況下，這是設定 `true`：在儲存的時候，關聯物件會被驗證。

#### `has_and_belongs_to_many` 工作域

有時你可能希望客製化 `has_and_belongs_to_many` 使用的查詢語句。可以透過傳入作用域區塊 (scope block) 來達到，比如：

```ruby
class Parts < ApplicationRecord
  has_and_belongs_to_many :assemblies, -> { where active: true }
end
```

你可以在工作域區塊使用任何一種 [查詢方法](active_record_querying.html)。下面我們將會討論幾個：

* `where`
* `extending`
* `group`
* `includes`
* `limit`
* `offset`
* `order`
* `readonly`
* `select`
* `distinct`

##### `where`

`where` 方法讓你指定關聯物件必須達到的條件。


```ruby
class Parts < ApplicationRecord
  has_and_belongs_to_many :assemblies,
    -> { where "factory = 'Seattle'" }
end
```

透過 Hash 建立條件：

```ruby
class Parts < ApplicationRecord
  has_and_belongs_to_many :assemblies,
    -> { where factory: 'Seattle' }
end
```

若使用 Hash 型式的 `where` 選項，產生出來的記錄會自動使用 Hash 工作域。上例中，`@parts.assemblies.create` 或 `@parts.assemblies.build` 會建立 `factory` 欄位且 confirmed 的值為 "Seattle"。

##### `extending`

`extending` 方法指定了一個模組的名稱，用來擴充 association proxy。關聯的擴充會[在之後的章節](#association-extensions) 談到。

##### `group`

`group` 方法提供用來對結果進行分組的屬性名稱，用在 SQL 的 `GROUP BY` 子句裡。

```ruby
class Parts < ApplicationRecord
  has_and_belongs_to_many :assemblies, -> { group "factory" }
end
```

##### `includes`

你可以使用 `includes` 方法用來指定需要 Eager Loading 的第二層關聯。看看下面這個例子：

##### `limit`

`limit` 方法讓你限制從關聯中被取出的物件總數。

```ruby
class Parts < ApplicationRecord
  has_and_belongs_to_many :assemblies,
    -> { order("created_at DESC").limit(50) }
end
```

##### `offset`

`offset` 方法讓你指定從關聯取出物件的偏移量。比如 `-> { offset(11) }` 會忽略前 11 個物件。

##### `order`

`order` 方法指定關聯物件取出後的排序方式（語法為 SQL 的 `ORDER BY` 子句）。


```ruby
class Parts < ApplicationRecord
  has_and_belongs_to_many :assemblies,
    -> { order "assembly_name ASC" }
end
```

##### `readonly`

若使用 `readonly` 則關聯物件取出時為唯讀。

##### `select`

`select` 方法讓你覆寫用來讀取資料關聯物件的 SQL `SELECT` 語句。 Rails 預設是讀取所有的欄位。

##### `distinct`

`distinct` 方法用來移除集合中重複的物件。

#### 物件何時被儲存？

當你賦值一個物件到 `has_and_belongs_to_many` 關聯時，該物件會自動被儲存（這樣才能同時更新該外鍵）。如果你在一個敘述中賦值多個物件，這些物件都會被儲存。

如果驗證失敗時，則賦值敘述會回傳 `false`，賦值動作也會被取消。

若父物件（有 `has_and_belongs_to_many` 的 Model）尚未儲存（也就是 `new_record?` 回傳 `true`），則不會儲存子物件。只有在父物件儲存時，才會儲存子物件。

若你希望賦值一個物件到 `has_and_belongs_to_many` 關聯而不要儲存該物件的話，使用 `association.build` 方法。


### 關聯回呼

一般回呼會介入 Active Record 物件的生命週期，讓你隨時使用對這些物件做處理。舉例來說，你可以使用 `:before_save` 回呼，在物件被儲存之前做處理。

關聯回呼和一般的回呼相同，但是受到集合生命週期事件的觸發。以下是四種關聯回呼：

* `before_add`
* `after_add`
* `before_remove`
* `after_remove`

宣告關聯時可藉由新增選項到來定義關聯回呼，如下的例子：

```ruby
class Author < ApplicationRecord
  has_many :books, before_add: :check_credit_limit

  def check_credit_limit(book)
    ...
  end
end
```
Rails 將新增或移除的物件傳入回呼中。
單一事件可觸發多個回呼，需以陣列形式指定：


```ruby
class Author < ApplicationRecord
  has_many :books,
    before_add: [:check_credit_limit, :calculate_shipping_charges]

  def check_credit_limit(book)
    ...
  end

  def calculate_shipping_charges(book)
    ...
  end
end
```
若是 `before_add` 回呼丟出一個異常，這個物件並不會被加入集合中。同樣地，要是 `before_remove` 回呼丟出一個異常，這個物件也不會被集合移除。

### 擴充關聯

不必拘限於 Rails 給關聯代理物件所加入的功能。可以透過匿名模組、加入新的查詢方法、建立物件的新方法或其他方法給貫連物件擴充功能。

```ruby
class Author < ApplicationRecord
  has_many :books do
    def find_by_book_prefix(book_number)
      find_by(category_id: book_number[0..2])
    end
  end
end
```
若是有功能可讓許多關聯共享，可以使用命名的擴充模組，比如：

```ruby
module FindRecentExtension
  def find_recent
    where("created_at > ?", 5.days.ago)
  end
end

class Author < ApplicationRecord
  has_many :books, -> { extending FindRecentExtension }
end

class Supplier < ApplicationRecord
  has_many :deliveries, -> { extending FindRecentExtension }
end
```
擴充功能可以參照到關聯代理的內部，透過使用以下三個 `proxy_association` 的存取器：

* `proxy_association.owner` 回傳關聯物件的擁有者。
* `proxy_association.reflection` 回傳描述關聯的反射動作（reflection object)。
* `proxy_association.target` 回傳 `belongs_to` 或 `has_one` 的關聯物件，或是 `has_many` or `has_and_belongs_to_many` 的關聯物件集合。


單表繼承
------------------------

有時你可能需要不同 Model 之間共享欄位與行為。比如說我們有 Car、Motorcycle 以及 Bicycle 這三個 Model。三個 Model 之間我們希望能共享 `color`、`price` 欄位和其他方法，但各自又有特定的行為與 controller 。

Rails 讓這件是做起來很容易。首先，我們先產生一個 Vehicle Model 作為基礎: 

```bash
$ rails generate model vehicle type:string color:string price:decimal{10.2}
```

注意到我們增加了一個 type 欄位嗎？因為所有的 Model 都會被存在一張「單一的資料表」中，Rails 會將這個 Model 名稱存在 type 欄位裡。以我們的例子來說，這些 type 的值就會是 "Car"、"Motorcycle"、"Bicycle"。單表繼承（Single Table Inheritance, STI) 需要一個 type 欄位才可以運作。

接下來，我們會產生三個從 Vehicle Model 繼承而來的 Model。這裡我們可以使用 `--parent=PARENT` 選項，會產生出繼承指定 parent 且不會產生遷移檔（因為資料表已經存在）。

比如說，產生一個 Car Model:

```bash
$ rails generate model car --parent=Vehicle
```

產生的 Model 看起來會像是：

```ruby
class Car < Vehicle
end
```

這表示所有加入 Vehicle 的行為 Car Model 裡也有，譬如關聯、公共方法等等。

新建 Car 會被存在 `vehicles` 資料表裡， `type` 會被設為 "Car":

```ruby
Car.create(color: 'Red', price: 10000)
```
會產生下列的 SQL: 

```sql
INSERT INTO "vehicles" ("type", "color", "price") VALUES ('Car', 'Red', 10000)
```
查詢 car 的記錄則會搜尋 vehicles 中的 "car":

```ruby
Car.all
```
會執行像是下面的查詢：

```sql
SELECT "vehicles".* FROM "vehicles" WHERE "vehicles"."type" IN ('Car')
```
