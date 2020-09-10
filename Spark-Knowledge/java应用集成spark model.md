```
    val model = NaiveBayesModel.load("")
    // 反射调用方法
    val predictRaw = model.getClass.getMethod("predictRaw", classOf[Vector]).invoke(model, vec).asInstanceOf[Vector]
    val raw2probabilityMethod = if ("".startsWith("2.3")) "raw2probabilityInPlace" else "raw2probability"
    val raw2probability = model.getClass.getMethod(raw2probabilityMethod, classOf[Vector]).invoke(model, predictRaw).asInstanceOf[Vector]
    // 获取分类id
    val categoryId = raw2probability.argmax
    
```