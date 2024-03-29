---
title: "Creating Anvil-like annotation for Hilt using KSP"
date: 2024-02-03T16:57:15+02:00
draft: false
---

_This post assumes some familiarity with dependency injection using Dagger Hilt on Android._

This blog post will address code generation to ease the process using `@Binds` annotation to create aliases to a particular type.

### Provide a type to the graph

When you are creating classes, usually only annotating them with `@Inject` will be enough to tell Dagger which constructor and fields should be used to create instances of classes.

```kotlin
class LocalDataSource @Iject constructor(db: Database) { ... }
```
Unfortunately, its usage is limited. This annotation alone cannot be relied upon when implementing an interface that will subsequently be injected into a constructor.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    fun providesMyDatabase(context: Context) : Database = MyDataBase(context)
}

// OR

@Module
@InstallIn(SingletonComponent::class)
interface DatabaseModule {
    
    @Binds
    fun bindsMyDataBase(db: MyDataBase) : Database
}
```

You can read more about it in the [official documentation](https://dagger.dev/dev-guide/).

With this short snippet, we can see that we need to create a `@Module` and provides a way for dagger to create an alias to the type - `Database` in the above example.

#### Anvil simplifies the process by reducing wiring boilerplate.
[Anvil](https://github.com/square/anvil) is a kotlin compiler plugin that makes dependency injection with Dagger easier by generating a lot of boilerplate in our behalf. You can reach to its official documentation to learn more about it. 

In this post, our focus will be on a specific annotation introduced by Anvil: `@ContributesBinding`. We'll explore how this annotation simplifies the process of binding interfaces to their implementations. This annotation replaces the need of creating a `@Module` by contributing the dependency directly to the `Compoent`.

### Generating a module for Hilt

The objective is to annotate the implementation of an interface, allowing the automatic generation of module and binding boilerplate. This facilitates the integration of the annotated class within the Dagger Hilt framework.

We need to create our annotation. We can put it inside a dedicated [module](https://github.com/brunohensel/Hilt-Annotation/tree/main/annotation). 
```kotlin
@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.CLASS)
@GeneratesRootInput
annotation class ContributesBinding(val component: KClass<*>, val boundType: KClass<*> = Any::class)
```
**Note:** [GeneratesRootInput](https://dagger.dev/api/latest/dagger/hilt/GeneratesRootInput.html) lets Hilt knows that it has to wait before creating the components.

_Bound type_ is useful in case your class implements more than one interface, and you need to tell which one needs to be provided.

Now we move onto creating our processor. For that, we need to add a dependency on [KSP](https://kotlinlang.org/docs/ksp-overview.html) that will generate code for us. In the link you can go to the `Quickstart` to get more familiar with it and go through the setup.

To integrate into the Kotlin Symbol Processing - KSP, we need our class to implement the `SymbolProcessor`. 

```kotlin
class BindingProcessor(private val codeGenerator: CodeGenerator) : SymbolProcessor {
    
    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver.getSymbolsWithAnnotation(Utils.CONTRIBUTES_BINDING.canonicalName)
        // creates a pair of valid and invalid list of symbols
        val (validSymbols, invalidSymbols) = symbols.partition { it.validate() }

