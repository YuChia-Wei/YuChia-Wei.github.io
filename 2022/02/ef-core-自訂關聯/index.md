# EF Core 自訂關聯


當 Db 內的 Table 都沒有設定關聯，又希望在不調整 Db 的情況下用操作有關連 Table 的方法使用 EF Core 時，可以在 EF Core 所使用的 Entity 與 Entity Configuration 中自訂關聯性，EF Core 會自行調整 Sql 語法 (使用 Left join 或其它 join 語法) 來加入資料。</br>

EF Power Tool 產生的 Entity / Entity Configuration 預設皆有引入 partial 關鍵字，因此，為了區別工具產生的 code 與自訂的 code，我們可以另外建立新的檔案處理自定義的部分。</br>

本篇範例為 一對一、一對多 的設定方式。</br>

{{<admonition warning >}}
TODO: 示範用程式碼待補 (2022-02-06)
{{</admonition >}}

<!--more-->

# 使用的專案結構

- Db.Repository 專案結構樹
    <details>
    <summary>結構樹</summary>

    ```
    .
    ├── Configurations
    ├── Entities
    ├── [db name]Context.cs
    └── efpt.config.json
    ```

    </details>

- 專案內部資料夾說明

    | 資料夾              | 說明                                                              |
    |---------------------|-------------------------------------------------------------------|
    | Configurations      | EF Power Tool 產生的 Entity 設定檔                                |
    | Entities            | EF Power Tool 產生的 Entity 檔                                    |
    | [db name]Context.cs | EF Core DbContext By Db                                           |
    | efpt.config.json    | EF Core Power Tool 設定檔，除了進行 Entity 增減外，不應有其它變更 |


**以下操作均使用 EF Core 5.0**

# Entity 調整

在 Db.Repository 專案中，找到想要加入自訂關聯的 Entity，並建立一個新的檔案，名為 {EntityName}.CustomProperty.cs。
註：新增的檔案後續的字節 (CustomProperty) 可以依據該檔案內放置的項目進行調整，本次以 CustomProperty 命名是因內容存放自定義屬性。

* EF Power Tool 產生的 Entity File
    ```cs
    //File: Db.Repository/Entities/MainTable.cs
    public partial class MainTable
    {
        public int MainId{ get; set; }
    }

    //File: Db.Repository/Entities/OneToManySubTable.cs
    public partial class OneToManySubTable
    {
        public int MainId{ get; set; }
        public int SubId{ get; set; }
    }

    //File: Db.Repository/Entities/OneToOneSubTable.cs
    public partial class OneToOneSubTable
    {
        public int MainId{ get; set; }
        public int SubId{ get; set; }
    }
    ```

* 手動新增的 Partial File
    ```cs
    //File: Db.Repository/Entities/MainTable.CustomProperty.cs
    public partial class MainTable
    {
        public MainTable()
        {
            this.OneToManySubItems = new HashSet<OneToManySubTable>();
        }

        public virtual ICollection<OneToManySubTable> OneToManySubItems { get; set; }

        public virtual OnoToOneSubTable OneToOneSubItem { get; set; }
    }

    //File: Db.Repository/Entities/OneToManySubTable.CustomProperty.cs
    public partial class OneToManySubTable
    {
        public virtual MainTable Header { get; set; }
    }

    //File: Db.Repository/Entities/OneToOneSubTable.CustomProperty.cs
    public partial class OneToOneSubTable
    {
        public virtual MainTable Header { get; set; }
    }
    ```

# Entity Configuration 調整

