## 提交任务执行、获取执行结果

```
static CompletableFuture<Void> runAsync(Runnable runnable) 
//1. 返回一个新的CompletableFuture，它在运行给定操作后由运行在 ForkJoinPool.commonPool()中的任务 异步完成。 
 
static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor) 
//2. 返回一个新的CompletableFuture，它在运行给定操作之后由在给定执行程序中运行的任务异步完成。  

static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) 
//3. 返回一个新的CompletableFuture，它通过在 ForkJoinPool.commonPool()中运行的任务与通过调用给定的供应商获得的值 异步完成。  

static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor) 
//4. 返回一个新的CompletableFuture，由给定执行器中运行的任务异步完成，并通过调用给定的供应商获得的值。 
```

## allOf 、 anyOf

```
static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) 
//5. 返回一个新的CompletableFuture，当所有给定的CompletableFutures完成时，完成。  

static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) 
//6. 返回一个新的CompletableFuture，当任何一个给定的CompletableFutures完成时，完成相同的结果。  
```

## 异步结果的处理

```
<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn) 
//7.返回一个新的CompletionStage，当此阶段正常完成时，将以该阶段的结果作为所提供函数的参数执行。 
 
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn) 
//8. 返回一个新的CompletionStage，当该阶段正常完成时，将使用此阶段的默认异步执行工具执行此阶段的结果作为所提供函数的参数。
  
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor) 
//9. 返回一个新的CompletionStage，当此阶段正常完成时，将使用提供的执行程序执行此阶段的结果作为提供函数的参数。  

```

```
CompletableFuture<Void> thenAccept(Consumer<? super T> action) 
//10. 返回一个新的CompletionStage，当此阶段正常完成时，将以该阶段的结果作为提供的操作的参数执行。
  
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action) 
//11. 返回一个新的CompletionStage，当此阶段正常完成时，将使用此阶段的默认异步执行工具执行，此阶段的结果作为提供的操作的参数。  

CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor) 
//12. 返回一个新的CompletionStage，当此阶段正常完成时，将使用提供的执行程序执行此阶段的结果作为提供的操作的参数。
```

## 任务串行执行,不处理上一步执行结

```
CompletableFuture<Void> thenRun(Runnable action) 
//13. 返回一个新的CompletionStage，当此阶段正常完成时，执行给定的操作。  

CompletableFuture<Void> thenRunAsync(Runnable action) 
//14. 返回一个新的CompletionStage，当此阶段正常完成时，使用此阶段的默认异步执行工具执行给定的操作。 
 
CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor) 
//15. 返回一个新的CompletionStage，当此阶段正常完成时，使用提供的执行程序执行给定的操作。
```

## 合并2个结果

```
<U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn) 
//16. 返回一个新的CompletionStage，当这个和另一个给定的阶段都正常完成时，两个结果作为提供函数的参数执行。
  
<U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn) 
//17. 返回一个新的CompletionStage，当这个和另一个给定阶段正常完成时，将使用此阶段的默认异步执行工具执行，其中两个结果作为提供函数的参数。  

<U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor) 
//18. 返回一个新的CompletionStage，当这个和另一个给定阶段正常完成时，使用提供的执行器执行，其中两个结果作为提供的函数的参数。  
```