        for (symbol in validSymbols) {
            if (symbol is KSClassDeclaration) {
                symbol.accept(
                    visitor = ContributeBindingVisitor(codeGenerator, logger),
                    data = Unit
                )
            }
        }
        return invalidSymbols // symbols that the processor can't process.
    }
} 
```
In the `process` method, we will get all the classes annotated with our annotation, and visit them to get the information about the `Component` where the binding has to be installed in, and the Super type being implemented by the annotated class. With that information we can create a new File and add our generated `@Module` in it.

As KSP processes occur in multiple rounds, our processor returns symbols it couldn't process. This ensures that other processors in the code can handle them during subsequent rounds of processing.
The documentation of the _process_ function explains that:
>Returns: A list of deferred symbols that the processor can't process. Only symbols that can't be processed at this round should be returned.

The `ContributeBindingVisitor` class extends `KSVisitorVoid` class, and we use one of its methods to visit the annotated class declaration. Here we can validate the correctness and expectations of the code.

```kotlin
override fun visitClassDeclaration(classDeclaration: KSClassDeclaration, data: Unit) {
    val annotation = getAnnotation(classDeclaration) ?: return
    val component = resolveType(annotation.arguments.component1().value!!)
    val boundType = annotation.arguments.component2()
    val defaultArgument = annotation.defaultArguments.component1()
    val superTypes = classDeclaration.superTypes
    val superTypesCount = superTypes.count()
    val firstSuperType = superTypes.firstOrNull()?.resolve()
        ?: throw IllegalStateException(
            "A class annotated with @ContributesBinding should implement at least one interface"
        )

    if (superTypesCount > 1 && boundType == defaultArgument) {
        // we could throw an exception here as well
        logger.exception(
            IllegalArgumentException(
                "There are more than one super type declared without any bounded type declaration "
            )
        )
    }

    val boundTypeArg = if (superTypesCount > 1) {
        BoundType(typeArg = resolveType(boundType.value!!), typeName = null)
    } else BoundType(typeName = firstSuperType.toClassName(), typeArg = null)

    val fileSpec: FileSpec = bindingFileSpec(
        subtype = classDeclaration.asType(listOf()),
        componentArg = component,
        boundTypeArg = boundTypeArg,
        logger = logger,
    )

    fileSpec.writeTo(codeGenerator = codeGenerator, aggregating = false)
}
```

Leveraging [Kotlinpoet](https://github.com/square/kotlinpoet), we can effortlessly generate Kotlin code. This is done by the function `bindingFileSpec`.

```kotlin
internal fun bindingFileSpec(
    subtype: KSType,
    componentArg: Any,
    boundTypeArg: BoundType,
): FileSpec {
    val subtypeClassName = subtype.toClassName().topLevelClassName()
    val moduleName = ClassName(
        packageName = subtypeClassName.packageName,
        "${subtypeClassName.simpleNames.joinToString("_")}_HiltBindingModule"
    )

    val (boundType, typeName) = boundTypeArg
    val bindName = boundType?.javaClass?.simpleName ?: when (typeName) {
        is ClassName -> typeName.simpleName
        else -> error("Bound type is not a class")
    }

    // This will create the @Module interface with @InstallIn and @OriginatingElement annotations
    val hiltModuleSpec = TypeSpec.interfaceBuilder(moduleName)
        .addAnnotation(Utils.DAGGER_MODULE)
        .addAnnotation(
            AnnotationSpec.builder(Utils.INSTALL_IN)
                .addMember("%T::class", componentArg)
                .build()
        )
        .addAnnotation(
            AnnotationSpec.builder(Utils.ORIGINATING_ELEMENT)
                .addMember("topLevelClass = %T::class", subtypeClassName)
                .build()
        )
        // this will create the @Binds fun bind(impl: TypeImpl): Type
        .addModifiers(KModifier.PUBLIC)
        .addFunction(
            FunSpec.builder("bind$bindName")
                .addAnnotation(Utils.DAGGER_BINDS)
                .addModifiers(KModifier.PUBLIC, KModifier.ABSTRACT)
                .applyReturn(boundType, typeName)
                .addParameter("impl", subtype.toTypeName())
                .build()
        )
        .build()
    
    return FileSpec.builder(moduleName).addType(hiltModuleSpec).build()
}
```

[OriginatingElement](https://dagger.dev/api/latest/dagger/hilt/codegen/OriginatingElement.html) is needed because we are generating Hilt modules.

Now, we can use the annotation and let KSP generate the necessary Dagger Hilt boilerplate code for our specific use case.  No need to manually creates a `@Module` or have to add a bind function inside of one.

```kotlin
interface Test {
    fun greeting(): String
}

@ContributesBinding(component = ActivityComponent::class)
class AnnotationTest @Inject constructor() : Test {
    override fun greeting(): String {
        return "KSP"
    }
}

//The code generated will be like this

@Module
@InstallIn(ActivityComponent::class)
@OriginatingElement(topLevelClass = AnnotationTest::class)
public interface AnnotationTest_HiltBindingModule {
    @Binds
    public fun bindTest(`impl`: AnnotationTest): Test
}
```

In conclusion, it's important to note that this post serves as a summary of my personal exploration with KSP. The code provided is experimental in nature and does not prioritize production-level considerations, such as performance optimization. It was crafted primarily to explore possibilities and demonstrate concepts.
The working project can be found in this [repository](https://github.com/brunohensel/Hilt-Annotation). 
