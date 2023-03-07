# UObject

## Construction
1. GENERRATED_BODY() only
    UHT generate UMyObject(FObjectInitializer){}

2. GENERRATED_BODY() + UMyObject(){};
    自定义构造函数 UHT不生成

3. GENERRATED_BODY() + UMyObject(int i){};
    UHT不生成+需要定义空构造函数或(FObjectInitializer)构造函数