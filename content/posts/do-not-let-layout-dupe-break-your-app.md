---
title: "Don't let Layout duplication break your App"
date: 2024-07-13T08:07:15+02:00
draft: true
---
###### How View Binding Can Help
View binding is a great feature that simplifies interacting with views in your Android app. 
It generates binding classes that directly reference views with IDs in your layouts.
In simpler terms, it replaces the need for `findViewById` to create a reference to a view by ID.

The usage is simple, you create a layout `example_layout.xml`, and a corresponding binding class, `ExampleLayoutBinding`,
will be automatically generated, with which you can access your layout nodes using syntax like `yourBinding.xYz`. You can get more information in
the [official docs](https://developer.android.com/topic/libraries/view-binding).

However, what if you need the same components present in one layout to be rearranged or regrouped for an Ab test. Depending
on the complexity of you layout, managing all placements programmatically at runtime can be challenging. The
fastest way to do it could be duplicating the layout and use the same binding as before.

While duplicating the layout might appear convenient, and your Fragment might not require changes, it introduces the risk of runtime exceptions. You can minimize this risk by writing tests.

### Understanding View Binding generated class

As an example, consider this layout `example_layout`:

```html

<LinearLayout ...>
    <TextView android:id="@+id/myText" .../>
    <Button android:id="@+id/myButton" .../>
    <LinearLayout ...>
        <TextView android:id="@+id/nestedText" .../>
        <Button android:id="@+id/nestedButton" .../>
    </LinearLayout>
</LinearLayout>
```

A binding class will be generated specifically for your layout, named `ExampleLayoutBinding` (notice it matches the layout name with "Binding" appended).
This class provides convenient access to your views through public fields:
* `rootView: LinearLayout`
* `myButton: Button`
* `myText: TextView`
* `nestedButton: Button`
* `nestedText: TextView`

Note that each field corresponds to a view in the layout that has an ID.

The generated class also provides a `bind` method, which is responsible to find a View based on an ID and initialize the
corresponding field. If the ID is not found, a `NullPointerException` is thrown.

Just a sneak peek:

```java

@NonNull
public static ExampleLayoutBinding bind(@NonNull View rootView) {
  // The body of this method is generated in a way you would not otherwise write.
  // This is done to optimize the compiled bytecode for size and performance.
  int id;
  missingId:
  {
    id = R.id.myButton;
    Button myButton = ViewBindings.findChildViewById(rootView, id);
    if (myButton == null) {
      break missingId;
    }
    ...
  }
  String missingId = rootView.getResources().getResourceName(id);
  throw new NullPointerException("Missing required view with ID: ".concat(missingId));
}
```

The documentation states
> If the layout is already inflated, you can instead call the binding class's static bind() method.

Imagine you have a Fragment that can display different layouts based on certain conditions. However, these layouts might
share the _same_ structure with identical view IDs. In such a case, you can create a single `ExampleLayoutBinding` instance
and use its `bind` method to link it with the inflated view, regardless of the specific layout source.

### Catching Layout Inconsistencies

{{< notice note >}} We'll focus on writing an integration test without using an emulator or real device. {{< /notice >}}

As mentioned before, having different layout sources to be inflated and only one binding class to be used in a Fragment
introduces the risk of inconsistencies between the original and copied layouts. These inconsistencies, like mismatched IDs or view types, can lead to runtime exceptions that crash your app.
We'll discuss how to catch these issues before they affect users.

#### Challenge: Maintaining Layout Consistency

The challenge lies in ensuring that any changes made to the original layout (adding/removing views with IDs) are reflected
identically in the duplicated layout(s). Ideally, you wouldn't want layout changes to go unnoticed, especially when
you're not actively working on the code (e.g., on vacation).

#### Solutions

We can write a test that checks the _ids_ and _view type_ equality between original and copied layout, if there is any difference, the
test will fail, and we will catch this problem before the users.

The first approach it came to my mind was to traverse both layout trees and then compare their ids to identify any discrepancies.
If a difference is found, the test fails, alerting you to the inconsistency.

The second one (simpler), leverages the exception-throwing behavior of view binding itself. We can write an integration test that
creates a Fragment scenario and launch a Fragment with a _view binding_ bound to a layout source passed in as argument to the Fragment.
___

#### First approach - Breadth-First Search (BFS) for Layout Traversal

To achieve this, we'll leverage a Breadth-First Search ([BFS](https://en.wikipedia.org/wiki/Breadth-first_search) algorithm.
This algorithm allows us to efficiently traverse the layout hierarchy, visiting all child nodes level by level.

First, we will create a dummy View, in which both view bindings (original and copied layouts) are inflated. This allows us to access the root view of each layout.


```kotlin
class DummyView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : LinearLayout(context, attrs, defStyleAttr) {

    val binding = ExampleLayoutBinding.inflate(LayoutInflater.from(context))
    val copyBinding = CopyExampleLayoutBindingg.inflate(LayoutInflater.from(context))
}
```

Now we'll define a function that will extract the assigned IDs from the layout tree. We need to visit all child
node and its children nodes.

This could be an _expensive_ operation, since it iterates through all child views of a ViewGroup,
regardless of whether they belong to the source layout. This can lead to unnecessary processing, especially for complex layouts with many nested custom views. That's why we need to compare if a `child#sourceLayoutResId` is the same as the root view. 

{{< notice warning >}} For the sake of brevity, the following algorithm is intentionally incomplete. It does not take into account `<merge>` or `<include>` tags. It also does not store view type for further equality check between layouts.  {{< /notice >}}


```kotlin
fun getViewsId(parent: View): Set<String> {
    val viewsId = mutableSetOf<String>()
    val nodes = LinkedList<View>().apply { /*add the root view*/ add(parent) }
    val parentSourceId = parent.sourceLayoutResId //resource ID of the source layout (parent)

    // Utility method to insert view id simple name 
    // Performance is not a concern here
    fun insert(id: Int) {
        // skip views that has no id declared
        if (id == NO_ID) return

        // return buttonId instead of com.example.binding.test:id/buttonId
        // not handling exception on purpose
        val idSimpleName = parent.resources.getResourceName(id).substringAfterLast("/")

        viewsId.add(idSimpleName)
    }

    while (nodes.isNotEmpty()) {
        val child: View = nodes.removeFirst()
        val childSourceId = child.sourceLayoutResId

        // view can only have children if it's a ViewGroup
        if (child !is ViewGroup || child.childCount == 0) {
            if (childSourceId == parentSourceId)
                insert(child.id)
            continue
        }

        // add child's children to be visited later
        child.children.forEach { v ->
            if (childSourceId == parentSourceId) {
                insert(child.id)
            }
            nodes.add(v)
        }
    }

    return viewsId
}
```

Now, we need to write a test that assert the equality of the ids between original and copy bindings.

```kotlin
@RunWith(RobolectricTestRunner::class)
class ExampleLayoutBindingTest {

    @Test
    fun `test id equality between original and copy bindings`() {
        val v = DummyView(InstrumentationRegistry.getInstrumentation().context)
        val originalIds = getViewsId(v.binding.root)
        val copyIds = getViewsId(v.copyBinding.root)

        val diffOriginal = originalIds.minus(copyIds)
        val diffExperimental = copyIds.minus(originalIds)

        assertEquals(diffOriginal, diffExperimental)
    }
}
```

As soon as we delete, let say, `myButton` from the `example_layout`, this test will fail with the following
message `java.lang.AssertionError: expected:<[]> but was:<[myButton]>`. While the default assertion message might not be
very informative, you can create a custom assertion function to provide a clearer error message. Additionally, you can
expand the test to compare not only IDs but also view types for a more comprehensive check.

___

#### Second approach - Let view binding do the heavy lifting

For this approach, you'll need the following additional dependencies:

> * `fragment-testing-manifest` and `fragment-testing` [source](https://developer.android.com/guide/fragments/test)
> * `robolectric` [source](https://github.com/robolectric/robolectric)

For this approach, the work is very minimal since we are relying on the exception-throwing behavior of view binding.
First, we create a dummy Fragment, in which we are binding the `ExampleLayoutBinding` to Fragment's view. We can pass different layout
resources to Fragment's constructor to simulate the real scenario.

```kotlin
class DummyFragment(@LayoutRes contentLayoutId: Int) : Fragment(contentLayoutId) {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        ExampleLayoutBinding.bind(view)
    }
}

@RunWith(RobolectricTestRunner::class)
class ExampleUnitTest {
    
    @Test
    fun assertIdsEquality() {
        launchFragment {
            val res = R.layout.example_layout // it could be also copy_example_layout
            DummyFragment(res)
        }
    }
}
```

When this test runs, the view binding itself checks that the resource used to create a view and the `ExampleLayoutBinding` have the same Views with the same id.
In case there isn't a match, the test will fail with this message: `Missing required view with ID: com.example.binding:id/myButton`.

This test also cover the (im)possibility to have the same id for two different View, e.g. Button with id `button` in the `example_layout` layout and a Text with the same id in the `copy_example_layout` layout. If this happens, the test will fail and the message will be something like `TextView cannot be cast to class Button`.

However, it's possible to have different view types between different layout configurations sharing the same id. For that, you have to tell the compiler what type to use in the generated binding class using `tools:viewBindingType="YourViewType"`. More about the hints in the [official doc](https://developer.android.com/topic/libraries/view-binding#viewbindingtype). 
 ___
That's it, now we can catch the exception earlier and not be surprised in production with a crash. The first approach (traversing the View tree) has its limitation since we are one step behind trying to catch up with built-in behavior of view binding; the second one (using Fragment scenario) is more reliable since we are using the view binding's machinery in the test.

**Note:** You can also write an AndroidTest that launches a real Fragment on an emulator to get an even more realistic test scenario, but this approach might be slower than an integration test without the emulator.
