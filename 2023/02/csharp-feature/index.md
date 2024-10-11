# C# Feature


以下紀錄一些常用的 C# 語法、特性

<!--more-->

## var

* 會自動依據給予的資料型態，在編譯時直接定義為強型別
* 編譯後不會是 var，而是強型別
* 使用 var 宣告匿名型別時，比較運算子不會一起複寫，只會處裡 Equals
  ```csharp
  var x = new { Name = "Bill", Age = 15 };
  var y = new { Name = "Bill", Age = 15 };

  Console.WriteLine($"x == y  {x == y}");  //false
  Console.WriteLine($"x.Equals(y) {x.Equals(y)}"); //true
  ```

## 匿名型別

* 直接使用 new {} 的形式建立的暫時性型別
* 相同組件 (相同專案) 中的 new { Name = ""} 產生的匿名型別都會被編譯為同一個型別物件，反之，不同專案的 new {Name = ""} 會因為編譯後屬於不同組件 (DLL)，因此運行時會建立不同的型別物件
* 匿名型別成員為唯讀

## TupleClass

* 類似於匿名型別，可動態建立物件
* 是參考型別物件

## ValueTuple

* C# 7.0 加入的型別
  - .NET Core 2.0
  - .NET Framework 4.7
  - 低於以上版本請從 nuget 安裝套件
* 為結構，是實質型別
* 可以有 1~8 個泛型成員
  - 第 8 的成員必須為結構
* 使用完畢後會釋放使用的記憶體資源
* 資料成員是欄位 (Field)
  ```csharp
  // 顯式宣告使用 ValueTuple.Create methods
  ValueTuple<int, string> x1 = ValueTuple.Create<int, string>(8, "ABC");

  // 隱式宣告使用 ValueTuple Constructor
  var x2 = (8, "ABC");

  // 具名宣告欄位名稱 (1)
  var x3 = (length: 8, letters: "ABC");

  // 具名宣告欄位名稱 (2)
  (int length, string letters) x4 = (8, "ABC");

  // 具名宣告欄位名稱 (3) -- 會以指派運算子的左邊為主
  (int length, string letters) x5 = (first: 8, second: "ABC");

  // 直接指派給區域變數
  (int length, string letters) = (8, "ABC");

  var (l, s) = (8, "ABC");
  ```

## 變數存放

* 實質型別存放於 Stack 記憶體，存取通常較快
* 參考型別存放於 Heap 記憶體，存取通常較慢

## Boxing and Unboxing

* 參考型別資料轉存至實質型別：Boxing
* 實質型別資料轉存至參考型別：Unboxing
* 永遠閃不掉的 Boxing : int.GetType()
* ArrayList 必定有此問題
* 問題成因：實質型別與參考型別的記憶體存放機制不同，因此會為了符合目標型別而有記憶體複製轉移等效能損耗

## 防禦性複製

* 只會發生在實質型別
* 資料會整包複製出來
* 建構式不會產生防禦性複製
* 要防止防禦性複製的話，需要使用唯讀結構，且所有成員都是唯讀
* 結構宣告非 readonly，但是使用時使用 readonly 變數承接，就會有防禦性複製

## 欄位 (Field)

* 物件內的全域變數
* 通常為私有成員 (若需要作為公開成員，通常會使用屬性)

## 屬性 (Property)

* 特定變數(欄位)的公開 Get / Set 方法
* 通常為公開成員
* 編譯後會自動長出私有欄位與 Get / Set 方法
    編譯前
    ```csharp
    public string SampleString { Get; Set; }
    ```

    編譯後 (僅示意，編譯完的實際 IL Code 並非如此)
    ```csharp
    private string _sampleString; //實際變數名稱由編譯器產出

    public string GetSampleString()
    {
        return _sampleString;
    }

    public void SetSampleString(string value)
    {
        _sampleString = value;
    }
    ```

## Nullable\<T\>

* T 被限制為結構 (ref : [microsoft learn - system.nullable](https://learn.microsoft.com/zh-tw/dotnet/api/system.nullable-1?view=net-8.0))
* 讓不能為 null 的結構偽裝為參考型別，變成可 null
* int? 實際上是 Nullable\<int\>，與 int 不同
* 判斷是否有值時，建議使用比較運算子，這樣在 LinQ to sql 時，出來的語法會比較漂亮
* Nullable\<T\>.Equals 僅有實作 Equals(Object)，並沒有實作 Equals(T)，無法避免 boxing 問題，因此在 Nullable\<T\> 變數上盡可能不要使用 Equals
* Nullable.GetUnderlyingType 用於確認該類型是不是 Nullable，在反射時會需要使用
* Nullable boxing 後沒有 Nullable，會呈現 Nullable\<T\> 的 T
  ```csharp
  int? I;
  Nullable.GetUnderlyingType(i.GetType());
  ```
* 結果會是 null，因為執行 GetType() 的時候會 boxing，而當下並沒有內容值，因此取得的是 null
* 如果要確實的拿到型別，給 GetUnderlyingType 的參數需要用 typeof(int?) 才能真的拿到 int

## 運算式主體

* 簡化方法的宣告

  ```csharp
  public string Something() => "";

  //等同以下寫法
  public string Something()
  {
      return "";
  }
  ```

## 參數關鍵子：ref / out / in

* ref: call by references
* out: 資料會回設定至該位置的外部變數，有 ValueTuple Structure 時就不是特別需要使用此關鍵字來傳出多項資料了
* in: 會進行防禦性複製，在方法內對該參數的調整不會反映到呼叫端
* 備註：不要過度倚賴 out / ref 來回傳資料，容易導致方法的職責不明確

## nameof()

* 取得成員名稱
* 編譯後會成為字串
* 可降低誤字問題

## 捨棄資料

```csharp
_ = GetSomething(); //GetSomething 方法所回傳的資料不會被使用
var _ = GetSomething(); //GetSomething 方法所回傳的資料會被存放於 _ 這個變數中
```

## 擴充方法

* 可以為特定型別加入功能
* 有撰寫限制，必須符合以下條件
  * 位於靜態類別
  * 方法必定為靜態
  * 第一個方法參數必須加上 this 關鍵字
* 設計擴充方法實應注意職責單一的問題
* 擴充方法內應盡可能降低外部依賴
* 擴充方法應為單純的處理邏輯
* 擴充方法應為無狀態方法 (不紀錄資料)

## 泛型的型別控制

```csharp
public class SampleInput<T> where T : ISampleClass
{
}
```

