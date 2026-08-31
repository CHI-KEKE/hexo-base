---
title: (NEW)EFCORE效能
date: 2024-07-07 18:25:05
categories: Others
top_img: https://i.imgur.com/I1A7fs1.png
cover : https://i.imgur.com/I1A7fs1.png
tags:
    - 
toc:
toc_number:
comments :
---
https://blog.darkthread.net/blog/ef-core-test-with-in-memory-db
單純 CRUD 用 EF (或自製 ORM) 享受強型別保護及不沾 SQL 的清爽，至於複雜查詢、批次更新刪除，則回歸自己寫 SQL 以確保執行效能。



案例 1. 叫用 csp 時
```CSHARP

        public virtual async Task<List<csp_GetPromotionEngineSpecialPriceDataResult>> csp_GetPromotionEngineSpecialPriceDataAsync(string queryType, string salePageIds, string skuIds, string promotionEngineIds, long? shopId, OutputParameter<int> returnValue = null, CancellationToken cancellationToken = default)
        {
            var parameterreturnValue = new SqlParameter
            {
                ParameterName = "returnValue",
                Direction = System.Data.ParameterDirection.Output,
                SqlDbType = System.Data.SqlDbType.Int,
            };

            var sqlParameters = new []
            {
                new SqlParameter
                {
                    ParameterName = "queryType",
                    Size = 20,
                    Value = queryType ?? Convert.DBNull,
                    SqlDbType = System.Data.SqlDbType.VarChar,
                },
                new SqlParameter
                {
                    ParameterName = "salePageIds",
                    Size = 500,
                    Value = salePageIds ?? Convert.DBNull,
                    SqlDbType = System.Data.SqlDbType.VarChar,
                },
                new SqlParameter
                {
                    ParameterName = "skuIds",
                    Size = 500,
                    Value = skuIds ?? Convert.DBNull,
                    SqlDbType = System.Data.SqlDbType.VarChar,
                },
                new SqlParameter
                {
                    ParameterName = "promotionEngineIds",
                    Size = 350,
                    Value = promotionEngineIds ?? Convert.DBNull,
                    SqlDbType = System.Data.SqlDbType.VarChar,
                },
                new SqlParameter
                {
                    ParameterName = "shopId",
                    Value = shopId ?? Convert.DBNull,
                    SqlDbType = System.Data.SqlDbType.BigInt,
                },
                parameterreturnValue,
            };
            var _ = await _context.SqlQueryAsync<csp_GetPromotionEngineSpecialPriceDataResult>("EXEC @returnValue = [dbo].[csp_GetPromotionEngineSpecialPriceData] @queryType, @salePageIds, @skuIds, @promotionEngineIds, @shopId", sqlParameters, cancellationToken);

            returnValue?.SetValue(parameterreturnValue.Value);

            return _;
        }

public static class DbContextExtensions
{
    public static async Task<List<T>> SqlQueryAsync<T>(this DbContext db, string sql, object[] parameters = null, CancellationToken cancellationToken = default) where T : class
    {
        if (parameters is null)
        {
            parameters = new object[] { };
        }

        if (typeof(T).GetProperties().Any())
        {
            return await db.Set<T>().FromSqlRaw(sql, parameters).ToListAsync(cancellationToken);
        }
        else
        {
            await db.Database.ExecuteSqlRawAsync(sql, parameters, cancellationToken);
            return default;
        }
    }
}
```