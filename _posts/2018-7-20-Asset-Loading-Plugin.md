---
layout: post
date: 2018-7-20
title: "Making an Asset Loader Plugin in C++"
img: brick_mountains.jpg
published: false
tags: [C++, Engine Development, Vulkan, Plugins]
---

# Creating an Asset Loading Plugin in C++

It will be a common task in an engine - or any sort of application, really, to load
data from disk. And there are a number of things we'll want this system to be able to
accomplish:

- Be flexible, i.e. allow users to register new loading methods as they see fit
- Avoid redundant re-loads: store data loaded, and if a user requests an already-loaded
file just return this file instead of loading it from disk again
- Don't block the main thread while loading data
- Call a user-supplied function once data is loaded, to notify them that they can proceed
with using the loaded data

Additionally, in my case in particular, this will be functionality exposed through a resource-focused
plugin in [VulpesSceneKit](https://github.com/fuchstraumer/VulpesSceneKit). So we can't use virtual
functions, pass user types (or library types) across the plugin/DLL boundary, or use templates. This
causes some design difficulties from the very start.

### Loading Unique Data Types

Let's start from the very basics, forgetting for a moment that a DLL would already dismiss this as an
option:

{% highlight cpp %}
class AssetLoader {
public:
    LoadedData LoadFile(const char* fname);
};
{% endhighlight %}

This won't be ideal. We now have two specialized functions for loading two types of data buried
in our asset loaders guts. These two functions might both be a lot of work to write and maintain,
and we're going to have to add more and more unique functions as we progress. So, that's not great.

Another potential option would be a virtual interface class that users inherit from, defining a `Load()`
function that the asset loader calls. Then we can have the internal data of the class be whatever we
like, and we don't have to modify the loaders code to add or maintain unique methods of translating file
data into program data.

But alas, I'm working with a plugin system and I can't link plugins to each other. So a user couldn't inherit
from the base class, and couldn't safely pass it in and out of the DLL. Heck. But maybe an old design pattern
can be used here: the factory pattern!

{% highlight cpp %}
class AssetLoader {
public:
    using FactoryFunctor = void*(*)(const char* fname);
    using SignalFunctor = void(*)(void* loaded_data);
    void Subscribe(const char* file_type, FactoryFunctor func);
    void Load(const char* file_type, const char* fname, SignalFunctor fn);
};
// Client code 

static void LoadObjFile(const char* fname) {
    return new ObjFile(fname);
}

void VulkanScene::setupAssets() {
    AssetSystem.Subscribe("OBJ", LoadObjFile);
}
{% endhighlight %}

Okay, this is starting to come together! We'll return a `void*`, and pass that `void*` back into the user-supplied
signal function. Inside that function, we can use a cast to cast into whatever type we want (e.g, `CompressedTextureData`)
or something. This satisfies requirement 1 already, so let's try requirement 2.

### Storing Loaded Data, and Avoiding Re-Loading

The approach to this isn't too wild - we'll keep the `void*` stored in our asset loader, most likely in a map mapping
the filename to a data pointer. Then it'll be super easy to check if we've already loaded something, like so:

{% highlight cpp %}
void ResourceLoader::Load(const char* file_type, const char* file_path, SignalFunctor signal) {
    namespace fs = std::experimental::filesystem;
    // convert to absolute path, so even if we request the same file from
    // multiple locations or with slightly separate starting path values,
    // we ultimately get the same path to read data from.
    fs::path fs_file_path = fs::absolute(fs::path(file_path));
    // have to convert to the string, as we can't currently hash fs::path
    const std::string absolute_path = fs_file_path.string();

    if (resources.count(absolute_path) != 0) {
        // Increment ref count
        ++resources.at(absolute_path).RefCount;
        // Call the signal saying this resource is already loaded
        signal(resources.at(absolute_path).Data);
        return;
    }

    // and our usual loading logic follows
}
{% endhighlight %}

Okay, great. But how we do decide how long to keep the data around? We probably don't want to persist everything in 
RAM constantly - so we'll want some way to know we're done the data. Reference counting seems reasonable: it just 
requires us to trust the user to call an `Unload()` function when they're done with a certain files data. This way 
users are able to keep data around if they like,  by not calling the `Unload()` function immediately - or they can 
choose to leave things loaded in RAM for a while. Monitoring RAM pressure isn't something I want to do in this simple 
of a system, and I don't want to make things more complicated than they already are. Keeping systems small and limited 
in scope/responsibility is a new trend of mine, but it's one I'm rapidly learning to appreciate. Regardless,
our structure repsenting the loaded data looks like so:

{% highlight cpp %}
struct ResourceData {
    void* Data;
    std::string FileType;
    std::string AbsoluteFilePath;
    size_t RefCount{ 0 };
};
{% endhighlight %}

But wait, how do we delete a `void*`? This is going to be undefined behavior, as there's no destructor for a `void*`. And
it's _very_ likely we'll have significant quantities of data attached to a loaded asset that we do want to make sure
are properly destroyed and released from memory.

It took me a while to figure this one out, but the answer is surprisingly simple. We're going to add another function
pointer to our factory registration method:

{% highlight cpp %}
class AssetLoader {
public:
    using FactoryFunctor = void*(*)(const char* fname);
    using DeleteFunctor = void(*)(void* obj_instance);
    void Subscribe(const char* file_type, FactoryFunctor func, DeleteFunctor del_fn);
};
{% endhighlight %}

Then, the client code implementing this delete function becomes trivial (as a bonus, I imagine either generating these functions via a script or
via templates/virtual classes would be rather easy, too!):

{% highlight cpp %}
static void* LoadObjFile(const char* fname) {
    return new ObjFile(fname);
}

static void DestroyObjFile(void * obj_file) {
    ObjFile* model = reinterpret_cast<ObjFile*>(obj_file);
    delete model;
}

void VulkanScene::setupAssets() {
    AssetSystem.Subscribe("OBJ", LoadObjFile, DestroyObjFile);
}
{% endhighlight %}

Since we've casted to the correct type before calling `delete`, we now know for certain that the type will have it's 
destructor called properly and can trust that things will be unloaded successfully. And by doing it like this, we
can still keep our plugin strictly decoupled from any clients, which is a big advantage! Onto our next requirement, though.

### Thread the Loading Operations

Disk I/O can be an expensive operation, and it's also likely that users will want to post-process the data after loading.
Take the ObjModel example in [VulpesSceneKit](https://github.com/fuchstraumer/VulpesSceneKit/blob/master/tests/plugin_tests/ResourceContextSceneTest/ObjModel.cpp), for example.

In that case, we load a bunch of data from disk then try to reduce duplicated vertices in the loaded data, making sure 
that we only end up storing and using unique vertices (note - this is nearly directly copied from how Overv does it in
his excellent [Vulkan Tutorial series](https://vulkan-tutorial.com/)). This helps reduce RAM usage while loading/staging the asset,
and reduces VRAM usage (and potentially the expense of drawing the object, as well). It seems like we are heading towards a 
producer-consumer setup - the producer will be the Load function called by clients, and the consumers (potentially
multiple, as I don't think one thread will be enough here) will be the threads executing the load functions.

Let's create a `loadRequest` structure to use internally, holding everything we need to load data on a separate thread:

{% highlight cpp %}
struct loadRequest {
    loadRequest(ResourceData& dest) : destinationData(dest) {}
    ResourceData& destinationData;
    SignalFunctor signal;
};
{% endhighlight %}

I chose the constructor as I didn't want to copy the `ResourceData` structure into another thread, and because we should only
be modifying one member of that object in the other thread, anyways. We'll then do the following when a user calls `Load()`. 
I chose to copy the two function pointers, though: FactoryFunctor 

{% highlight cpp %}
void ResourceLoader::Load(const char* file_type, const char* file_path, SignalFunctor signal)
    ResourceData data;      
    data.FileType = file_type;
    data.AbsoluteFilePath = absolute_path;
    data.RefCount = 1;
    auto iter = resources.emplace(absolute_path, data);

    loadRequest req(iter.first->second);
    req.requester = _requester;
    req.signal = signal;
    {
        std::lock_guard<std::mutex> guard(queueMutex);
        requests.push_back(req);
    }
    cVar.notify_one();
}
{% endhighlight %}

Here, we use a `std::lock_guard` and a `std::condition_variable` to implement our producer-consumer idiom: the consumer will wait on
the condition variable until it is able to satisfy a condition, and checks the condition when notified by a call to `notify_one()`. So,
we have the thread that is producing the requests acquire the mutex (to safely mutate the requests queue), add a request, then call
notify a single waiting thread that there is potentially a request available (note that the mutex is implicitly released after we
exit the scope it was declared in, thanks to the wondrous `std::lock_guard` <3).

Our worker thread (or threads) is executing a function like so:

{% highlight cpp %}
void ResourceLoader::workerFunction() {
    while (!shutdown) {
        std::unique_lock<std::mutex> lock{queueMutex};
        cVar.wait(lock, [this]()->bool { return shutdown || !requests.empty(); });
        if (shutdown) {
            return;
        }
        loadRequest request = requests.front();
        requests.pop_front();
        FactoryFunctor factory_fn = factories.at(request.destinationData.AbsoluteFilePath);
        lock.unlock();
        request.destinationData.Data = factory_fn(request.destinationData.AbsoluteFilePath.c_str());
        request.signal(request.requester, request.destinationData.Data);
    }
}
{% endhighlight %}

`shutdown` is an atomic boolean used to get all threads to re-join when we are trying to shutdown/stop our system - otherwise we might
potentially have threads in a detached state just running away into nowhere. Here, we are now using a `std::unique_lock` instead of
a `std::lock_guard` - a `std::unique_lock` also will release the mutex once it exits scope, but unlike a guard we can explicitly unlock
the attached mutex once we are done with it. Additionally, `std::condition_variable` just takes it as an argument for `wait()`.

Note that the wait function

##### Potential Issues and Fixes When Using Multiple Threads

I quickly noted during my tests (running `ResourceContextSceneTest` in VulpesSceneKit) that attempting to join my two worker threads
was failing on a deadlock. And behavior here became weird - calling `notify_all()` would cause a crash, as the notified thread would check
requests and get an invalid value for `empty()` - as it's size had gone to the max for a `size_t`, and retrieving the first element
caused a segfault. Which felt a little odd.

### Call a User-Supplied "Signal" Function


### Afterthoughts and Conclusion

