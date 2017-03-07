**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON http://guides.rubyonrails.org.**

Active Record 驗證
=========================

本篇將教你如何使用 Active Record 的驗證功能，驗證物件存入資料庫前的狀態。

讀完本篇之後你將會知道：

* 如何使用 Active Record 內建的驗證輔助方法。
* 如何新建你自己的驗證方法。
* 該如何處理驗證時所產生的錯誤訊息。


--------------------------------------------------------------------------------

驗證總觀
--------------------

以下是一個簡單的驗證例子：

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end

Person.create(name: "John Doe").valid? # => true
Person.create(name: nil).valid? # => false
```

從上述的驗證中 `Person` 必須要有一個 `name` 的屬性才算有效。所以第二個 `Person` 因為缺少了 `name` 屬性所以不會被存到資料庫中。

在我們更深入探討細節之前，我們先談談驗證在應用程式中所扮演的角色。

### 為什麼要驗證？

驗證是用來確保只有有效的資料才能存入資料庫，比如對一個應用程式而言，每個使用者都提供有效的Email及郵件地址是非常重要的。Model 層級的驗證是最好的，它能確保只有有效的資料存入資料庫中，且不需要考慮資料庫的種類、無法在用戶端（瀏覽器）跳過驗證、且更容易測試與維護。Rails 使資料驗證用起來非常簡單，提供了各種內建輔助方法，來滿足常見的需求，也可以新建自訂的驗證方法。

以下是一些可以在資料被存入資料庫前進行驗證的方式，包含原生資料庫的約束（constraint）、客戶端的驗證及 controller 層級的驗證。以下是對於驗證的優缺點整理：

* 資料庫約束和存取手續使驗證機制只適用於單一資料庫，也因此讓測試及維護更加困難。然而，如果其他的應用程式也使用你的資料庫，最好的方式就是在資料庫層級上使用一些約束。除此之外，資料庫層級的驗證可以很安全地處理某些在其他層級上較難處理的問題（例如在使用頻繁的資料表裡檢查唯一性）。

* 用戶端驗證可以很有幫助，但單獨使用時可靠性不高。如果是透過 Javascript 執行的話，將瀏覽器中的Javascript 關掉便能跳過驗證。若是結合其它種驗證方式，用戶端驗證可提供使用者即時的反饋。

* Controller 層級的驗證是一個很誘人的使用方式，但使用起來不靈活，也不易測試與維護。無論如何，盡量讓 controller 保持輕巧短小，長遠下來看，應用程式會更好維護。

根據不同的情況選擇不同的驗證方式，Rails 團隊的觀點是 Model 層級的驗證，符合最多數應用的情形。

### 驗證何時發生？

Active Record 分成兩種物件：一種是可以對應到資料庫裡的欄位，另一種則無法。當你新增了一個新的物件時，比如說你使用了 `new` ，這個物件還不算儲存在資料庫裡，一旦你呼叫了 `save` ，這個物件就會被存到正確的資料表裡。 Active Record 使用了 `new_record?` 這個實體方式去決定這物件是否已經存在在資料庫裡。參考下面這個簡單的 Active Record 類別：


```ruby
class Person < ApplicationRecord
end
```

我們能從 `rails console` 中看到它是怎麼運作的：

```ruby
$ bin/rails console
>> p = Person.new(name: "John Doe")
=> #<Person id: nil, name: "John Doe", created_at: nil, updated_at: nil>
>> p.new_record?
=> true
>> p.save
=> true
>> p.new_record?
=> false
```

新增與儲存一個紀錄都會發送一個 SQL `INSERT` 的操作到資料庫中。更新存在紀錄則會發送一個 SQL `UPDATE` 的操作。驗證通常都會在執行這些指令時就先送到資料庫中。如果有任何的驗證失敗，這個物件就會被標記為無效，Active Record 就不會執行 `INSERT` 或 `UPDATE` 的操作。這可以避免存到無效物件到資料庫中。你可以指定在物件建立、儲存、更新時，各個階段要做何種資料驗證。


CAUTION: 有許多方法可以改變物件在資料庫的狀態，有些方法會觸發驗證的機制，有些則不會，這表示有可能不小心將無效的物件存入資料庫。

以下的方式會觸發驗證機制，且只會將有效物件存入資料庫中：

* `create`
* `create!`
* `save`
* `save!`
* `update`
* `update!`

上述的 BANG 版本（比如 save!）會對無效的記錄拋出異常。非 BANG 版本的則不會，`save` 與 `update` 僅回傳 `false`，而`create` 僅回傳物件本身。

### 略過驗證

下列的方法會跳過驗證，而且會將物件存入資料庫，無論物件是否是有效，應該謹慎使用。

* `decrement!`
* `decrement_counter`
* `increment!`
* `increment_counter`
* `toggle!`
* `touch`
* `update_all`
* `update_attribute`
* `update_column`
* `update_columns`
* `update_counters`

注意 `save` 也可能跳過驗證如果傳入 `validate:false` 作為參數，所以這個技巧應該要謹慎使用。

* `save(validate: false)`

### `valid?` 與 `invalid?`

在儲存一個 Active Record 的物件前，Rails 會先進行驗證。如果這些驗證有任何的錯誤，Rails 就不會儲存這些物件。

你也可以自行進行驗證。`valid?` 會觸發驗證，如果物件中沒有任何錯誤會回傳 true ，反之就會回傳 false。前面已經見過的例子：

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end

Person.create(name: "John Doe").valid? # => true
Person.create(name: nil).valid? # => false
```

