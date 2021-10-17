# Il2cpp 源码阅读笔记

## Il2cpp 的全局初始化

程序有全局变量 `static il2cpp::utils::RegisterRuntimeInitializeAndCleanup s_Il2CppCodegenRegistrationVariable (&s_Il2CppCodegenRegistration, NULL);`

构造函数指针位于`.rdata`内，构造函数名称(未导出)`dynamic_initializer_for__s_Il2CppCodegenRegistrationVariable__`

该构造函数传入一个函数指针指向`s_Il2CppCodegenRegistration`，这是初始化两个重要全局变量的函数，之后会被调用到

```cpp
// IDA
void dynamic_initializer_for__s_Il2CppCodegenRegistrationVariable__()
{
  il2cpp::utils::RegisterRuntimeInitializeAndCleanup::RegisterRuntimeInitializeAndCleanup(
    &s_Il2CppCodegenRegistrationVariable,
    s_Il2CppCodegenRegistration,
    0i64,
    0);
}
```

```cpp
// utils\RegisterRuntimeInitializeAndCleanup.cpp
namespace il2cpp
{
namespace utils
{
    typedef std::set<RegisterRuntimeInitializeAndCleanup::CallbackFunction> RegistrationCallbackSet;

    static RegistrationCallbackSet* _registrationCallbacks = NULL;

    RegisterRuntimeInitializeAndCleanup::RegisterRuntimeInitializeAndCleanup(CallbackFunction Initialize, CallbackFunction Cleanup, int order)
    {
        if (!_registrationCallbacks)
            _registrationCallbacks = new RegistrationCallbackSet();
        (*_registrationCallbacks).insert(Initialize);
    }

    void RegisterRuntimeInitializeAndCleanup::ExecuteInitializations()
    {
        if (_registrationCallbacks == NULL)
            return;

        for (RegistrationCallbackSet::iterator iter = (*_registrationCallbacks).begin(); iter != (*_registrationCallbacks).end(); ++iter)
        {
            (*iter)();
        }
    }

    void RegisterRuntimeInitializeAndCleanup::ExecuteCleanup()
    {
        IL2CPP_ASSERT(0);
    }
} /* namespace vm */
} /* namespace utils */
```

函数`RegisterRuntimeInitializeAndCleanup`的主要作用就是把初始化函数放入`_registrationCallbacks`这个set中，其他什么事都不干，甚至还有两个无用的参数

这里注册的函数什么时候被调用呢，由`il2cpp_init()->il2cpp::vm::Runtime::Init(domain_name, "v2.0.50727");->il2cpp::utils::RegisterRuntimeInitializeAndCleanup::ExecuteInitializations()` 然后就可以来到上述代码的第二个函数，函数内遍历整个set并分别调用每一个初始化函数



## 细究 s_Il2CppCodegenRegistration()

```cpp
void s_Il2CppCodegenRegistration()
{
	il2cpp_codegen_register (&g_CodeRegistration, &g_MetadataRegistration, &s_Il2CppCodeGenOptions);
}
```

整个代码就执行一个事情，那就是注册了三个全局变量，这也应该是il2cpp最重要的三个全局变量

往下跟一下：

```cpp
inline void il2cpp_codegen_register(const Il2CppCodeRegistration* const codeRegistration, const Il2CppMetadataRegistration* const metadataRegistration, const Il2CppCodeGenOptions* const codeGenOptions)
{
    il2cpp::vm::MetadataCache::Register(codeRegistration, metadataRegistration, codeGenOptions);
}
```

```cpp
void MetadataCache::Register(const Il2CppCodeRegistration* const codeRegistration, const Il2CppMetadataRegistration* const metadataRegistration, const Il2CppCodeGenOptions* const codeGenOptions)
{
    s_Il2CppCodeRegistration = codeRegistration;
    s_Il2CppMetadataRegistration = metadataRegistration;
    s_Il2CppCodeGenOptions = codeGenOptions;

    for (int32_t j = 0; j < metadataRegistration->genericClassesCount; j++)
        if (metadataRegistration->genericClasses[j]->typeDefinitionIndex != kTypeIndexInvalid)
            metadata::GenericMetadata::RegisterGenericClass(metadataRegistration->genericClasses[j]);

    for (int32_t i = 0; i < metadataRegistration->genericInstsCount; i++)
        s_GenericInstSet.insert(metadataRegistration->genericInsts[i]);

    s_InteropData.assign_external(codeRegistration->interopData, codeRegistration->interopDataCount);
}
```

其实就是把`s_Il2CppCodeRegistration`,`s_Il2CppMetadataRegistration`,`s_Il2CppCodeGenOptions`赋值成了传入的参数，然后在这里注册了一些泛型的classes，关于注册过程这里暂时就不深究了

## 细究 g_CodeRegistration

其类型是`Il2CppCodeRegistration`

```cpp
// il2cpp-class-internals.h
typedef struct Il2CppCodeRegistration
{
    uint32_t methodPointersCount;
    const Il2CppMethodPointer* methodPointers;
    uint32_t reversePInvokeWrapperCount;
    const Il2CppMethodPointer* reversePInvokeWrappers;
    uint32_t genericMethodPointersCount;
    const Il2CppMethodPointer* genericMethodPointers;
    uint32_t invokerPointersCount;
    const InvokerMethod* invokerPointers;
    CustomAttributeIndex customAttributeCount;
    const CustomAttributesCacheGenerator* customAttributeGenerators;
    uint32_t unresolvedVirtualCallCount;
    const Il2CppMethodPointer* unresolvedVirtualCallPointers;
    uint32_t interopDataCount;
    Il2CppInteropData* interopData;
} Il2CppCodeRegistration;
```

定义在用户生成cpp代码的`Il2CppCodeRegistration.cpp`文件里

```cpp
// Il2CppCodeRegistration.cpp
extern const Il2CppCodeRegistration g_CodeRegistration = 
{
	8469,
	g_MethodPointers,
	0,
	NULL,
	2351,
	g_Il2CppGenericMethodPointers,
	1268,
	g_Il2CppInvokerPointers,
	2226,
	g_AttributeGenerators,
	202,
	g_UnresolvedVirtualMethodPointers,
	119,
	g_Il2CppInteropData,
};
```

这里注册了一堆的指针，根据名称推断，含义应该是如下的

`g_MethodPointers` : 所有函数的指针

`g_Il2CppGenericMethodPointers` : 所有泛型函数的指针

`g_Il2CppInvokerPointers` : ???

`g_AttributeGenerators` : 所有类的构造函数指针

`g_UnresolvedVirtualMethodPointers` : 未解析的虚函数指针

`g_Il2CppInteropData` : 反射相关的数据?

接下来点开每一个细看看

### g_MethodPointers

```cpp
// Il2CppMethodPointerTable.cpp
extern const Il2CppMethodPointer g_MethodPointers[8469] = 
{
	Locale_GetText_m3374010885,
	Locale_GetText_m1601577974,
	SafeHandleZeroOrMinusOneIsInvalid__ctor_m2667299826,
	SafeHandleZeroOrMinusOneIsInvalid_get_IsInvalid_m1185299356,
	SafeWaitHandle__ctor_m3710504225,
	SafeWaitHandle_ReleaseHandle_m2890681297,
	CodePointIndexer__ctor_m2813317897,
	CodePointIndexer_ToIndex_m1008730487,
	TableRange__ctor_m3039750162_AdjustorThunk,
	Contraction__ctor_m2731863112,
	ContractionComparer__ctor_m3439667810,
	ContractionComparer__cctor_m1682260389,
	ContractionComparer_Compare_m732151595,
	Level2Map__ctor_m3459390739,
	//......
	FieldWithTarget_set_doStatic_m3781634168,
	FieldWithTarget_get_staticString_m2597372683,
	FieldWithTarget_set_staticString_m3203090702,
	FieldWithTarget_GetValue_m2315870500,
	TrackablePropertyBase__ctor_m2201057683,
	TrackableTrigger__ctor_m4147744174,
	TriggerListContainer__ctor_m1390596431,
	TriggerListContainer_get_rules_m110726358,
	TriggerListContainer_set_rules_m474815225,
	TriggerMethod__ctor_m1863255651,
	TriggerRule__ctor_m225466603,
	TriggerRule_Test_m2595382785,
	TriggerRule_Test_m3134988565,
	TriggerRule_TestByObject_m3099447300,
	TriggerRule_TestByEnum_m2798194165,
	TriggerRule_TestByString_m2890887363,
	TriggerRule_TestByBool_m4144604729,
	TriggerRule_TestByDouble_m960810189,
	TriggerRule_SafeEquals_m2862813043,
	TriggerRule_GetDouble_m2201978033,
	ValueProperty__ctor_m2723448180,
	ValueProperty_get_valueType_m2516961412,
	ValueProperty_set_valueType_m3307530546,
	ValueProperty_get_propertyValue_m3340090327,
	ValueProperty_get_target_m1728151320,
	ValueProperty_IsValid_m3724034537,
};

```

