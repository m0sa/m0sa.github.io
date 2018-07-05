+++
title = "Async Producer-Consumer with Linq, IEnumerable<T> and Threading.Tasks"
description = "Async Producer-Consumer with Linq, IEnumerable<T> and Threading.Tasks"
date = "2011-03-09"
categories = [ "Development" ]
+++

Lately I've been dealing a lot with IEnumerable<T> classes and the yield
return keyword. I found it really useful for stream like processing of
otherwise really memory consuming operations. The workflow looks like this:

```
\- Retrieve Item 1 _(yield return)
_\- Process Item 1 _(GetNext())
_\- …
\- Retrieve Item N
\- Process Item N
```

The approximate time it takes to process the whole operation is:

```
N x AvgTime(Retrieve) + N x AvgTime(Process)
```

So I was thinking how to parallelize the whole operation, preferably in a Linq
fashion, so it could fit into my existing code. Here is the result, using the
(relatively) new System.Collections.Concurrent namespace and the Task Parallel
Library:

```csharp
public static IEnumerable<TResult> SelectAsync<TProduct, TResult>(
    this IEnumerable<TProduct> producer,
    Func<TProduct, TResult> consumer,
    int? productsBoundedCapacity = null,
    int? resultsBoundedCapacity = null)
{
    var productsQueue = productsBoundedCapacity.HasValue
        ? new BlockingCollection<TProduct>(productsBoundedCapacity.Value)
        : new BlockingCollection<TProduct>();
    var resultsQueue = resultsBoundedCapacity.HasValue
        ? new BlockingCollection<Tuple<int, TResult>>(resultsBoundedCapacity.Value)
        : new BlockingCollection<Tuple<int, TResult>>();
    Task.Factory.StartNew(() =>
    {
        foreach (var product in producer)
        {
            productsQueue.Add(product);
        }
        productsQueue.CompleteAdding();
    });
    Task.Factory.StartNew(() =>
    {
        var tasks = productsQueue
            .GetConsumingEnumerable()
            .Select((product, ind) => Task.Factory.StartNew(() =>
                    resultsQueue.Add(new Tuple<int, TResult>(ind, consumer(product)))))
            .ToArray();
        Task.WaitAll(tasks);
        resultsQueue.CompleteAdding();
    });
    return resultsQueue
        .GetConsumingEnumerable()
        .OrderBy(x => x.Item1)
        .Select(x => x.Item2);
}
```

Note however, that the producer part parallelizes correctly only with the
usage of true enumerables that yield return the long running operation result
(Arrays, List etc. already contain all the data). Here is a small proof of
concept app:

```
class Program
{
    private static IEnumerable<int> Producer()
    {
        for (int i = 0; i < 100; i++)
        {
            Thread.Sleep(50); // long running operation
            yield return i;
        }
    }

    private static int Consumer(int item)
    {
        Thread.Sleep(100); // long running operation
        return item * item;
    }

    static void Main()
    {
        var sw = new Stopwatch();
        sw.Start();
        var resultSync = Producer().Select(Consumer).Sum();
        sw.Stop();
        Console.WriteLine("Result: {0},  {1}ms", resultSync, sw.ElapsedMilliseconds);
        sw.Reset();

        sw.Start();
        var resultAsync = Producer().SelectAsync(Consumer).Sum();
        sw.Stop();
        Console.WriteLine("Result: {0},  {1}ms", resultAsync, sw.ElapsedMilliseconds);
        Console.ReadLine();
    }
}
```