在 Active Record 完成驗證後，所有找到的錯誤都可透過 `errors.messages` 這個實例方法來存取，會回傳一個錯誤集合。就定義來看，物件完成驗證之後，錯誤集合為空才是有效的。

注意 `new` 實體化出來的物件，即使是錯誤的，也不會回報，因為驗證只有在物件要被儲存的時候才會自動執行，例如使用 `create` 或 `save` 方法時。

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end

>> p = Person.new
# => #<Person id: nil, name: nil>
>> p.errors.messages
# => {}

>> p.valid?
# => false
>> p.errors.messages
# => {name:["can't be blank"]}

>> p = Person.create
# => #<Person id: nil, name: nil>
>> p.errors.messages
# => {name:["can't be blank"]}

>> p.save
# => false

>> p.save!
# => ActiveRecord::RecordInvalid: Validation failed: Name can't be blank

>> Person.create!
# => ActiveRecord::RecordInvalid: Validation failed: Name can't be blank
```

`invalid?` 基本上是 `valid?` 的反項，物件中若找到任何錯誤會回傳 true，反之回傳 false。

### `errors[]`

要驗證一個物件的屬性是否有效。你可以使用 `errors[:attribute]`。它會以陣列形式回傳該屬性的所有錯誤，沒有錯誤則回傳空陣列。

這個方法只有在驗證 _後_ 呼叫才有用，因為它只會檢查 errors 的集合，而不會觸發驗證。 `errors[:attribute]` 與前一個 `ActiveRecord::Base#invalid?` 方法不同，因為它不是檢查整個物件的有效性，只是檢查物件中的單一屬性是否有錯誤。

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end

>> Person.new.errors[:name].any? # => false
>> Person.create.errors[:name].any? # => true
```

我們會在 [處理驗證錯誤](#working-with-validation-errors) 這個部分更加深入地討論關於驗證錯誤。




### `errors.details`

當要檢查是由哪一個無效的屬性造成驗證失敗的時候，你可以利用 `errors.details[:attribute]`，他會回傳一個由Hash組成的陣列附上 `:error` 鍵，以便找到 Validator:

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end

>> person = Person.new
>> person.valid?
>> person.errors.details[:name] # => [{error: :blank}]
```