确实是所有函数的指针

跟踪IDA也可以发现类似的结果

```
.rdata:00000001803F61B0 ; Il2CppCodeRegistration g_CodeRegistration
.rdata:00000001803F61B0 ?g_CodeRegistration@@3UIl2CppCodeRegistration@@B dq 2115h
.rdata:00000001803F61B0                                         ; DATA XREF: s_Il2CppCodegenRegistration(void)+E↑o
.rdata:00000001803F61B8                 dq offset ?g_MethodPointers@@3QBQ6AXXZB ; void (*const near * const g_MethodPointers)(void)
.rdata:00000001803F61C0                 dq 0
.rdata:00000001803F61C8                 dq 0
.rdata:00000001803F61D0                 dq 92Fh
.rdata:00000001803F61D8                 dq offset ?g_Il2CppGenericMethodPointers@@3QBQ6AXXZB ; void (*const near * const g_Il2CppGenericMethodPointers)(void)
.rdata:00000001803F61E0                 dq 4F4h
.rdata:00000001803F61E8                 dq offset ?g_Il2CppInvokerPointers@@3QBQ6APEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@ZB ; void * (*const near * const g_Il2CppInvokerPointers)(void (*)(void),MethodInfo const *,void *,void * *)
.rdata:00000001803F61F0                 dq 8B2h
.rdata:00000001803F61F8                 dq offset ?g_AttributeGenerators@@3QBQ6AXPEAUCustomAttributesCache@@@ZB ; void (*const near * const g_AttributeGenerators)(CustomAttributesCache *)
.rdata:00000001803F6200                 dq 0CAh
.rdata:00000001803F6208                 dq offset ?g_UnresolvedVirtualMethodPointers@@3QBQ6AXXZB ; void (*const near * const g_UnresolvedVirtualMethodPointers)(void)
.rdata:00000001803F6210                 dq 77h
.rdata:00000001803F6218                 dq offset ?g_Il2CppInteropData@@3PAUIl2CppInteropData@@A ; Il2CppInteropData near * g_Il2CppInteropData
```

```
.rdata:000000018037CBF0 ?g_MethodPointers@@3QBQ6AXXZB dq offset Convert_ToInt64_m2075162888, offset Locale_GetText_m1601577974
.rdata:000000018037CBF0                                         ; DATA XREF: .rdata:00000001803F61B8↓o
.rdata:000000018037CBF0                 dq offset SafeHandleZeroOrMinusOneIsInvalid__ctor_m2667299826 ; ChannelData_t3353629972::get_Ref_0(void) ...
.rdata:000000018037CBF0                 dq offset SafeHandleZeroOrMinusOneIsInvalid_get_IsInvalid_m1185299356
.rdata:000000018037CBF0                 dq offset SafeWaitHandle__ctor_m3710504225, offset SafeWaitHandle_ReleaseHandle_m2890681297
.rdata:000000018037CBF0                 dq offset CodePointIndexer__ctor_m2813317897, offset CodePointIndexer_ToIndex_m1008730487
.rdata:000000018037CBF0                 dq offset TableRange__ctor_m3039750162_AdjustorThunk, offset DictionaryNode__ctor_m1380016344
.rdata:000000018037CBF0                 dq offset RemotingSurrogateSelector__ctor_m1846610173
.rdata:000000018037CBF0                 dq offset ContractionComparer__cctor_m1682260389, offset ContractionComparer_Compare_m732151595
.rdata:000000018037CBF0                 dq offset Level2Map__ctor_m3459390739, offset RemotingSurrogateSelector__ctor_m1846610173
.rdata:000000018037CBF0                 dq offset Level2MapComparer__cctor_m1866197409, offset Level2MapComparer_Compare_m2874495629
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable__cctor_m2887118684, offset MSCompatUnicodeTable_GetTailoringInfo_m1575560208
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_BuildTailoringTables_m1316979344
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_SetCJKReferences_m2637101499
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_Category_m1834196420, offset MSCompatUnicodeTable_Level1_m18730923
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_Level2_m3823292331, offset MSCompatUnicodeTable_Level3_m1870873670
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_IsIgnorable_m3957534007
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_IsIgnorableNonSpacing_m47098938
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_ToKanaTypeInsensitive_m2886449430
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_ToWidthCompat_m3110108204
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_HasSpecialWeight_m1621324272
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_IsHalfWidthKana_m4030661976
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_IsHiragana_m3884380055
.rdata:000000018037CBF0                 dq offset MSCompatUnicodeTable_IsJapaneseSmallLetter_m2666144582
...
```

所以可以看出这个表的特征是，表内有很多连续的函数指针，一个接着一个

### g_Il2CppGenericMethodPointers

```cpp
// Il2CppGenericMethodPointerTable.cpp
extern const Il2CppMethodPointer g_Il2CppGenericMethodPointers[2351] = 
{
	NULL/* 0*/,
	(Il2CppMethodPointer)&Array_InternalArray__IEnumerable_GetEnumerator_TisRuntimeObject_m3132609973_gshared/* 1*/,
	(Il2CppMethodPointer)&Array_InternalArray__ICollection_Add_TisRuntimeObject_m4216329873_gshared/* 2*/,
	(Il2CppMethodPointer)&Array_InternalArray__ICollection_Remove_TisRuntimeObject_m2110193223_gshared/* 3*/,
	(Il2CppMethodPointer)&Array_InternalArray__ICollection_Contains_TisRuntimeObject_m4067783231_gshared/* 4*/,
	(Il2CppMethodPointer)&Array_InternalArray__ICollection_CopyTo_TisRuntimeObject_m4245759982_gshared/* 5*/,
	(Il2CppMethodPointer)&Array_InternalArray__Insert_TisRuntimeObject_m1619219378_gshared/* 6*/,
	(Il2CppMethodPointer)&Array_InternalArray__IndexOf_TisRuntimeObject_m2971736253_gshared/* 7*/,
	(Il2CppMethodPointer)&Array_InternalArray__get_Item_TisRuntimeObject_m3347010206_gshared/* 8*/,
	(Il2CppMethodPointer)&Array_InternalArray__set_Item_TisRuntimeObject_m2895257685_gshared/* 9*/,
	(Il2CppMethodPointer)&Array_get_swapper_TisRuntimeObject_m1378919517_gshared/* 10*/,
	(Il2CppMethodPointer)&Array_Sort_TisRuntimeObject_m1972115694_gshared/* 11*/,
	(Il2CppMethodPointer)&Array_Sort_TisRuntimeObject_TisRuntimeObject_m1685639929_gshared/* 12*/,
	(Il2CppMethodPointer)&Array_Sort_TisRuntimeObject_m460813780_gshared/* 13*/,
	//......
	(Il2CppMethodPointer)&UnityAction_2__ctor_m2941677221_gshared/* 2344*/,
	(Il2CppMethodPointer)&UnityAction_2_BeginInvoke_m1733258791_gshared/* 2345*/,
	(Il2CppMethodPointer)&UnityAction_2_EndInvoke_m2385586247_gshared/* 2346*/,
	(Il2CppMethodPointer)&UnityEvent_1_RemoveListener_m1953458448_gshared/* 2347*/,
	(Il2CppMethodPointer)&UnityEvent_1_FindMethod_Impl_m1397247356_gshared/* 2348*/,
	(Il2CppMethodPointer)&UnityEvent_1_GetDelegate_m617150804_gshared/* 2349*/,
	(Il2CppMethodPointer)&UnityEvent_1_GetDelegate_m2283422164_gshared/* 2350*/,
};
```

