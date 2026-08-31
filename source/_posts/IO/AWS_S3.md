

不用 MemoryStream，用 StringContent 更省記憶體（推薦）

```csharp
private async Task UpdateS3DataAsync(string s3RecordData, string s3Path)
{
    var putRequest = new PutObjectRequest
    {
        BucketName = _bucketName,
        Key = s3Path,
        ContentBody = s3RecordData,   // ⭐不用手動 new MemoryStream()
        ContentType = "application/json"
    };

    var response = await _amazonS3.PutObjectAsync(putRequest);

    if (response.HttpStatusCode != HttpStatusCode.OK)
        throw new ApplicationException($"S3 上傳失敗: {response.HttpStatusCode}");

    _logger.LogInformation("S3 備份成功：{Path}", s3Path);
}
```

1. 加上輸入參數防呆

避免空字串造成 PutObject 失敗或產生奇怪的 S3 檔案。

2. 使用 await using

PutObjectAsync 不會自動 dispose InputStream
→ async 流程要配合 await using 確保 memory 不會積累。

3. 指定 ContentType（尤其是 JSON）

AWS console + CDN 都會正確處理格式。

4. AutoCloseStream = true

簡化資源管理，避免誤用造成記憶體持續佔用。

5. 只 Log 重要資訊

ResponseMetadata 印整包太大且沒必要，只需要 RequestId。


如果你的 s3RecordData 很大（例如 > 5MB），MemoryStream 會造成 GC 壓力

✔ 少一份 buffer
✔ 減少 GC
✔ 更乾淨
✔ 適合 JSON / Text 類型內容



✔ 加 Retry（網路不穩時 PutObject 常失敗）

var retryPolicy = Policy
    .Handle<AmazonS3Exception>()
    .Or<HttpRequestException>()
    .WaitAndRetryAsync(3, retry => TimeSpan.FromSeconds(2));

await retryPolicy.ExecuteAsync(async () =>
{
    var response = await _amazonS3.PutObjectAsync(request);
    if (response.HttpStatusCode != HttpStatusCode.OK)
        throw new Exception("S3 upload failed");
});


