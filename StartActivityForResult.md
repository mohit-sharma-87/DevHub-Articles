## Introduction

Android has been on the edge of evolution for a while recently, with updates to `androidx.activity:activity-ktx`
to `1.2.0`. It has deprecated `startActivityForResult` in favour of `registerForActivityResult`.

It was one of the first fundamentals that any Android developer has learned, and the backbone of Android's way of
communicating between two components. API design was simple enough to get started quickly but had its cons, like how
it’s hard to find the caller in real-world applications (except for cmd+F in the project 😂), getting results on the
fragment, results missed if the component is recreated, conflicts with the same request code, etc.

Let’s try to understand how to use the new API with a few examples.

## Example 1: Activity A calls Activity B for the result

Old School:

```kotlin
// Caller 
val intent = Intent(context, Activity1::class.java)
startActivityForResult(intent, REQUEST_CODE)

// Receiver 
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    if (resultCode == Activity.RESULT_OK && requestCode == REQUEST_CODE) {
        val value = data?.getStringExtra("input")
    }
}
```

New Way:

```kotlin

// Caller 
val intent = Intent(context, Activity1::class.java)
getResult.launch(intent)

// Receiver 
private val getResult =
    registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) {
        if (it.resultCode == Activity.RESULT_OK) {
            val value = it.data?.getStringExtra("input")
        }
    }
```

As you would have noticed, `registerForActivityResult` takes two parameters. The first defines the type of
action/interaction needed (`ActivityResultContracts`) and the second is a callback function where we receive the result.

Nothing much has changed, right? Let’s check another example.

## Example 2: Start external component like the camera to get the image:

```kotlin
//Caller
getPreviewImage.launch(null)

//Receiver 
private val getPreviewImage = registerForActivityResult(ActivityResultContracts.TakePicture { bitmap ->
    // we get bitmap as result directly
})
```

The above snippet is the complete code getting a preview image from the camera. No need for permission request code, as
this is taken care of automatically for us!

Another benefit of using the new API is that it forces developers to use the right contract. For example,
with `ActivityResultContracts.TakePicture()` — which returns the full image — you need to pass a `URI` as a parameter
to `launch`, which reduces the development time and chance of errors.

Other default contracts available can be
found [here](https://developer.android.com/reference/androidx/activity/result/contract/package-summary).

---

## Example 3: Fragment A calls Activity B for the result

This has been another issue with the old system, with no clean implementation available, but the new API works
consistently across activities and fragments. Therefore, we refer and add the snippet from example 1 to our fragments.

---

## Example 4: Receive the result in a non-Android class

Old Way: 😄

![](https://mongodb-devhub-cms.s3.us-west-1.amazonaws.com/giphy_f99f638e0c.webp)

With the new API, this is possible using `ActivityResultRegistry` directly.

```kotlin
class MyLifecycleObserver(private val registry: ActivityResultRegistry) : DefaultLifecycleObserver {

    lateinit var getContent: ActivityResultLauncher<String>

    override fun onCreate(owner: LifecycleOwner) {
        getContent = registry.register("key", owner, GetContent()) { uri ->
            // Handle the returned Uri
        }
    }

    fun selectImage() {
        getContent.launch("image/*")
    }
}

class MyFragment : Fragment() {
    lateinit var observer: MyLifecycleObserver

    override fun onCreate(savedInstanceState: Bundle?) {
        // ...

        observer = MyLifecycleObserver(requireActivity().activityResultRegistry)
        lifecycle.addObserver(observer)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val selectButton = view.findViewById<Button>(R.id.select_button)

        selectButton.setOnClickListener {
            // Open the activity to select an image
            observer.selectImage()
        }
    }
}
```

## Summary

I have found the registerForActivityResult useful and clean. Some of the cons, in my opinion, are:

1. Improve the code readability, no need to remember to jump to `onActivityResult()` after `startActivityForResult`.

2. `ActivityResultLauncher` returned from `registerForActivityResult` used to launch components, clearly defining the
   input parameter for desired results.

3. Removed the boilerplate code for requesting permission from the user. 

Hope this was informative and enjoyed reading it.