也是一堆函数指针构成的表

跟踪IDA发现类似结果

```
.rdata:00000001803ABF30 ?g_Il2CppGenericMethodPointers@@3QBQ6AXXZB dq 0
.rdata:00000001803ABF30                                         ; DATA XREF: .rdata:00000001803F61D8↓o
.rdata:00000001803ABF38                 dq offset Array_InternalArray__IEnumerable_GetEnumerator_TisWorkRequest_t1354518612_m2622205355_gshared
.rdata:00000001803ABF40                 dq offset Array_InternalArray__ICollection_Add_TisRuntimeObject_m4216329873_gshared
.rdata:00000001803ABF48                 dq offset Array_InternalArray__ICollection_Remove_TisRuntimeObject_m2110193223_gshared
.rdata:00000001803ABF50                 dq offset Array_InternalArray__ICollection_Contains_TisRuntimeObject_m4067783231_gshared
.rdata:00000001803ABF58                 dq offset Array_InternalArray__ICollection_CopyTo_TisRuntimeObject_m4245759982_gshared
.rdata:00000001803ABF60                 dq offset Array_InternalArray__Insert_TisRuntimeObject_m1619219378_gshared
.rdata:00000001803ABF68                 dq offset Array_InternalArray__IndexOf_TisRuntimeObject_m2971736253_gshared
.rdata:00000001803ABF70                 dq offset Array_InternalArray__get_Item_TisRuntimeObject_m3347010206_gshared
.rdata:00000001803ABF78                 dq offset Array_InternalArray__set_Item_TisRuntimeObject_m2895257685_gshared
```

值得注意的一点是他的第一个函数指针始终是0，这是特例还是普遍情况呢？值得商议

### g_Il2CppInvokerPointers

```cpp
extern const InvokerMethod g_Il2CppInvokerPointers[1268] = 
{
	RuntimeInvoker_Void_t1185182177,
	RuntimeInvoker_Boolean_t97287965_RuntimeObject,
	RuntimeInvoker_Boolean_t97287965_RuntimeObject_RuntimeObject,
	RuntimeInvoker_Int32_t2950945753,
	RuntimeInvoker_RuntimeObject,
	RuntimeInvoker_Int32_t2950945753_RuntimeObject,
	RuntimeInvoker_Boolean_t97287965_RuntimeObject_RuntimeObject_ObjectU5BU5DU26_t712384779,
	RuntimeInvoker_Int32_t2950945753_RuntimeObject_ObjectU5BU5DU26_t712384779,
	RuntimeInvoker_Void_t1185182177_RuntimeObject_RuntimeObject,
	RuntimeInvoker_RuntimeObject_RuntimeObject_RuntimeObject,
	RuntimeInvoker_RuntimeObject_RuntimeObject_RuntimeObject_SByte_t1669577662,
	RuntimeInvoker_Boolean_t97287965_RuntimeObject_RuntimeObject_SByte_t1669577662,
	//.......
	RuntimeInvoker_Enumerator_t1862690208,
	RuntimeInvoker_CustomAttributeNamedArgument_t287865710_RuntimeObject,
	RuntimeInvoker_CustomAttributeTypedArgument_t2723150157_RuntimeObject,
	RuntimeInvoker_Boolean_t97287965_Nullable_1_t2603721331,
	RuntimeInvoker_RuntimeObject_CustomAttributeNamedArgument_t287865710_RuntimeObject_RuntimeObject,
	RuntimeInvoker_RuntimeObject_CustomAttributeTypedArgument_t2723150157_RuntimeObject_RuntimeObject,
	RuntimeInvoker_RuntimeObject_OrderBlock_t1585977831_RuntimeObject_RuntimeObject,
	RuntimeInvoker_RuntimeObject_Single_t1397266774_RuntimeObject_RuntimeObject,
	RuntimeInvoker_RuntimeObject_Scene_t2348375561_RuntimeObject_RuntimeObject,
	RuntimeInvoker_RuntimeObject_Scene_t2348375561_Int32_t2950945753_RuntimeObject_RuntimeObject,
	RuntimeInvoker_RuntimeObject_Scene_t2348375561_Scene_t2348375561_RuntimeObject_RuntimeObject,
};

```

没啥特点，还是一堆的函数指针

```
.rdata:000000018037A440 ?g_Il2CppInvokerPointers@@3QBQ6APEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@ZB dq offset ?RuntimeInvoker_Void_t1185182177@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                                         ; DATA XREF: .rdata:00000001803F61E8↓o
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_Byte_t1134296376_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z ; RuntimeInvoker_Boolean_t97287965_RuntimeObject_UInt16U26_t2814738322(void (*)(void),MethodInfo const *,void *,void * *) ...
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_Boolean_t97287965_RuntimeObject_UInt16U26_t2814738322@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_CameraClearFlags_t2362496923@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_TypeCode_t2987224087_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_Boolean_t97287965_Int32U26_t1369213839_RuntimeObject_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_UInt32_t2560061978_RuntimeObject_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_Void_t1185182177_RuntimeObject_UriFormatExceptionU26_t2370715857@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_RuntimeObject_RuntimeObject_DictionaryNodeU26_t3740769087@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_RuntimeObject_RuntimeObject_RuntimeObject_SByte_t1669577662@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_Boolean_t97287965_RuntimeObject_RuntimeObject_SByte_t1669577662@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_Byte_t1134296376_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_UInt16_t2177724958_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq 2 dup(offset ?RuntimeInvoker_KeyValuePair_2_t71524366_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z)
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_Double_t594665363_DecimalU26_t3714369516@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_UInt16_t2177724958_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
.rdata:000000018037A440                 dq offset ?RuntimeInvoker_IntPtr_t_RuntimeObject@@YAPEAXP6AXXZPEBUMethodInfo@@PEAXPEAPEAX@Z
```

### g_AttributeGenerators

```cpp
extern const CustomAttributesCacheGenerator g_AttributeGenerators[2226] = 
{
	NULL,
	g_mscorlib_Assembly_CustomAttributesCacheGenerator,
	RuntimeObject_CustomAttributesCacheGenerator,
	RuntimeObject_CustomAttributesCacheGenerator_Object__ctor_m297566312,
	RuntimeObject_CustomAttributesCacheGenerator_Object_Finalize_m3076187857,
	RuntimeObject_CustomAttributesCacheGenerator_Object_ReferenceEquals_m610702577,
	ValueType_t3640485471_CustomAttributesCacheGenerator,
	Attribute_t861562559_CustomAttributesCacheGenerator,
	_Attribute_t122494719_CustomAttributesCacheGenerator,
	Int32_t2950945753_CustomAttributesCacheGenerator,
	IFormattable_t1450744796_CustomAttributesCacheGenerator,
	IConvertible_t2977365677_CustomAttributesCacheGenerator,
	IComparable_t36111218_CustomAttributesCacheGenerator,
	SerializableAttribute_t1992588303_CustomAttributesCacheGenerator,
    //......
	FieldWithTarget_t3058750293_CustomAttributesCacheGenerator_m_FieldPath,
	FieldWithTarget_t3058750293_CustomAttributesCacheGenerator_m_TypeString,
	FieldWithTarget_t3058750293_CustomAttributesCacheGenerator_m_DoStatic,
	FieldWithTarget_t3058750293_CustomAttributesCacheGenerator_m_StaticString,
	TriggerListContainer_t2032715483_CustomAttributesCacheGenerator_m_Rules,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_IsTriggerExpanded,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_Type,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_LifecycleEvent,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_ApplyRules,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_Rules,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_TriggerBool,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_InitTime,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_RepeatTime,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_Repetitions,
	EventTrigger_t2527451695_CustomAttributesCacheGenerator_m_Method,
	TrackableTrigger_t621205209_CustomAttributesCacheGenerator_m_Target,
	TrackableTrigger_t621205209_CustomAttributesCacheGenerator_m_MethodPath,
	TriggerRule_t1946298321_CustomAttributesCacheGenerator_m_Target,
	TriggerRule_t1946298321_CustomAttributesCacheGenerator_m_Operator,
	TriggerRule_t1946298321_CustomAttributesCacheGenerator_m_Value,
	TriggerRule_t1946298321_CustomAttributesCacheGenerator_m_Value2,
};

```