在 Db.Repository 專案中，找到想要加入自訂關聯的 Entity Configuration，並建立一個新的檔案，名為 {EntityName}Configuration.Partial.cs。
註：新增的檔案後續的字節 (Partial) 可以依據該檔案內放置的項目進行調整，本次以 Partial 命名是因 "關聯" 的英文過長，而採用較短且符合檔案目的的 Partial
* EF Power Tool 產生的 Entity Configuration
    ```cs
    //File: Db.Repository/Configurations/OneToOneSubTableConfiguration.cs
    public partial class OneToOneSubTableConfiguration : IEntityTypeConfiguration<OneToOneSubTable>
    {
        public void Configure(EntityTypeBuilder<OneToOneSubTable> entity)
        {
            entity.HasKey(e => e.SubId);

            entity.ToTable("OneToOneSubTable", "dbo");

            entity.HasComment("一對一子表");

            entity.Property(e => e.SubId).HasColumnName("SubId");

            this.OnConfigurePartial(entity);
        }

        partial void OnConfigurePartial(EntityTypeBuilder<OneToOneSubTable> entity);
    }

    //File: Db.Repository/Configurations/OneToManySubTableConfiguration.cs
    public partial class OneToManySubTableConfiguration : IEntityTypeConfiguration<OneToManySubTable>
    {
        public void Configure(EntityTypeBuilder<OneToManySubTable> entity)
        {
            entity.HasKey(e => e.SubId);

            entity.ToTable("OneToManySubTable", "dbo");

            entity.HasComment("一對多子表");

            entity.Property(e => e.SubId).HasColumnName("SubId");

            this.OnConfigurePartial(entity);
        }

        partial void OnConfigurePartial(EntityTypeBuilder<OneToManySubTable> entity);
    }
    ```

