

```csharp

public static async Task TaskStatusTest()
{
    var cts = new CancellationTokenSource();
    var token = cts.Token;
    System.Console.WriteLine("Please Choose a Status : A : Completed, B : Fault : C : Cancel");

    try
    {
        var result = await Task.Run(async () =>
        {
            var inputTask = Task.Run(() => System.Console.ReadKey()); // 讓他跑, 不 await
            for (int i = 0; i < 5; i++)
            {
                System.Console.WriteLine($"Waiting for the :{i + 1} sec...");
                var deplayTask = Task.Delay(1000); // 不 await void
                var completedTask = await Task.WhenAny(inputTask, deplayTask); // await 取回 真正先完成的那個 Task, 而非裁判自己

                if (completedTask == inputTask)
                {
                    var userInputResult = inputTask.Result;
                    switch (userInputResult.Key)
                    {
                        case ConsoleKey.A:
                            return "OK";
                        case ConsoleKey.B:
                            throw new ApplicationException("MyMyException");
                        case ConsoleKey.C:
                            cts.Cancel();
                            token.ThrowIfCancellationRequested();
                            return "Cancel";
                        default:
                            return "Unknown Input";
                    }
                }
            }

            return "No Input!";
        }, token);

        System.Console.WriteLine("Completed! Result={0}", result);
    }
    catch (OperationCanceledException)
    {
        System.Console.WriteLine("Canceled!");
    }
    catch (Exception ex)
    {
        System.Console.WriteLine("Faulted!");
        System.Console.WriteLine("Error: {0}", ex.Message);
    }
    finally
    {
        System.Console.WriteLine("Async Run...");
    }
}
```