```
.rdata:0000000180336DA0 ?g_AttributeGenerators@@3QBQ6AXPEAUCustomAttributesCache@@@ZB dq 0, offset g_mscorlib_Assembly_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                                         ; DATA XREF: .rdata:00000001803F61F8↓o
.rdata:0000000180336DA0                 dq offset RuntimeObject_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                 dq offset Thread_t2300836069_CustomAttributesCacheGenerator_Thread_get_ExecutionContext_m1861734668
.rdata:0000000180336DA0                 dq 2 dup(offset UnhandledExceptionEventArgs_t2886101344_CustomAttributesCacheGenerator_UnhandledExceptionEventArgs_get_IsTerminating_m4073714616)
.rdata:0000000180336DA0                 dq offset WindowsIdentity_t2948242406_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                 dq offset Attribute_t861562559_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                 dq offset _Attribute_t122494719_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                 dq 2 dup(offset WindowsIdentity_t2948242406_CustomAttributesCacheGenerator)
.rdata:0000000180336DA0                 dq offset UIntPtr_t_CustomAttributesCacheGenerator, offset WindowsIdentity_t2948242406_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                 dq offset SerializableAttribute_t1992588303_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                 dq offset SynchronizationAttribute_t3946661254_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                 dq offset ComVisibleAttribute_t1362837655_CustomAttributesCacheGenerator
.rdata:0000000180336DA0                 dq offset WindowsIdentity_t2948242406_CustomAttributesCacheGenerator
......
```

### g_UnresolvedVirtualMethodPointers

```cpp
extern const Il2CppMethodPointer g_UnresolvedVirtualMethodPointers[202] = 
{
	(const Il2CppMethodPointer) UnresolvedVirtualCall_0,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_1,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_2,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_3,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_4,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_5,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_6,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_7,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_8,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_9,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_10,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_11,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_12,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_13,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_14,
    //......
    (const Il2CppMethodPointer) UnresolvedVirtualCall_195,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_196,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_197,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_198,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_199,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_200,
	(const Il2CppMethodPointer) UnresolvedVirtualCall_201,
};

```

```
.rdata:00000001803BD770 ?g_UnresolvedVirtualMethodPointers@@3QBQ6AXXZB dq offset UnresolvedVirtualCall_95, 2 dup(offset UnresolvedVirtualCall_93)
.rdata:00000001803BD770                                         ; DATA XREF: .rdata:00000001803F6208↓o
.rdata:00000001803BD770                 dq 4 dup(offset UnresolvedVirtualCall_95), 3 dup(offset UnresolvedVirtualCall_9)
.rdata:00000001803BD770                 dq 2 dup(offset UnresolvedVirtualCall_93), offset UnresolvedVirtualCall_72
.rdata:00000001803BD770                 dq offset UnresolvedVirtualCall_97, offset UnresolvedVirtualCall_9
.rdata:00000001803BD770                 dq 2 dup(offset UnresolvedVirtualCall_95), offset UnresolvedVirtualCall_9
.rdata:00000001803BD770                 dq 2 dup(offset UnresolvedVirtualCall_95), 2 dup(offset UnresolvedVirtualCall_9)
.rdata:00000001803BD770                 dq offset UnresolvedVirtualCall_99, offset UnresolvedVirtualCall_72
.rdata:00000001803BD770                 dq offset UnresolvedVirtualCall_95, 3 dup(offset UnresolvedVirtualCall_9)
.rdata:00000001803BD770                 dq offset UnresolvedVirtualCall_97, offset UnresolvedVirtualCall_9
.rdata:00000001803BD770                 dq offset UnresolvedVirtualCall_99, offset UnresolvedVirtualCall_9
......
```

需要特别注意的是，这个表的函数有着明显的特征，那就是所有函数行为都相同

```cpp
static  void UnresolvedVirtualCall_199 (RuntimeObject * __this, Scene_t2348375561  ___Scene1, int32_t ___Int322, const RuntimeMethod* method)
{
	il2cpp_codegen_raise_execution_engine_exception(method);
	il2cpp_codegen_no_return();
}

static  void UnresolvedVirtualCall_200 (RuntimeObject * __this, Scene_t2348375561  ___Scene1, Scene_t2348375561  ___Scene2, const RuntimeMethod* method)
{
	il2cpp_codegen_raise_execution_engine_exception(method);
	il2cpp_codegen_no_return();
}

static  void UnresolvedVirtualCall_201 (RuntimeObject * __this, RuntimeObject * ___Object1, RuntimeObject * ___Object2, RuntimeObject * ___Object3, RuntimeObject * ___Object4, const RuntimeMethod* method)
{
	il2cpp_codegen_raise_execution_engine_exception(method);
	il2cpp_codegen_no_return();
}
```

在IDA中也可以看到这样的特征

```cpp
void __fastcall __noreturn UnresolvedVirtualCall_93(Il2CppObject *__this, const MethodInfo *method)
{
  il2cpp_codegen_raise_execution_engine_exception(method);
}
```

```cpp
void __fastcall __noreturn UnresolvedVirtualCall_95(DateTime_t3738529785 *result, Il2CppObject *__this, const MethodInfo *method)
{
  il2cpp_codegen_raise_execution_engine_exception(method);
}
```

这个可以作为判断这个表的一个特征

### g_Il2CppInteropData