* 手動新增的 Partial File
    補充：OnDelete 的參數可參照 [微軟官方](https://docs.microsoft.com/zh-tw/dotnet/api/microsoft.entityframeworkcore.deletebehavior?view=efcore-5.0) 的說明調整設定
    ```cs
    //File: Db.Repository/Configurations/OneToOneSubTableConfiguration.Partial.cs
    public partial class OneToOneSubTableConfiguration : IEntityTypeConfiguration<OneToOneSubTable>
    {
        partial void OnConfigurePartial(EntityTypeBuilder<OneToOneSubTable> entity)
        {
            entity.HasOne(d => d.Header)                    //設定 OneToOneSubTable 這個 Entity Header 這個屬性是屬於外部資料
                  .WithOne(p => p.OneToOneSubItem)          //設定前一行 HasOne 中選擇的屬性 (Header) 的型別 (MainTable) 中的 OneToOneSubItem 屬性對應自己 (OneToOneSubTable)
                  .HasForeignKey(d => d.MainId)             //設定自己 (OneToOneSubTable) 的哪一個屬性是紀錄 Header 的 PK
                  .OnDelete(DeleteBehavior.ClientSetNull);  //設定資料刪除後的處理方式 (Header 被刪掉後是否要跟著被刪掉)
        }
    }

    //File: Db.Repository/Configurations/OneToManySubTableConfiguration.Partial.cs
    public partial class OneToManySubTableConfiguration : IEntityTypeConfiguration<OneToManySubTable>
    {
        partial void OnConfigurePartial(EntityTypeBuilder<OneToManySubTable> entity)
        {
            //各方法說明皆與 OneToOne 相同，只是因為 OneToMany 是一對多，所以第二行設定從 WithOne 改為 WithMany
            entity.HasOne(d => d.Header)
                  .WithMany(p => p.OneToManySubItems)
                  .HasForeignKey(d => d.SubId)
                  .OnDelete(DeleteBehavior.ClientSetNull);
        }
    }
    ```

# 使用方式

本章節提供三種 EF Core 操作 Db 的方法，可依需要自行選擇做法。
EF Core 中，觸發有關聯的 SubTable 需要調用 Include / ThenInclude 方法。

* 直接轉成輸出用的 Dto 物件寫法
    此寫法可自訂輸出欄位，減少 SqlServer <-> WebService 間的資料傳輸量
    ```cs
    private async Task<IEnumerable<MainTableDto>> GetAllMainTableDtos()
    {

        return await this._testContext.MainTables
                            .AsNoTracking()
                            .Get()
                            .Include(o=>o.OneToManySubItems)
                            .Include(o=>o.OneToOneSubItem)
                            .Select(o => new MainTableDto
                            {
                                MainId = o.MainId,

                                // 這邊允許使用 AutoMapper 直接映照欄位，如註解，但若 Sub Table 中只需要特定欄位的話，則建議使用此處範例使用的寫法
                                // OneToManySubItems = this._mapper.Map<IEnumerable<OneToManySubTableDto>>(o.OneToManySubItems)
                                OneToManySubItems = o.OneToManySubItems.Select(m=>new OneToManySubTableDto
                                {
                                    SubId = m.SubId,
                                    MainId = m.MainId
                                }),

                                // OneToOneSubItem = this._mapper.Map<OneToOneSubTableDto>(o.OneToOneSubItem)
                                OneToOneSubItem = new OneToOneSubTableDto
                                {
                                    SubId = o.OneToOneSubItem.SubId,
                                    MainId = o.OneToOneSubItem.MainId
                                })
                            }).ToListAsync();
    }
    ```

* 完整依據 Table Entity 內容轉出 Db 資料，再自行組合 Service 輸出用的 Dto 物件寫法
    此寫法適合在需要一些額外的欄位判斷事情，但是那些欄位又不需要輸出的情境
    ```cs
    private async Task<IEnumerable<MainTableDto>> GetAllMainTableDtos()
    {
        var mainTableList = await this._testContext.MainTables
                                            .AsNoTracking()
                                            .Get()
                                            .Include(o=>o.OneToManySubItems)
                                            .Include(o=>o.OneToOneSubItem)
                                            .ToListAsync();

        // 若無其它需求，可直接使用 AutoMapper 進行資料映照並轉出
        return this._mapper.Map<IEnumerable<MainTableDto>>(mainTableList);

        //==============================

        // 不使用 AutoMapper 進行資料設定的方式，彈性較高，但是撰寫較麻煩
        return mainTableList.Select(o => new MainTableDto
                            {
                                MainId = o.MainId,

                                // 這邊允許使用 AutoMapper 直接映照欄位，如註解，但若 Sub Table 中只需要特定欄位的話，則建議使用此處範例使用的寫法
                                // OneToManySubItems = this._mapper.Map<IEnumerable<OneToManySubTableDto>>(o.OneToManySubItems)
                                OneToManySubItems = o.OneToManySubItems.Select(m=>new OneToManySubTableDto
                                {
                                    SubId = m.SubId,
                                    MainId = m.MainId
                                }),

                                // OneToOneSubItem = this._mapper.Map<OneToOneSubTableDto>(o.OneToOneSubItem)
                                OneToOneSubItem = new OneToOneSubTableDto
                                {
                                    SubId = o.OneToOneSubItem.SubId,
                                    MainId = o.OneToOneSubItem.MainId
                                })
                            }).ToListAsync();
    }
    ```

* 前述兩種方式的混和寫法，動態調整 Db 出來的欄位，在依據需要組合成輸出用的 Dto
    此寫法彈性高，惟程式碼較複雜
    ```cs
    private async Task<IEnumerable<MainTableDto>> GetAllMainTableDtos()
    {
        var mainTableList = await this._testContext.MainTables
                                            .AsNoTracking()
                                            .Get()
                                            .Include(o=>o.OneToManySubItems)
                                            .Include(o=>o.OneToOneSubItem)
                                            .Select(o=> new MainTable()
                                            {
                                                MainId = o.MainId,
                                                OneToOneSubItem = o.OneToOneSubItem
                                                //這邊不再另外設定 OneToMainSubItem 欄位，這樣觸發的 Sql 就會掠過 OneToOneSubItem 這個欄位
                                            })
                                            .ToListAsync();

        // 若無其它需求，可直接使用 AutoMapper 進行資料映照並轉出
        return this._mapper.Map<IEnumerable<MainTableDto>>(mainTableList);

        //==============================

        // 不使用 AutoMapper 進行資料設定的方式，彈性較高，但是撰寫較麻煩
        return mainTableList.Select(o => new MainTableDto
                            {
                                MainId = o.MainId,

                                // 因為前面沒有把 OneToManySubItem 叫出來，這邊就不設定 OneToManySubItem
                                // 若仍保留 OneToManySubItem 欄位的設定，則必須注意 null 值問題

                                // OneToOneSubItem = this._mapper.Map<OneToOneSubTableDto>(o.OneToOneSubItem)
                                OneToOneSubItem = new OneToOneSubTableDto
                                {
                                    SubId = o.OneToOneSubItem.SubId,
                                    MainId = o.OneToOneSubItem.MainId
                                })
                            }).ToListAsync();
    }
    ```