如何使用 `details` 在自訂的驗證中會在 [處理驗證錯誤](#working-with-validation-errors) 中詳細探討。
 
驗證輔助方法
------------------

Active Record 提供了許多預先定義的驗證輔助方法讓你可以在類別定義中直接使用。這些輔助方法提供了
常見的驗證規則。每次只要一個驗證失敗，一個錯誤訊息就會被加到該物件的 `errors` 集合，而這個訊息
和出錯的屬性是相關的。

每個一輔助方式都接受任意數量的屬性名稱，所以在一行程式碼中，可以將相同的驗證加入不同的屬性。

所有的輔助方法都接受 `:on` 和 `:message` 的選項，分別定義了何時這些驗證會執行和有哪些錯誤訊息會被加到 `errors` 的集合裡。 `:on` 選項接受 `:create` 或 `:update` 的值。每個驗證輔助方法都有預設的錯誤訊息。
這些訊息在沒有指定 `:message` 選項時很有用。讓我們看看每一個可用的輔助方法。



### `acceptance`


這個驗證方法使用在當一個表單送出時，檢查使用者界面上出現的checkbox是否打勾。這對於使用者需要接受服務條款、隱私權政策、確認某些內容已閱讀等相關的情況很有用。

```ruby
class Person < ApplicationRecord
  validates :terms_of_service, acceptance: true
end
```

這個方法只會在 `terms_of_service` 不是 `nil` 的時候執行。這個輔助方法預設的錯誤訊息是 _"must be accepted"_。你也可以傳達其他自訂的訊息，透過使用 `message` 選項。


```ruby
class Person < ApplicationRecord
  validates :terms_of_service, acceptance: true, message: 'must be abided'
end
```

這個方法也接受 `:accept` 選項，用來決定什麼值代表「接受」。預設是 `['1', true]`，改成別的也很簡單。

```ruby
class Person < ApplicationRecord
  validates :terms_of_service, acceptance: { accept: 'yes' }
  validates :eula, acceptance: { accept: ['TRUE', 'accepted'] }
end
```

這個驗證是特別針對網頁應用程式，而這個 'acceptance' 不需要存入資料庫。如果沒有為它開一個欄位，輔助方法自己會使用一個虛擬屬性。如果在資料庫中已有一個欄位，`accept` 選項就必須被設成（或者包含） `true` ，不然這個驗證就不會執行。

### `validates_associated`

當 Model 和其他 Model 有關聯，且相關連的 Model 也需要被驗證時，就可以使用這個輔助方法。當你想要儲存物件時，會對相關聯的物件呼叫 `valid?`。

```ruby
class Library < ApplicationRecord
  has_many :books
  validates_associated :books
end
```

此方法適用於所有的關聯類型。

CAUTION: 不要在相關聯的兩邊都使用 `validates_associated`，這只會讓它們陷入互相呼叫的無窮迴圈。 

`validates_associated` 預設的錯誤訊息是 _"is invalid"_ 。注意每一個相關的物件都會有自己的 `errors` 集合; 錯誤不會集中到呼叫該方法的 Model。

### `confirmation`

當有兩個相同的輸入欄位（text field）內容需要完全相同時，你就可以使用這個輔助方法。舉例來說，你可能會需要確認（confirm）Email 或者密碼兩次輸入是否相同。這個驗證方式會建立一個新的虛擬屬性，名字就是該欄位 (field) 的名稱，後面加上 "_confirmation" 。

```ruby
class Person < ApplicationRecord
  validates :email, confirmation: true
end
```

在 View 模板中，可以這麼用：

```erb
<%= text_field :person, :email %>
<%= text_field :person, :email_confirmation %>
```

只會在 `email_confirmation` 不是 `nil` 的時候才會進行驗證，需要確認的話，記得給確認的屬性加上存在性（presence）。我們會在後面介紹存在性（presence）。

```ruby
class Person < ApplicationRecord
  validates :email, confirmation: true
  validates :email_confirmation, presence: true
end
```

同時還有一個 `:case_sensitive` 的選項你可以用來決定驗證是否要大小寫的區別限制，這個選項的預設是 ‘true’。

```ruby
class Person < ApplicationRecord
  validates :email, confirmation: { case_sensitive: false }
end
```

這個輔助方法的預設錯誤訊息是 _"doesn't match confirmation"_ 。

### `exclusion`

這個輔助方法驗證屬性是否「不屬於」一個特定的集合，集合需是由 enumerable 物件組成。

```ruby
class Account < ApplicationRecord
  validates :subdomain, exclusion: { in: %w(www us ca jp),
    message: "%{value} is reserved." }
end
```

`exclusion` 這個輔助方法有一個 `:in` 的選項，決定驗證屬性「不可接受」的值。`:in`的別名為 `:within`，也有相同的用法。上述的例子用了 `:message` 選項示範如何在錯誤訊息中加入屬性。關於更多 message 選項請看 [message documentation](#message)。

預設的錯誤訊息是 _"is reserved"_。

### `format`

這個輔助方法驗證屬性的值是否符合正規的表達式，藉由透過 `:with` 來標記。


```ruby
class Product < ApplicationRecord
  validates :legacy_code, format: { with: /\A[a-zA-Z]+\z/,
    message: "only allows letters" }
end
```
同樣地，也可以驗證一個屬性是否 _不_ 符合正規的表達式，利用 `:without`。

預設的錯誤訊息是 _"is invalid"_。

### `inclusion`

這個輔助方法驗證屬性的值是否「屬於」一個特定的集合，集合是任何 enumerable 物件組成。

```ruby
class Coffee < ApplicationRecord
  validates :size, inclusion: { in: %w(small medium large),
    message: "%{value} is not a valid size" }
end
```

`inclusion` 這個輔助方法也有一個 `:in` 的選項，決定屬性「可以接受」的值。`:in`的別名為 `:within`，也有相同的用法。上述的例子用了 `:message` 選項示範如何在錯誤訊息中加入屬性。關於更多 message 選項的使用請看 [message documentation](#message)。

預設的錯誤訊息是 _"is not included in the list"_。

### `length`

這個輔助方法驗證屬性值的長度，它提供許多選項來限制長度（如下）：

```ruby
class Person < ApplicationRecord
  validates :name, length: { minimum: 2 }
  validates :bio, length: { maximum: 500 }
  validates :password, length: { in: 6..20 }
  validates :registration_number, length: { is: 6 }
end
```

其他限制長度的選項還有：

* `:minimum` - 屬性不能少於特定的長度。
* `:maximum` - 屬性不能多於特定的長度。
* `:in` (或 `:within`) - 屬性的長度必須在特定的區間內，這個值必須是一個範圍。
* `:is` - 屬性的長度必須同等於一個給定的值。

預設的錯誤訊息取決於使用哪一種長度驗證，你可以自訂這些訊息利用 `:wrong_length`、`:too_long`、`:too_short` 和 `%{count}` 代表該長度限制。你同樣也能利用 `:message` 選項註名錯誤訊息。

```ruby
class Person < ApplicationRecord
  validates :bio, length: { maximum: 1000,
    too_long: "%{count} characters is the maximum allowed" }
end
```

注意這裏的預設錯誤訊息都為複數（例："is too short (minimum is %{count}characters)"）。因此，當 `:minimum` 是 1 時，應該提供一個個人化訊息或使用 `presence: true` 代替。 當 `:in` 或 `:within` 的下限小於 1 時，你也應該提供一個個人化訊息，或先驗證 `length` 再驗證 `presence`。

### `numericality`

這個輔助方法驗證屬性是不是純數字。預設的情況下，這會配對到一個可選擇的整數或者浮點數。如果要特別註明只允許整數，將 `:only_integer` 設為 true.

`:only_integer` 為 `true`，會使用下面的正規表達式來檢查屬性的值：

```ruby
/\A[+-]?\d+\z/
```

不然的話，它會試著將值利用 `Float` 轉為數字。

WARNING. 注意這個正規表示法允許最後接續新行字元。

```ruby
class Player < ApplicationRecord
  validates :points, numericality: true
  validates :games_played, numericality: { only_integer: true }
end
```

除了 `:only_integer` 之外，這個輔助方法同時接受以下的選項，來限制允許的數值：
  
* `:greater_than` - 屬性的值必須大於指定的值。預設的錯誤訊息是 _"must be greater than %{count}"_。
* `:greater_than_or_equal_to` - 屬性的值必須大於或等於指定的值。預設的錯誤訊息是 _"must be greater than or equal to %{count}"_。
* `:equal_to` - 屬性的值必須等於指定的值。預設的錯誤訊息是 _"must be equal to %{count}"_。
* `:less_than` - 屬性的值必須小於指定的值。預設的錯誤訊息是 _"must be less than %{count}"_。
* `:less_than_or_equal_to` - 屬性的值必須小於或等於指定的值。預設的錯誤訊息是 _"must be
  less than or equal to %{count}"_。
* `:other_than` - 屬性的值必須是指定的值以外的值。預設的錯誤訊息是 _"must be other than %{count}"_。
* `:odd` - 當此選項設為 true 時，屬性的值必須是奇數。預設的錯誤訊息是 _"must be odd"_。
* `:even` - 當此選項設為 true 時，屬性的值必須是偶數。預設的錯誤訊息是 _"must be even"_。

NOTE: 在預設的情況下，`numericality` 不允許值為 `nil`。但是你可以使用 `allow_nil: true` 選項將之改為允許。

預設的錯誤訊息是 _"is not a number"_。

### `presence`

這個輔助方法驗證屬性是否「存在」。它會使用 `blank?` 這個方法去確認這個屬性的值使否為 `nil` 或一個空的字串（或由空白鍵組成。）

```ruby
class Person < ApplicationRecord
  validates :name, :login, :email, presence: true
end
```
如果你要確認這項關連是「存在」的，就需要去測試這關聯物件本身(如下為： LineItem)是否存在，而不是對應的外鍵。

```ruby
class LineItem < ApplicationRecord
  belongs_to :order
  validates :order, presence: true
end
```
而在 Order 這一邊，要用 inverse_of 來檢查關聯的物件是否存在。

```ruby
class Order < ApplicationRecord
  has_many :line_items, inverse_of: :order
end
```
如果利用 `has_one` 或 `has_many` 關係來驗證關聯的物件是否存在，則會對該物件呼叫 `blank?` 與 `marked_for_destruction?`，來確定物件的存在性。

由於 `false.blank?` 為 true，如果想驗證布林欄位的存在性，應該要使用下列的驗證方法：

```ruby
validates :boolean_field_name, inclusion: { in: [true, false] }
validates :boolean_field_name, exclusion: { in: [nil] }
```
利用上述的任一驗證方法，都能驗證該值「不」為 `nil`，而通常 `nil` 會造成 `NULL` 的結果。

### `absence`

這個輔助方法驗證屬性是否「不存在」。它會使用 `present?` 方法檢查這個屬性的值使否為 `nil` 或一個空的字串（或由空白鍵組成。）

```ruby
class Person < ApplicationRecord
  validates :name, :login, :email, absence: true
end
```
如果你想要確認一項關連「不存在」，就需要去測試這關聯物件本身(如下為： LineItem)是否存在，而不是對應的外鍵。

```ruby
class LineItem < ApplicationRecord
  belongs_to :order
  validates :order, absence: true
end
```
而在 Order 這一邊，要用 inverse_of 來檢查關聯的物件是否存在。

```ruby
class Order < ApplicationRecord
  has_many :line_items, inverse_of: :order
end
```
如果透過 `has_one` 或 `has_many` 驗證關聯的物件是否存在，則會對該物件呼叫 `present?` 或 `marked_for_destruction?`，來確定物件的存在性。

由於 `false.present?` 為 false，如果想驗證布林欄位的存在性，應該要使用 `validates :field_name, exclusion: { in: [true, false] }`。
這裏的預設錯誤訊息是 _"must be blank"_。



### `uniqueness`

在物件被儲存之前，這個輔助方法驗證該屬性的值是否「唯一」，但這並不會在資料庫中產生唯一性約束，因此若同時有兩個資料庫連接，便有可能建立出兩個相同的紀錄。因此需要在資料庫加上 unique 索引。
```ruby
class Account < ApplicationRecord
  validates :email, uniqueness: true
end
```
這個驗證透過對 Model 的資料表進行 SQL 查詢，搜尋是否已經有同樣數值的紀錄存在。

`:scope` 選項可以標明一或多個屬性來限制唯一性：

```ruby
class Holiday < ApplicationRecord
  validates :name, uniqueness: { scope: :year,
    message: "should happen once per year" }
end
```
如果你希望利用 `:scope` 新建一個對於資料庫的約束，防止可能的驗證衝突，你必須在資料庫欄位中建立一個 unique 索引。請看 [the MySQL manual](http://dev.mysql.com/doc/refman/5.7/en/multiple-column-indexes.html) 關於更多的多欄位索引介紹或看 [the PostgreSQL manual](http://www.postgresql.org/docs/current/static/ddl-constraints.html) 更多唯一性約束的例子。
另有 `:case_sensitive` 選項可以用來定義是否要分大小寫。此選項預設開啟。

```ruby
class Person < ApplicationRecord
  validates :name, uniqueness: { case_sensitive: false }
end
```
WARNING. 注意某些資料庫預設搜尋是不分大小寫的。

預設的錯誤訊息是 _"has already been taken"_。

### `validates_with`

這個輔助方法將記錄傳入，另開一類別來驗證。

```ruby
class GoodnessValidator < ActiveModel::Validator
  def validate(record)
    if record.first_name == "Evil"
      record.errors[:base] << "This person is evil"
    end
  end
end

class Person < ApplicationRecord
  validates_with GoodnessValidator
end
```
NOTE: 注意錯誤會加到 `record.errors[:base]`中。這個錯誤與整個物件有關，不單只屬於某個屬性。

`validates_with` 這個輔助方法接受一個類別或一組類別。`validates_with` 沒有預設錯誤訊息，你必須要手動新增錯誤記錄到 errors 集合中。

實際運作驗證的方法，必須要定義一個參數為 `record` ，也就是要被驗證的紀錄。

就如所有的驗證相似，`validates_with` 接受：`:if`、`:unless`和 `:on` 選項。如果你傳入其他的選項，它會將傳入的選項設為 `options` 類別，如下：

```ruby
class GoodnessValidator < ActiveModel::Validator
  def validate(record)
    if options[:fields].any?{|field| record.send(field) == "Evil" }
      record.errors[:base] << "This person is evil"
    end
  end
end

class Person < ApplicationRecord
  validates_with GoodnessValidator, fields: [:first_name, :last_name]
end
```
注意這個驗證類別（上例為 GoodnessValidator）在應用程式生命週期內只會實體化*一次*，而不是每次驗證時就實體化一次。所以使用實體變數時要很小心。

如果驗證類別夠複雜的話，需要用到實體變數，可以用純 Ruby 物件（Plain Old Ruby Object, PORO） 來取代：

```ruby
class Person < ApplicationRecord
  validate do |person|
    GoodnessValidator.new(person).validate
  end
end

class GoodnessValidator
  def initialize(person)
    @person = person
  end

  def validate
    if some_complex_condition_involving_ivars_and_private_methods?
      @person.errors[:base] << "This person is evil"
    end
  end

  # ...
end
```

### `validates_each`

這個輔助方法採用區塊（block）來驗證屬性。這並沒有一個預先定義的驗證功能，你可以先在區塊裡寫要驗證的行為，而所有被傳入 `validates_each` 的屬性都會被傳入後方的區塊做驗證。在下面的例子中，
我們要驗證名和姓是否以小寫字母開頭：

```ruby
class Person < ApplicationRecord
  validates_each :name, :surname do |record, attr, value|
    record.errors.add(attr, 'must start with upper case') if value =~ /\A[[:lower:]]/
  end
end
```
這個區塊接受紀錄、屬性的名和屬性的值，你可以在區塊裡寫任何你想要驗證的行為。驗證時就新增錯誤訊息到 Model 裡，這樣就能讓該紀錄無效。

常見的驗證選項
-------------------------
以下是常見的驗證選項：

### `:allow_nil`

`:allow_nil` 選項在驗證的值是 `nil` 時會略過驗證。

```ruby
class Coffee < ApplicationRecord
  validates :size, inclusion: { in: %w(small medium large),
    message: "%{value} is not a valid size" }, allow_nil: true
end
```
更多關於 message 的內容請參考 [message documentation](#message)。

### `:allow_blank`

`:allow_blank` 選項會在驗證的值是 `blank?` （如 `nil` 或一個空的字串時）會略過驗證。

```ruby
class Topic < ApplicationRecord
  validates :title, length: { is: 5 }, allow_blank: true
end

Topic.create(title: "").valid?  # => true
Topic.create(title: nil).valid? # => true
```

### `:message`

如上已經介紹過，`:message` 這個選項可在驗證失敗時，加上自訂的錯誤訊息至 errors 集合。若沒給入此選項時，Active Record 會根據不同的輔助方法使用該預設的錯誤訊息。 `:message` 選項接受 `String` 或 `Proc`。

`String` `:message` 值可以包含 `%{value}`、`%{attribute}` 和 `%{model}`，當驗證失敗時，這些都會被動態性地取代。這些取代性是利用了 I18n gem，而這些佔位符 (placeholder) 必須要完全相符，不能有空白。

`Proc` `:message` 值給了兩種參數：一是要被驗證的物件，另一個是含 `:model`、`:attribute` 和 `:value` 的 key-value Hash 如下。

```ruby
class Person < ApplicationRecord
  # Hard-coded message
  validates :name, presence: { message: "must be given please" }

  # Message with dynamic attribute value. %{value} will be replaced with
  # the actual value of the attribute. %{attribute} and %{model} also
  # available.
  validates :age, numericality: { message: "%{value} seems wrong" }

  # Proc
  validates :username,
    uniqueness: {
      # object = person object being validated
      # data = { model: "Person", attribute: "Username", value: <username> }
      message: ->(object, data) do
        "Hey #{object.name}!, #{data[:value]} is taken already! Try again #{Time.zone.tomorrow}"
      end
    }
end
```

### `:on`

`:on` 的選項能讓你指定驗證何時發生。所有輔助方法都是預設在要存檔時觸發驗證（就是建立和更新時）也可以指定只在新建時做驗證 `on: :create`，或是只在更新時做驗證 `on: :update`。

```ruby
class Person < ApplicationRecord
  # it will be possible to update email with a duplicated value
  validates :email, uniqueness: true, on: :create

  # it will be possible to create the record with a non-numerical age
  validates :age, numericality: true, on: :update

  # the default (validates on both create and update)
  validates :name, presence: true
end
```

你也可以使用 `on:` 自訂一個內容。若是要觸發特定的自訂內容，則必須將內容名稱傳入 `valid?`、`invalid?` 或 `save`。

```ruby
class Person < ApplicationRecord
  validates :email, uniqueness: true, on: :account_setup
  validates :age, numericality: true, on: :account_setup
  
end

person = Person.new
```

`person.valid?(:account_setup)` 會同時執行兩個驗證而不用將它儲存到 Model 裡。`person.save(context: :account_setup)` 在儲存之前驗證 `person` 在 `account_setup` 的內容。在標明內容的情況下，Model 會通過標明名稱內容的驗證與其原始驗證。

嚴格驗證
------------------
你也能將使用嚴格驗證，當物件無效時回應 `ActiveModel::StrictValidationFailed`。

```ruby
class Person < ApplicationRecord
  validates :name, presence: { strict: true }
end

Person.new.valid?  # => ActiveModel::StrictValidationFailed: Name can't be blank
```
也能自訂特例到 `:strict` 中。

```ruby
class Person < ApplicationRecord
  validates :token, presence: true, uniqueness: true, strict: TokenGenerationException
end

Person.new.valid?  # => TokenGenerationException: Token can't be blank
```

條件式驗證
----------------------
有時候在物件滿足條件的情況下再進行驗證比較合理。你可以利用 `:if` 和 `:unless` 來設定條件，它們接受符號、字串、`Proc` 或 `Array`。使用 `:if` 來標明何時驗證 **應該** 發生，反之，用 `:unless` 來標明何時驗證 **不應該** 發生。

### `:if` 和 `:unless` ：使用Symbol

`:if` 和 `:unless` 接受符號，該符號對應驗證執行之前所呼叫的方法，這是最常見的用法。

```ruby
class Order < ApplicationRecord
  validates :card_number, presence: true, if: :paid_with_card?

  def paid_with_card?
    payment_type == "card"
  end
end
```

### `:if` 和 `:unless` ：使用String

`:if` 和 `:unless` 接受能被 `eval` 求值且需要包含有效的 Ruby 程式碼的字串。使用這個方法時，最好是在當字串很短的時候。

```ruby
class Person < ApplicationRecord
  validates :surname, presence: true, if: "name.nil?"
end
```

### `:if` 和 `:unless` ：使用Proc

最後，`:if` 和 `:unless` 也能接受 `Proc` 物件。使用 `Proc` 物件可以寫一行條件式在區塊裡，而不用另外寫在方法裡。一行的條件式最適合用 Proc：

```ruby
class Account < ApplicationRecord
  validates :password, confirmation: true,
    unless: Proc.new { |a| a.password.blank? }
end
```

### 組合條件式驗證

你可以使用 `with_options` 來處理一個條件擁有多個驗證的情況。

```ruby
class User < ApplicationRecord
  with_options if: :is_admin? do |admin|
    admin.validates :password, length: { minimum: 10 }
    admin.validates :email, presence: true
  end
end
```
所有在 `with_options` 區塊裡的驗證都會自動傳入 `if: :is_admin?` 這個條件裡。

### 結合驗證條件

另一種情況下，當多重條件決定是否一個驗證應該發生時，可以使用 `陣列`（`Array`）。此外，你也能將 `:if` 和`:unless` 使用在同一個驗證上。

```ruby
class Computer < ApplicationRecord
  validates :mouse, presence: true,
                    if: ["market.retail?", :desktop?],
                    unless: Proc.new { |c| c.trackpad.present? }
end
```

The validation only runs when all the `:if` conditions and none of the
`:unless` conditions are evaluated to `true`.

這個驗證只有在所有  `:if` 條件為 true 而所有 `:unless` 不為 true 的情況下才會執行。

使用自訂驗證
-----------------------------
當內建的驗證輔助方法已經無法滿足你的需求時，你也能定義自訂 Validator 或驗證方法。

### 自訂 Validators

自訂 Validators 是繼承了 `ActiveModel::Validator` 的類別。這些類別必須實踐 `validate` 方法，將 record 作為一個參數並實行驗證。使用 `validates_with` 方法來呼叫自訂驗證。

```ruby
class MyValidator < ActiveModel::Validator
  def validate(record)
    unless record.name.starts_with? 'X'
      record.errors[:name] << 'Need a name starting with X please!'
    end
  end
end

class Person
  include ActiveModel::Validations
  validates_with MyValidator
end
```
加入自訂 Validator 來驗證每一個屬性，最簡單方法是使用 `ActiveModel::EachValidator`。在這個情況下自訂的 Validator 必須實踐 `validate_each` 方法，該方法需要三個參數：record、attribute 和 value。分別對應到要驗證的紀錄、屬性、屬性值。

```ruby
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless value =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
      record.errors[attribute] << (options[:message] || "is not an email")
    end
  end
end

class Person < ApplicationRecord
  validates :email, presence: true, email: true
end
```
如上述的範例，你也能在自訂的 Validator 中結合標準的驗證方法。

### 自訂方法

你也能寫方法來驗證你的 Model 並在無效時加訊息至 `errors` 集合中。必須使用 `validate` ([API](http://api.rubyonrails.org/classes/ActiveModel/Validations/ClassMethods.html#method-i-validate)) 這個類別方法來註冊，並傳入符號至驗證方法的名字。

你能傳入一個以上的符號給每個類別方法，執行的順序按照註冊的順序。

`valid?` 這個方法會查清錯誤的集合是空的，所以當你希望驗證失敗的時，須在自訂驗證方法中加入錯誤：

```ruby
class Invoice < ApplicationRecord
  validate :expiration_date_cannot_be_in_the_past,
    :discount_cannot_be_greater_than_total_value

  def expiration_date_cannot_be_in_the_past
    if expiration_date.present? && expiration_date < Date.today
      errors.add(:expiration_date, "can't be in the past")
    end
  end

  def discount_cannot_be_greater_than_total_value
    if discount > total_value
      errors.add(:discount, "can't be greater than total value")
    end
  end
end
```
預設的情況下，該驗證在每次呼叫 `valid?` 時都會執行或者儲存該物件。也可以加 `:on` 至 `validate`  方法中來控制何時執行這些自訂驗證，如 `:create` 或 `:update`。

```ruby
class Invoice < ApplicationRecord
  validate :active_customer, on: :create

  def active_customer
    errors.add(:customer_id, "is not active") unless customer.active?
  end
end
```

處理驗證錯誤
------------------------------
除了前面提到的 `valid?` 和 `invalid?` 用法之外， Rails 也提供了很多處理 `errors` 集合的方法與查詢物件的有效性。

下面是常用的方法列表，請參考 `ActiveModel::Errors` 的文件來了解所有可用的方法。

### `errors`

回傳一個 `ActiveModel::Errors` 包含所有錯誤的實體，每個鍵都是一個屬性名稱，值則是由錯誤訊息組成的字串陣列。

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end

person = Person.new
person.valid? # => false
person.errors.messages
 # => {:name=>["can't be blank", "is too short (minimum is 3 characters)"]}

person = Person.new(name: "John Doe")
person.valid? # => true
person.errors.messages # => {}
```

### `errors[]`

`errors[]` 用來檢查特定屬性的錯誤訊息。它會回傳字串陣列形式的錯誤訊息給特定屬性，每個字串都是一個錯誤訊息。如果該屬性沒有錯誤，則返回空陣列。

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end

person = Person.new(name: "John Doe")
person.valid? # => true
person.errors[:name] # => []

person = Person.new(name: "JD")
person.valid? # => false
person.errors[:name] # => ["is too short (minimum is 3 characters)"]

person = Person.new
person.valid? # => false
person.errors[:name]
 # => ["can't be blank", "is too short (minimum is 3 characters)"]
```

### `errors.add`

`add` 方法用來新增錯誤訊息到特定的屬性裡，其接受的參數為：要加上錯誤訊息的屬性、錯誤訊息內容。
你可以使用 `errors.full_messages` 方法（同等於 `errors.to_a` ）回傳使用者端的訊息，且屬性名稱呈大寫形式，如下所示：

```ruby
class Person < ApplicationRecord
  def a_method_used_for_validation_purposes
    errors.add(:name, "cannot contain the characters !@#%*()_-+=")
  end
end

person = Person.create(name: "!@#")

person.errors[:name]
 # => ["cannot contain the characters !@#%*()_-+="]

person.errors.full_messages
 # => ["Name cannot contain the characters !@#%*()_-+="]
```

一個等同於 `errors#add` 的方法是利用  `<<` 將訊息附加在 `errors.messages` 陣列中：

```ruby
  class Person < ApplicationRecord
    def a_method_used_for_validation_purposes
      errors.messages[:name] << "cannot contain the characters !@#%*()_-+="
    end
  end

  person = Person.create(name: "!@#")

  person.errors[:name]
   # => ["cannot contain the characters !@#%*()_-+="]

  person.errors.to_a
   # => ["Name cannot contain the characters !@#%*()_-+="]
```

### `errors.details`

你可以使用 `errors.add` 方法，將一個指定的 Validator 類型回傳至 error details hash 中。

```ruby
class Person < ApplicationRecord
  def a_method_used_for_validation_purposes
    errors.add(:name, :invalid_characters)
  end
end

person = Person.create(name: "!@#")

person.errors.details[:name]
# => [{error: :invalid_characters}]
```
使 error details 中含有不允許的字元組，你能將該鍵加入 `errors.add` 中。

```ruby
class Person < ApplicationRecord
  def a_method_used_for_validation_purposes
    errors.add(:name, :invalid_characters, not_allowed: "!@#%*()_-+=")
  end
end

person = Person.create(name: "!@#")

person.errors.details[:name]
# => [{error: :invalid_characters, not_allowed: "!@#%*()_-+="}]
```

All built in Rails validators populate the details hash with the corresponding
validator type.

所有內建的 Rails Validators 會填充 detail hash 相對應的 Validator 類型。

### `errors[:base]`

可以針對整個物件狀態新增錯誤訊息，而不是針對某個特定的屬性。不論是一個物件中的哪個值所導致的錯誤，想要把物件標記為無效的時候，都可以使用這個方法。由於 `errors[:base]` 是個陣列，可以加字串進去，字串就會被當成錯誤訊息使用。

```ruby
class Person < ApplicationRecord
  def a_method_used_for_validation_purposes
    errors[:base] << "This person is invalid because ..."
  end
end
```

### `errors.clear`

`clear` 方法可以清除 `errors` 集合裡的所有錯誤。當然了，對無效物件呼叫 `errors.clear` 不會使其有效：只是暫時清除了錯誤訊息，若下次再呼叫 `valid?` 或其它儲存物件的方法時，驗證會再次觸發。如果任何驗證失敗，錯誤訊息仍會將錯誤填入 `errors` 集合。

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end

person = Person.new
person.valid? # => false
person.errors[:name]
 # => ["can't be blank", "is too short (minimum is 3 characters)"]

person.errors.clear
person.errors.empty? # => true

person.save # => false

person.errors[:name]
# => ["can't be blank", "is too short (minimum is 3 characters)"]
```

### `errors.size`

`size` 方法回傳物件錯誤訊息的總數。

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end

person = Person.new
person.valid? # => false
person.errors.size # => 2

person = Person.new(name: "Andrea", email: "andrea@example.com")
person.valid? # => true
person.errors.size # => 0
```

在 View 中顯示驗證失敗訊息
-------------------------------------
一旦 Model 建好，也加入驗證後，這個 Model 如果是用表單建成的話，你可能會希望在驗證失敗的欄位顯示錯誤訊息。

因為每個應用程式處理錯誤的方式不同，Rails 沒有直接提供 View 層級的輔助方法，來直接地幫助產生這些錯誤訊息。然而，Rails 提供大量且豐富的驗證方法，自己寫一個顯示錯誤的輔助方法也不難。當使用 Scaffold 產生時，Rails 會在 `_form.html.erb` 加入一些 ERB，用來產生 Model 的完整錯誤清單。

假設我們有個 Model 存在實體變數 `@article` 裡，View 會長像這樣：

```ruby
<% if @article.errors.any? %>
  <div id="error_explanation">
    <h2><%= pluralize(@article.errors.count, "error") %> prohibited this article from being saved:</h2>

    <ul>
    <% @article.errors.full_messages.each do |msg| %>
      <li><%= msg %></li>
    <% end %>
    </ul>
  </div>
<% end %>
```

除此之外，如果你利用 Rails 的表單輔助方法來產生表單，當一個欄位的驗證發生錯誤時，Rails 會自行產生一個多的 `<div>` 包住這個欄位 。

```
<div class="field_with_errors">
 <input id="article_title" name="article[title]" size="30" type="text" value="">
</div>
```
你可以將這個 div 加上任何樣式。Rails Scaffold 產生的預設 CSS 樣式為：

```
.field_with_errors {
  padding: 2px;
  background-color: red;
  display: table;
}
```
這表示任何一個欄位若是發生錯誤，會有 2px 的紅色邊框將其包住。