```cpp
extern Il2CppInteropData g_Il2CppInteropData[120] = 
{
	{ NULL, Context_t1744531130_marshal_pinvoke, Context_t1744531130_marshal_pinvoke_back, Context_t1744531130_marshal_pinvoke_cleanup, NULL, NULL, &Context_t1744531130_0_0_0 } /* Mono.Globalization.Unicode.SimpleCollator/Context */,
	{ NULL, Escape_t3294788190_marshal_pinvoke, Escape_t3294788190_marshal_pinvoke_back, Escape_t3294788190_marshal_pinvoke_cleanup, NULL, NULL, &Escape_t3294788190_0_0_0 } /* Mono.Globalization.Unicode.SimpleCollator/Escape */,
	{ NULL, PreviousInfo_t2148130204_marshal_pinvoke, PreviousInfo_t2148130204_marshal_pinvoke_back, PreviousInfo_t2148130204_marshal_pinvoke_cleanup, NULL, NULL, &PreviousInfo_t2148130204_0_0_0 } /* Mono.Globalization.Unicode.SimpleCollator/PreviousInfo */,
	{ DelegatePInvokeWrapper_AppDomainInitializer_t682969308, NULL, NULL, NULL, NULL, NULL, &AppDomainInitializer_t682969308_0_0_0 } /* System.AppDomainInitializer */,
	{ DelegatePInvokeWrapper_Swapper_t2822380397, NULL, NULL, NULL, NULL, NULL, &Swapper_t2822380397_0_0_0 } /* System.Array/Swapper */,
	{ NULL, DictionaryEntry_t3123975638_marshal_pinvoke, DictionaryEntry_t3123975638_marshal_pinvoke_back, DictionaryEntry_t3123975638_marshal_pinvoke_cleanup, NULL, NULL, &DictionaryEntry_t3123975638_0_0_0 } /* System.Collections.DictionaryEntry */,
	{ NULL, Slot_t3975888750_marshal_pinvoke, Slot_t3975888750_marshal_pinvoke_back, Slot_t3975888750_marshal_pinvoke_cleanup, NULL, NULL, &Slot_t3975888750_0_0_0 } /* System.Collections.Hashtable/Slot */,
	{ NULL, Slot_t384495010_marshal_pinvoke, Slot_t384495010_marshal_pinvoke_back, Slot_t384495010_marshal_pinvoke_cleanup, NULL, NULL, &Slot_t384495010_0_0_0 } /* System.Collections.SortedList/Slot */,
	{ NULL, Enum_t4135868527_marshal_pinvoke, Enum_t4135868527_marshal_pinvoke_back, Enum_t4135868527_marshal_pinvoke_cleanup, NULL, NULL, &Enum_t4135868527_0_0_0 } /* System.Enum */,
	{ DelegatePInvokeWrapper_ReadDelegate_t714865915, NULL, NULL, NULL, NULL, NULL, &ReadDelegate_t714865915_0_0_0 } /* System.IO.FileStream/ReadDelegate */,
	{ DelegatePInvokeWrapper_WriteDelegate_t4270993571, NULL, NULL, NULL, NULL, NULL, &WriteDelegate_t4270993571_0_0_0 } /* System.IO.FileStream/WriteDelegate */,
	{ NULL, MonoIOStat_t592533987_marshal_pinvoke, MonoIOStat_t592533987_marshal_pinvoke_back, MonoIOStat_t592533987_marshal_pinvoke_cleanup, NULL, NULL, &MonoIOStat_t592533987_0_0_0 } /* System.IO.MonoIOStat */,
	{ NULL, MonoEnumInfo_t3694469084_marshal_pinvoke, MonoEnumInfo_t3694469084_marshal_pinvoke_back, MonoEnumInfo_t3694469084_marshal_pinvoke_cleanup, NULL, NULL, &MonoEnumInfo_t3694469084_0_0_0 } /* System.MonoEnumInfo */,
	{ NULL, CustomAttributeNamedArgument_t287865710_marshal_pinvoke, CustomAttributeNamedArgument_t287865710_marshal_pinvoke_back, CustomAttributeNamedArgument_t287865710_marshal_pinvoke_cleanup, NULL, NULL, &CustomAttributeNamedArgument_t287865710_0_0_0 } /* System.Reflection.CustomAttributeNamedArgument */,
	{ NULL, CustomAttributeTypedArgument_t2723150157_marshal_pinvoke, CustomAttributeTypedArgument_t2723150157_marshal_pinvoke_back, CustomAttributeTypedArgument_t2723150157_marshal_pinvoke_cleanup, NULL, NULL, &CustomAttributeTypedArgument_t2723150157_0_0_0 } /* System.Reflection.CustomAttributeTypedArgument */,
    //......
    { DelegatePInvokeWrapper_UpdatedEventHandler_t1027848393, NULL, NULL, NULL, NULL, NULL, &UpdatedEventHandler_t1027848393_0_0_0 } /* UnityEngine.RemoteSettings/UpdatedEventHandler */,
	{ NULL, CertificateHandler_t2739891000_marshal_pinvoke, CertificateHandler_t2739891000_marshal_pinvoke_back, CertificateHandler_t2739891000_marshal_pinvoke_cleanup, NULL, NULL, &CertificateHandler_t2739891000_0_0_0 } /* UnityEngine.Networking.CertificateHandler */,
	{ NULL, Subsystem_t89723475_marshal_pinvoke, Subsystem_t89723475_marshal_pinvoke_back, Subsystem_t89723475_marshal_pinvoke_cleanup, NULL, NULL, &Subsystem_t89723475_0_0_0 } /* UnityEngine.Experimental.Subsystem */,
	{ NULL, SubsystemDescriptorBase_t2374447182_marshal_pinvoke, SubsystemDescriptorBase_t2374447182_marshal_pinvoke_back, SubsystemDescriptorBase_t2374447182_marshal_pinvoke_cleanup, NULL, NULL, &SubsystemDescriptorBase_t2374447182_0_0_0 } /* UnityEngine.Experimental.SubsystemDescriptorBase */,
	{ NULL, FrameReceivedEventArgs_t2588080103_marshal_pinvoke, FrameReceivedEventArgs_t2588080103_marshal_pinvoke_back, FrameReceivedEventArgs_t2588080103_marshal_pinvoke_cleanup, NULL, NULL, &FrameReceivedEventArgs_t2588080103_0_0_0 } /* UnityEngine.Experimental.XR.FrameReceivedEventArgs */,
	{ NULL, PlaneAddedEventArgs_t2550175725_marshal_pinvoke, PlaneAddedEventArgs_t2550175725_marshal_pinvoke_back, PlaneAddedEventArgs_t2550175725_marshal_pinvoke_cleanup, NULL, NULL, &PlaneAddedEventArgs_t2550175725_0_0_0 } /* UnityEngine.Experimental.XR.PlaneAddedEventArgs */,
	{ NULL, PlaneRemovedEventArgs_t1567129782_marshal_pinvoke, PlaneRemovedEventArgs_t1567129782_marshal_pinvoke_back, PlaneRemovedEventArgs_t1567129782_marshal_pinvoke_cleanup, NULL, NULL, &PlaneRemovedEventArgs_t1567129782_0_0_0 } /* UnityEngine.Experimental.XR.PlaneRemovedEventArgs */,
	{ NULL, PlaneUpdatedEventArgs_t349485851_marshal_pinvoke, PlaneUpdatedEventArgs_t349485851_marshal_pinvoke_back, PlaneUpdatedEventArgs_t349485851_marshal_pinvoke_cleanup, NULL, NULL, &PlaneUpdatedEventArgs_t349485851_0_0_0 } /* UnityEngine.Experimental.XR.PlaneUpdatedEventArgs */,
	{ NULL, PointCloudUpdatedEventArgs_t3436657348_marshal_pinvoke, PointCloudUpdatedEventArgs_t3436657348_marshal_pinvoke_back, PointCloudUpdatedEventArgs_t3436657348_marshal_pinvoke_cleanup, NULL, NULL, &PointCloudUpdatedEventArgs_t3436657348_0_0_0 } /* UnityEngine.Experimental.XR.PointCloudUpdatedEventArgs */,
	{ NULL, SessionTrackingStateChangedEventArgs_t2343035655_marshal_pinvoke, SessionTrackingStateChangedEventArgs_t2343035655_marshal_pinvoke_back, SessionTrackingStateChangedEventArgs_t2343035655_marshal_pinvoke_cleanup, NULL, NULL, &SessionTrackingStateChangedEventArgs_t2343035655_0_0_0 } /* UnityEngine.Experimental.XR.SessionTrackingStateChangedEventArgs */,
	{ DelegatePInvokeWrapper_OnTrigger_t4184125570, NULL, NULL, NULL, NULL, NULL, &OnTrigger_t4184125570_0_0_0 } /* UnityEngine.Analytics.EventTrigger/OnTrigger */,
	NULL,
};
```

这个表的特征明显，那就是他的表项里并不是单纯存放的函数指针，而是`Il2CppInteropData`结构体

```cpp
typedef struct Il2CppInteropData
{
    Il2CppMethodPointer delegatePInvokeWrapperFunction;
    PInvokeMarshalToNativeFunc pinvokeMarshalToNativeFunction;
    PInvokeMarshalFromNativeFunc pinvokeMarshalFromNativeFunction;
    PInvokeMarshalCleanupFunc pinvokeMarshalCleanupFunction;
    CreateCCWFunc createCCWFunction;
    const Il2CppGuid* guid;
#if RUNTIME_MONO
    MonoMetadataToken typeToken;
    uint64_t hash;
#else
    const Il2CppType* type;
#endif
} Il2CppInteropData;
```

基本上是一系列函数指针

加载了符号的IDA会这样显示

