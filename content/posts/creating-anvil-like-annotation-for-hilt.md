---
title: "Creating Anvil-like annotation for Hilt"
date: 2024-01-26T17:05:15+02:00
draft: true
---

_This post assumes some familiarity with dependency injection using Dagger Hilt on Android._

This blog post will address code generation to ease the process using `@Binds` annotation to create aliases to a particular type.

### Provide a type to the graph

When you are creating classes, usually only annotating them with `@Inject` will be enough to tell Dagger which constructor and fields should be used to create instances of classes.

```kotlin
class LocalDataSource @Iject constructor(db: Database) { ... }
```

Unfortunately, we can't use it in all places. It's not possible to only rely on this annotation when we are implementing an interface which itself will be injected in some constructor later on. For this case, we could use either `@Provides` or `@Binds` annotation.

```kotlin
@Module
object DatabaseModule {
    
    @Provides
    fun providesMyDatabase(context: Context) : Database = MyDataBase(context)
}

// OR

@Module
interface DatabaseModule {
    
    @Binds
    fun bindsMyDataBase(db: MyDataBase) : Database
}
```

You can read more about it in the [official documentation](https://dagger.dev/dev-guide/).

With this short snippet, we can see that we need to create a `@Module` and add provides a way for dagger to create an alias to the type - `Database` in the above example.

[Anvil](https://github.com/square/anvil) is a kotlin compiler plugin that makes dependency injection with Dagger easier by generating a lot of boilerplate in our behalf. You can reach to its official documentation to learn more about it. 

We are focusing in a particular type of annotation that Anvil introduced, which makes very easy to bind interfaces with its implementation: `@ContributesBinding`. This annotation replaces the need of creating a `@Module`, contributing the dependency directly to the `Compoent`.

### Generating a module for Hilt

The goal is to annotate the implementation of an interface and have all the module, binding boilerplate be created for us.

We need to create our annotation. We can put it inside a dedicated [module](https://github.com/brunohensel/Hilt-Annotation/tree/main/annotation). 
```kotlin
@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.CLASS)
@GeneratesRootInput
annotation class ContributesBinding(val component: KClass<*>, val boundType: KClass<*> = Any::class)
```
[GeneratesRootInput](https://dagger.dev/api/latest/dagger/hilt/GeneratesRootInput.html) lets Hilt knows that it has to wait before creating the components.

Now we move to create our processor. For that, we need to add a dependency on [KSP](https://kotlinlang.org/docs/ksp-overview.html) to generate code for us. In the link you can go to the `Quickstart` to get more familiar with it.

To integrate into the Kotlin Symbol Processing - KSP, we need our class to implement the `SymbolProcessor`. 

```kotlin
class BindingProcessor(private val codeGenerator: CodeGenerator) : SymbolProcessor {
    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver.getSymbolsWithAnnotation(Utils.CONTRIBUTES_BINDING.canonicalName)
        val invalid = symbols.filter { !it.validate() }.toList()

        for (symbol in symbols) {
            if (symbol is KSClassDeclaration && symbol.validate()) {
                symbol.accept(
                    visitor = ContributeBindingVisitor(codeGenerator, logger),
                    data = Unit
                )
            }
        }
        return invalid
    }
} 
```
In the `process` method, we will get all the classes annotated with our annotation, and visit them in order get the information about the `Component` where the binding has to be installed in, and the super type being implemented by the annotated class. With that information we can create a new File and add our @Module in it.

The `ContributeBindingVisitor` class extends `KSVisitorVoid` class, and we use one of its methods to visit the annotated class declaration. Here we can validate the correctness and expectations of the code.

```kotlin
override fun visitClassDeclaration(classDeclaration: KSClassDeclaration, data: Unit) {
    val annotation = getAnnotation(classDeclaration) ?: return
    val component = resolveType(annotation.arguments.component1().value!!)
    val boundType = annotation.arguments.component2()
    val defaultArgument = annotation.defaultArguments.component1()
    val superTypesCount = classDeclaration.superTypes.count()
    val firstSuperType = classDeclaration.getAllSuperTypes().first()

    if (superTypesCount > 1 && boundType == defaultArgument) {
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

With the help of [Kotlinpoet](https://github.com/square/kotlinpoet), we can easily create kotlin code. This is done by the function `bindingFileSpec`.

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

[OriginatingElement](https://dagger.dev/api/latest/dagger/hilt/codegen/OriginatingElement.html) is needed because we are generating modules.

Now, we can use the annotation and let KSP generate the boilerplate for use.

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

This was more a summary of my own experimentation trying to use KSP, the working project can be found in this [repository](https://github.com/brunohensel/Hilt-Annotation). 