```
.data:0000000180455D00 ?g_Il2CppInteropData@@3PAUIl2CppInteropData@@A Il2CppInteropData <0, offset Context_t1744531130_marshal_pinvoke, \
.data:0000000180455D00                                         ; DATA XREF: .rdata:00000001803F6218↑o
.data:0000000180455D00                                    offset Context_t1744531130_marshal_pinvoke_back, \ ; Il2CppType const ReadDelegate_t714865915_0_0_0 ...
.data:0000000180455D00                                    offset SecurityParser_OnProcessingInstruction_m2327827622,\
.data:0000000180455D00                                    0, 0, \
.data:0000000180455D00                                    offset ?Context_t1744531130_0_0_0@@3UIl2CppType@@B>
.data:0000000180455D00                 Il2CppInteropData <0, offset Escape_t3294788190_marshal_pinvoke, \
.data:0000000180455D00                                    offset Escape_t3294788190_marshal_pinvoke_back, \
.data:0000000180455D00                                    offset Escape_t3294788190_marshal_pinvoke_cleanup, \
.data:0000000180455D00                                    0, 0, \
.data:0000000180455D00                                    offset ?Escape_t3294788190_0_0_0@@3UIl2CppType@@B>
.data:0000000180455D00                 Il2CppInteropData <0, \
.data:0000000180455D00                                    offset PreviousInfo_t2148130204_marshal_pinvoke_back,\
.data:0000000180455D00                                    offset PreviousInfo_t2148130204_marshal_pinvoke_back,\
.data:0000000180455D00                                    offset SecurityParser_OnProcessingInstruction_m2327827622,\
.data:0000000180455D00                                    0, 0, \
.data:0000000180455D00                                    offset ?PreviousInfo_t2148130204_0_0_0@@3UIl2CppType@@B>
.data:0000000180455D00                 Il2CppInteropData <offset DelegatePInvokeWrapper_AppDomainInitializer_t682969308,\
.data:0000000180455D00                                    0, 0, 0, 0, 0, \
.data:0000000180455D00                                    offset ?AppDomainInitializer_t682969308_0_0_0@@3UIl2CppType@@B>
.data:0000000180455D00                 Il2CppInteropData <offset DelegatePInvokeWrapper_Swapper_t2822380397, \
.data:0000000180455D00                                    0, 0, 0, 0, 0, \
.data:0000000180455D00                                    offset ?Swapper_t2822380397_0_0_0@@3UIl2CppType@@B>
.data:0000000180455D00                 Il2CppInteropData <0, \
.data:0000000180455D00                                    offset DictionaryEntry_t3123975638_marshal_pinvoke,\
.data:0000000180455D00                                    offset DictionaryEntry_t3123975638_marshal_pinvoke_back,\
.data:0000000180455D00                                    offset DictionaryEntry_t3123975638_marshal_pinvoke_cleanup,\
.data:0000000180455D00                                    0, 0, \
.data:0000000180455D00                                    offset ?DictionaryEntry_t3123975638_0_0_0@@3UIl2CppType@@B>
```

如果没加符号暂时不清楚会怎么样，之后可能会实验到

### 小结

对`g_CodeRegistration`的研究差不多就告一段落了，回顾这个全局变量，可以发现他与il2cpp里面的函数密切相关，因此被叫做“Code”Registration嘛。里面包含了所有的il2cpp函数的指针信息，包括普通函数，类构造函数，虚函数等等，这几个表都十分的重要！



## 细究 g_MetadataRegistration

类型为`Il2CppMetadataRegistration`

```cpp
typedef struct Il2CppMetadataRegistration
{
    int32_t genericClassesCount;
    Il2CppGenericClass* const * genericClasses;
    int32_t genericInstsCount;
    const Il2CppGenericInst* const * genericInsts;
    int32_t genericMethodTableCount;
    const Il2CppGenericMethodFunctionsDefinitions* genericMethodTable;
    int32_t typesCount;
    const Il2CppType* const * types;
    int32_t methodSpecsCount;
    const Il2CppMethodSpec* methodSpecs;

    FieldIndex fieldOffsetsCount;
    const int32_t** fieldOffsets;

    TypeDefinitionIndex typeDefinitionsSizesCount;
    const Il2CppTypeDefinitionSizes** typeDefinitionsSizes;
    const size_t metadataUsagesCount;
    void** const* metadataUsages;
} Il2CppMetadataRegistration;
```

`g_MetadataRegistration `由以下几个表构成

```cpp
extern const Il2CppMetadataRegistration g_MetadataRegistration = 
{
	1496,
	s_Il2CppGenericTypes,
	411,
	g_Il2CppGenericInstTable,
	2511,
	s_Il2CppGenericMethodFunctions,
	6788,
	g_Il2CppTypeTable,
	2730,
	g_Il2CppMethodSpecTable,
	1788,
	g_FieldOffsetTable,
	1788,
	g_Il2CppTypeDefinitionSizesTable,
	5293,
	g_MetadataUsages,
};
```

根据变量名猜测含义如下：

`s_Il2CppGenericTypes` : 泛型的类型表

`g_Il2CppGenericInstTable` : 泛型的调用参数表

`s_Il2CppGenericMethodFunctions` :  泛型函数表

`g_Il2CppTypeTable` : 类型表

`g_Il2CppMethodSpecTable` : 特殊方法表？

`g_FieldOffsetTable` : 属性偏移表 (应该是非常重要的一个表)

`g_Il2CppTypeDefinitionSizesTable` : 类型定义的大小表

`g_MetadataUsages` : metadata用法表？？？

下面分别查看每个表的内容

### s_Il2CppGenericTypes

```cpp
extern Il2CppGenericClass* const s_Il2CppGenericTypes[1496] = 
{
	&IEnumerator_1_t3512676632_GenericClass,
	&InternalEnumerator_1_t3987170281_GenericClass,
	&IList_1_t600458651_GenericClass,
	&ICollection_1_t1613291102_GenericClass,
	&IEnumerable_1_t2059959053_GenericClass,
	&IComparable_1_t2319610166_GenericClass,
	&IEquatable_1_t3842091480_GenericClass,
	&IEnumerator_1_t4067030938_GenericClass,
    //...
	&InvokableCall_1_t3918776686_GenericClass,
	&UnityAction_1_t1306688849_GenericClass,
	&UnityEvent_1_t1603512212_GenericClass,
	&InvokableCall_1_t839016946_GenericClass,
	&InvokableCall_2_t39947495_GenericClass,
	&InvokableCall_3_t3505746873_GenericClass,
	&InvokableCall_4_t3420349028_GenericClass,
	&Dictionary_2_t2206118517_GenericClass,
	&IComparer_1_t884274696_GenericClass,
	&IComparer_1_t1188505307_GenericClass,
	&IEqualityComparer_1_t2502945182_GenericClass,
	&IEqualityComparer_1_t730487731_GenericClass,
};
```

这样一个由指针构成的表，比较好奇`Il2CppGenericClass`里面有什么

```cpp
typedef struct Il2CppGenericClass
{
    TypeDefinitionIndex typeDefinitionIndex;    /* the generic type definition */
    Il2CppGenericContext context;   /* a context that contains the type instantiation doesn't contain any method instantiation */
    Il2CppClass *cached_class;  /* if present, the Il2CppClass corresponding to the instantiation.  */
} Il2CppGenericClass;
```

第二个成员如下

```cpp
typedef struct Il2CppGenericContext
{
    /* The instantiation corresponding to the class generic parameters */
    const Il2CppGenericInst *class_inst;
    /* The instantiation corresponding to the method generic parameters */
    const Il2CppGenericInst *method_inst;
} Il2CppGenericContext;
```

主要存储的是参数，参数定义如下

```cpp
typedef struct Il2CppGenericInst
{
    uint32_t type_argc;
    const Il2CppType **type_argv;
} Il2CppGenericInst;
```

存储参数个数和参数的指针

回到`Il2CppGenericClass`上来

`typeDefinitionIndex`应该就对应了表`g_MetadataRegistration.s_Il2CppGenericTypes`中的一个下标

`context` 这里存的东西不是太懂，不知道是干嘛的

`cached_class` 也不太懂是什么，但是发现表中大多数项目的这一个成员都是NULL

```cpp
Il2CppGenericClass I...s = { 41, { &GenInst_RuntimeObject_0_0_0, NULL }, NULL };
Il2CppGenericClass I...s = { 9, { &GenInst_Int32_t2950945753_0_0_0, NULL }, NULL };
Il2CppGenericClass Inte...1_Ge...ass = { 41, { &GenInst_Char_t3634460470_0_0_0, NULL }, NULL };
```

### g_Il2CppGenericInstTable

```cpp
extern const Il2CppGenericInst* const g_Il2CppGenericInstTable[411] = 
{
	&GenInst_RuntimeObject_0_0_0,
	&GenInst_Int32_t2950945753_0_0_0,
	&GenInst_Char_t3634460470_0_0_0,
	&GenInst_Int64_t3736567304_0_0_0,
	&GenInst_UInt32_t2560061978_0_0_0,
	&GenInst_UInt64_t4134040092_0_0_0,
	&GenInst_Byte_t1134296376_0_0_0,
    //...
	&GenInst_InvokableCall_4_t3865002609_gp_1_0_0_0,
	&GenInst_InvokableCall_4_t3865002609_gp_2_0_0_0,
	&GenInst_InvokableCall_4_t3865002609_gp_3_0_0_0,
	&GenInst_CachedInvokableCall_1_t3153979999_gp_0_0_0_0,
	&GenInst_UnityEvent_1_t74220259_gp_0_0_0_0,
	&GenInst_UnityEvent_2_t477504786_gp_0_0_0_0_UnityEvent_2_t477504786_gp_1_0_0_0,
	&GenInst_UnityEvent_3_t3206388141_gp_0_0_0_0_UnityEvent_3_t3206388141_gp_1_0_0_0_UnityEvent_3_t3206388141_gp_2_0_0_0,
	&GenInst_UnityEvent_4_t3609672668_gp_0_0_0_0_UnityEvent_4_t3609672668_gp_1_0_0_0_UnityEvent_4_t3609672668_gp_2_0_0_0_UnityEvent_4_t3609672668_gp_3_0_0_0,
	&GenInst_LockDictionary_2_t3023059588_gp_0_0_0_0_LockDictionary_2_t3023059588_gp_1_0_0_0,
};
```

是一个这样的指针构成的表，构造和刚才的那个表类似，刚才已经说明过了，`Il2CppGenericInst`这个结构体实际上就包含两个部分，表示一个函数的参数

```cpp
typedef struct Il2CppGenericInst
{
    uint32_t type_argc; //参数个数
    const Il2CppType **type_argv; //参数内容
} Il2CppGenericInst;
```

那他具体是怎么去生成这个表的呢，我们随机选取一项表项来看

```cpp
extern const Il2CppGenericInst GenInst_LockDictionary_2_t3023059588_gp_0_0_0_0_LockDictionary_2_t3023059588_gp_1_0_0_0 = { 2, GenInst_LockDictionary_2_t3023059588_gp_0_0_0_0_LockDictionary_2_t3023059588_gp_1_0_0_0_Types };
```

比如这一项，我们看到2表示包含两个参数，而这两个参数分别内容是什么呢，继续看

```cpp
static const RuntimeType* GenInst_LockDictionary_2_t3023059588_gp_0_0_0_0_LockDictionary_2_t3023059588_gp_1_0_0_0_Types[] = { (&LockDictionary_2_t3023059588_gp_0_0_0_0), (&LockDictionary_2_t3023059588_gp_1_0_0_0) };
```

注意，`RuntimeType`实际上和`Il2CppType`是同一个东西

```cpp
// codegen\il2cpp-codegen.h
typedef Il2CppType RuntimeType;
```

这里刚好有两个类型被放在这样一个数组里，这两个类型的类型是Il2cpp基本类型

```cpp
extern const Il2CppType LockDictionary_2_t3023059588_gp_0_0_0_0;
extern const Il2CppType LockDictionary_2_t3023059588_gp_1_0_0_0;
```

你可以看到，这两个类型的定义是

```cpp
extern const RuntimeType LockDictionary_2_t3023059588_gp_0_0_0_0 = { (void*)175, 0, IL2CPP_TYPE_VAR, 0, 0, 0 };
extern const RuntimeType LockDictionary_2_t3023059588_gp_1_0_0_0 = { (void*)176, 0, IL2CPP_TYPE_VAR, 0, 0, 0 };
```

实际上就是填充了`Il2CppType`的union部分

```cpp
typedef struct Il2CppType
{
    union
    {
        // We have this dummy field first because pre C99 compilers (MSVC) can only initializer the first value in a union.
        void* dummy;
        TypeDefinitionIndex klassIndex; /* for VALUETYPE and CLASS */
        const Il2CppType *type;   /* for PTR and SZARRAY */
        Il2CppArrayType *array; /* for ARRAY */
        //MonoMethodSignature *method;
        GenericParameterIndex genericParameterIndex; /* for VAR and MVAR */
        Il2CppGenericClass *generic_class; /* for GENERICINST */
    } data;
    unsigned int attrs    : 16; /* param attributes or field flags */
    Il2CppTypeEnum type     : 8;
    unsigned int num_mods : 6;  /* max 64 modifiers follow at the end */
    unsigned int byref    : 1;
    unsigned int pinned   : 1;  /* valid when included in a local var signature */
    //MonoCustomMod modifiers [MONO_ZERO_LEN_ARRAY]; /* this may grow */
} Il2CppType;
```

总之，这个表存了一些函数的参数的构成，包括参数个数和参数类型

### s_Il2CppGenericMethodFunctions

```cpp
extern const Il2CppGenericMethodFunctionsDefinitions s_Il2CppGenericMethodFunctions[2511] = 
{
	{ 0, 0/*NULL*/, 5/*5*/},
	{ 1, 0/*NULL*/, 1/*1*/},
	{ 2, 0/*NULL*/, 4/*4*/},
	{ 3, 0/*NULL*/, 4/*4*/},
	{ 4, 1/*(Il2CppMethodPointer)&Array_InternalArray__IEnumerable_GetEnumerator_TisRuntimeObject_m3132609973_gshared*/, 4/*4*/},
	{ 5, 2/*(Il2CppMethodPointer)&Array_InternalArray__ICollection_Add_TisRuntimeObject_m4216329873_gshared*/, 89/*89*/},
	{ 6, 3/*(Il2CppMethodPointer)&Array_InternalArray__ICollection_Remove_TisRuntimeObject_m2110193223_gshared*/, 1/*1*/},
	{ 7, 4/*(Il2CppMethodPointer)&Array_InternalArray__ICollection_Contains_TisRuntimeObject_m4067783231_gshared*/, 1/*1*/},
	{ 8, 5/*(Il2CppMethodPointer)&Array_InternalArray__ICollection_CopyTo_TisRuntimeObject_m4245759982_gshared*/, 85/*85*/},
	{ 9, 6/*(Il2CppMethodPointer)&Array_InternalArray__Insert_TisRuntimeObject_m1619219378_gshared*/, 188/*188*/},
	{ 10, 7/*(Il2CppMethodPointer)&Array_InternalArray__IndexOf_TisRuntimeObject_m2971736253_gshared*/, 5/*5*/},
	{ 11, 8/*(Il2CppMethodPointer)&Array_InternalArray__get_Item_TisRuntimeObject_m3347010206_gshared*/, 97/*97*/},
	{ 12, 9/*(Il2CppMethodPointer)&Array_InternalArray__set_Item_TisRuntimeObject_m2895257685_gshared*/, 188/*188*/},
    //...
	{ 2500, 2342/*(Il2CppMethodPointer)&UnityAction_2_BeginInvoke_m1769266175_gshared*/, 1266/*1266*/},
	{ 2501, 2343/*(Il2CppMethodPointer)&UnityAction_2_EndInvoke_m2179051926_gshared*/, 89/*89*/},
	{ 2502, 2344/*(Il2CppMethodPointer)&UnityAction_2__ctor_m2941677221_gshared*/, 212/*212*/},
	{ 2503, 2345/*(Il2CppMethodPointer)&UnityAction_2_BeginInvoke_m1733258791_gshared*/, 1267/*1267*/},
	{ 2504, 2346/*(Il2CppMethodPointer)&UnityAction_2_EndInvoke_m2385586247_gshared*/, 89/*89*/},
	{ 2505, 2347/*(Il2CppMethodPointer)&UnityEvent_1_RemoveListener_m1953458448_gshared*/, 89/*89*/},
	{ 2506, 2348/*(Il2CppMethodPointer)&UnityEvent_1_FindMethod_Impl_m1397247356_gshared*/, 9/*9*/},
	{ 2507, 2349/*(Il2CppMethodPointer)&UnityEvent_1_GetDelegate_m617150804_gshared*/, 9/*9*/},
	{ 2508, 2350/*(Il2CppMethodPointer)&UnityEvent_1_GetDelegate_m2283422164_gshared*/, 40/*40*/},
	{ 2728, 413/*(Il2CppMethodPointer)&UnityEvent_1_FindMethod_Impl_m322741469_gshared*/, 9/*9*/},
	{ 2729, 414/*(Il2CppMethodPointer)&UnityEvent_1_GetDelegate_m1223269239_gshared*/, 9/*9*/},
};
```

这个表有点意思了，存的全是一些数字,然后看他的结构体定义：

```cpp
typedef struct
{
    MethodIndex methodIndex;
    MethodIndex invokerIndex;
} Il2CppGenericMethodIndices;

typedef struct Il2CppGenericMethodFunctionsDefinitions
{
    GenericMethodIndex genericMethodIndex;
    Il2CppGenericMethodIndices indices;
} Il2CppGenericMethodFunctionsDefinitions;
```

可以知道第一个数字代表泛型函数的下标，第二个数字表示methodIndex，第三个数字表示invokerIndex

### g_Il2CppTypeTable

```cpp
extern const RuntimeType* const  g_Il2CppTypeTable[6788] = 
{
	&IEnumerator_1_t3512676632_0_0_0,
	&RuntimeObject_0_0_0,
	&InternalEnumerator_1_t3987170281_0_0_0,
	&IList_1_t600458651_0_0_0,
	&ICollection_1_t1613291102_0_0_0,
	&IEnumerable_1_t2059959053_0_0_0,
	&IComparable_1_t2319610166_0_0_0,
	&Int32_t2950945753_0_0_0,
	&IEquatable_1_t3842091480_0_0_0,
	&IEnumerator_1_t4067030938_0_0_0,
	&Char_t3634460470_0_0_0,
	&InternalEnumerator_1_t246557291_0_0_0,
	&IList_1_t1154812957_0_0_0,
	&ICollection_1_t2167645408_0_0_0,
	&IEnumerable_1_t2614313359_0_0_0,
    //...
	&GUIStyleU5BU5D_t2383250302_0_0_0,
	&ILTokenInfoU5BU5D_t973106575_0_0_0,
	&MarkU5BU5D_t3645422402_0_0_0,
	&TailoringInfoU5BU5D_t1797664499_0_0_0,
	&DateTimeU5BU5D_t1184652292_0_0_0,
	&DecimalU5BU5D_t1145110141_0_0_0,
	&TimeSpanU5BU5D_t4291357516_0_0_0,
	&TypeTagU5BU5D_t1563918664_0_0_0,
	&MemberInfoU5BU5D_t1302094432_0_0_0,
	&PlayableBindingU5BU5D_t829358056_0_0_0,
	&ResourceInfoU5BU5D_t2132996019_0_0_0,
	&HitInfoU5BU5D_t1685002053_0_0_0,
	&SlotU5BU5D_t227397015_0_0_0,
	&ITrackingHandlerU5BU5D_t3758023570_0_0_0,
	&ConstructorBuilderU5BU5D_t3223009221_0_0_0,
	&UriSchemeU5BU5D_t2082808316_0_0_0,
};
```

看样子就是把之前分析过的（g_Il2CppGenericInstTable），所有`RuntimeType`那些类型放在里面去了，对应了一个下标

```
000000180359FD0 ?g_Il2CppTypeTable@@3QBQEBUIl2CppType@@B dq offset ?IEnumerator_1_t3512676632_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                                         ; DATA XREF: .rdata:Il2CppMetadataRegistration const g_MetadataRegistration↓o
.rdata:0000000180359FD0                 dq offset ?RuntimeObject_0_0_0@@3UIl2CppType@@B, offset ?InternalEnumerator_1_t3987170281_0_0_0@@3UIl2CppType@@B ; Il2CppType const IEnumerator_1_t1351688939_0_0_2 ...
.rdata:0000000180359FD0                 dq offset ?IList_1_t600458651_0_0_0@@3UIl2CppType@@B, offset ?ICollection_1_t1613291102_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?IEnumerable_1_t2059959053_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?IComparable_1_t2319610166_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?Int32_t2950945753_0_0_0@@3UIl2CppType@@B, offset ?IEquatable_1_t3842091480_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?IEnumerator_1_t4067030938_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?Char_t3634460470_0_0_0@@3UIl2CppType@@B, offset ?InternalEnumerator_1_t246557291_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?IList_1_t1154812957_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?ICollection_1_t2167645408_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?IEnumerable_1_t2614313359_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?IComparable_1_t2448770577_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?IEquatable_1_t3971251891_0_0_0@@3UIl2CppType@@B
.rdata:0000000180359FD0                 dq offset ?IComparable_1_t3105231717_0_0_0@@3UIl2CppType@@B
......
```

### g_Il2CppMethodSpecTable

```cpp
extern const Il2CppMethodSpec g_Il2CppMethodSpecTable[2730] = 
{
	{ 85, 0, -1 },
	{ 93, 0, -1 },
	{ 302, 0, -1 },
	{ 483, 0, -1 },
	{ 764, -1, 0 },
	{ 766, -1, 0 },
	{ 767, -1, 0 },
	{ 768, -1, 0 },
    //...
	{ 1667, 342, -1 },
	{ 1763, 363, -1 },
	{ 1762, 363, -1 },
	{ 1763, 364, -1 },
	{ 1762, 364, -1 },
	{ 8439, 405, -1 },
	{ 8496, 189, -1 },
	{ 8497, 189, -1 },
};
```

也是一个很神奇的列表，看看`Il2CppMethodSpec`的结构

```cpp
typedef struct Il2CppMethodSpec
{
    MethodIndex methodDefinitionIndex;
    GenericInstIndex classIndexIndex;
    GenericInstIndex methodIndexIndex;
} Il2CppMethodSpec;
```

具体功能是什么，还需要看代码分析，现在暂时不分析。

### g_FieldOffsetTable

```cpp
extern const int32_t* g_FieldOffsetTable[1788] = 
{
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	g_FieldOffsetTable5,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	g_FieldOffsetTable11,
	g_FieldOffsetTable12,
	NULL,
	g_FieldOffsetTable14,
	g_FieldOffsetTable15,
	g_FieldOffsetTable16,
	g_FieldOffsetTable17,
	g_FieldOffsetTable18,
	g_FieldOffsetTable19,
	g_FieldOffsetTable20,
	g_FieldOffsetTable21,
	NULL,
    //...
	g_FieldOffsetTable1769,
	g_FieldOffsetTable1770,
	g_FieldOffsetTable1771,
	g_FieldOffsetTable1772,
	g_FieldOffsetTable1773,
	g_FieldOffsetTable1774,
	g_FieldOffsetTable1775,
	g_FieldOffsetTable1776,
	g_FieldOffsetTable1777,
	g_FieldOffsetTable1778,
	g_FieldOffsetTable1779,
	g_FieldOffsetTable1780,
	g_FieldOffsetTable1781,
	g_FieldOffsetTable1782,
	g_FieldOffsetTable1783,
	NULL,
	g_FieldOffsetTable1785,
	NULL,
	g_FieldOffsetTable1787,
};
```

可以很明显看出，这个表存的就是一堆属性的偏移量，具体里面的每个偏移的定义是分开的

比如：`g_FieldOffsetTable1769`的定义是

```cpp
extern const int32_t g_FieldOffsetTable1769[6];
```

赋值在:

```cpp
extern const int32_t g_FieldOffsetTable1769[6] = 
{
	StandardEventPayload_t1629891255::get_offset_of_m_IsEventExpanded_0(),
	StandardEventPayload_t1629891255::get_offset_of_m_StandardEventType_1(),
	StandardEventPayload_t1629891255::get_offset_of_standardEventType_2(),
	StandardEventPayload_t1629891255::get_offset_of_m_Parameters_3(),
	StandardEventPayload_t1629891255_StaticFields::get_offset_of_m_EventData_4(),
	StandardEventPayload_t1629891255::get_offset_of_m_Name_5(),
};
```

随便点开一个获取偏移的函数，发现是这样的

```cpp
inline static int32_t get_offset_of_m_IsEventExpanded_0() { return static_cast<int32_t>(offsetof(StandardEventPayload_t1629891255, ___m_IsEventExpanded_0)); }
```

```cpp
#define offsetof(TYPE, MEMBER) __builtin_offsetof (TYPE, MEMBER)
```

总之这个表就是用来存偏移信息的