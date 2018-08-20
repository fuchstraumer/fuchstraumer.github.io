---
layout: post
date: 2018-7-20
title: "Making an Asset Loader Plugin in C++"
img: brick_mountains.jpg
published: true
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
plugin in [Caelestis](https://github.com/fuchstraumer/Caelestis). So we can't use virtual
functions, pass user types (or library types) across the plugin/DLL boundary, or use templates. This
causes some design difficulties from the very start.

### Loading Unique Data Types

Let's start from the very basics, forgetting for a moment that a DLL would already dismiss this as an
option:

{% highlight cpp %}
class ResourceLoader {
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
class ResourceLoader {
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
are properly destroyed/releasted.

It took me a while to figure this one out, but the answer is surprisingly simple. We're going to add another function
pointer to our factory registration method:

{% highlight cpp %}
class ResourceLoader {
public:
    using FactoryFunctor = void*(*)(const char* fname);
    using SignalFunctor = void(*)(void* loaded_data);
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
can still keep our plugin strictly decoupled from any clients, which is a big advantage! 

### Thread the Loading Operations

Our next requirement seems fairly apparent: Disk I/O can be an expensive operation, and it's also likely that users will want to post-process the data after loading. Take the ObjModel example in [Caelestis](https://github.com/fuchstraumer/Caelestis/blob/master/tests/plugin_tests/ResourceContextSceneTest/ObjModel.cpp), for example.

In that case, we load a bunch of data from disk then try to reduce duplicated vertices in the loaded data, making sure 
that we only end up storing and using unique vertices (note - this is nearly directly copied from how Overv does it in
his excellent [Vulkan Tutorial series](https://vulkan-tutorial.com/)). This helps reduce RAM use while loading/staging the asset,
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
be modifying one member of that object in the other thread, anyways. We'll then do the following when a user calls `Load()`:

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
we have the thread that is producing the requests acquire and lock the mutex (to safely mutate the requests queue), add a request, then call
`notify_one()` on the condition variable. This notifies a single waiting thread that there is potentially a request available for it to process.

Worker threads have been executing the following function:

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

They should be spending most of their time sleeping on `cVar.wait()`: waiting for their call to wake up and process data, at which point they check against `shutdown` and to see if the queue isn't empty. Here, we are now using a `std::unique_lock` instead of a `std::lock_guard` - a `std::unique_lock` will also release the mutex once it exits scope, but unlike a guard we can explicitly `lock`, `try_lock`, and `unlock` the attached mutex. Lastly, `std::condition_variable` just takes it as an argument for `wait()`.

To summarize what's going on here: we release the lock once we call `cVar.wait()` (we had implicitly locked it upon construction of the `unique_lock`) - but the current thread is blocked (so, our worker thread) and adds itself to a list of threads waiting on `cVar`. The thread is woken, as mentioned, when we call one of the condition variables notify functions - in this case, we use the supplied lambda function to check some conditions and make sure we _actually_ should have awoken.

If the condition returns true, the thread unblocks and continues execution. Here, we unblock if we're shutting down - so we can exit execution
fully and halt this thread for good - or because there are requests for us to process. If there are requests to process we pop a request from the queue and grab a factory function pointer. Once we've done that, we unlock the lock so that other threads can potentially acquire it (as we're no longer mutating potentially shared data). From there, it's just a matter of executing the two function pointers we have. 

##### Potential Issues and Fixes When Using Multiple Threads

I quickly noted during my tests (running `ResourceContextSceneTest` in VulpesSceneKit) that attempting to join my two worker threads (when stopping the system) was failing on a deadlock. And behavior here became weird - calling `notify_all()` would cause a crash, as the notified thread would check requests and get an invalid value for `empty()` - as it's size had gone to the max for a `size_t`, and retrieving the first element caused a segfault. Which felt a little odd. 

I believe the issue related to not using a `std::recursive_mutex`: this object is able to propagate state across multiple threads competing for it, so switching to that plus adding another check for the `shutdown` flag (if true, we return immediately before executing further) seemed to fix the problem for now.

Lastly, what if a user calls `Load()` on something that is _already_ being loaded? The answer is fairly simple: we keep a list of assets that are currently
in the process of loading, and make sure to not add them to our final container (`resources`, here) until they're truly complete. Otherwise, we would check
`resources` and find a suitable `ResourceData` structure, with potentially a `Data` member filled out - however, the reality is that it's not quite loaded yet and isn't actually useable. For the sake of brevity, I won't show this code here - but it is available in `Caelestis`.

### Call a User-Supplied "Signal" Function

Once our loading is complete, we're going to call the user supplied signal function - this lets the user process the data as they see fit. There's a slight problem in our signal function signature that will quickly become apparent in-use, however:

{% highlight cpp %}
using SignalFunctor = void(*)(void* loaded_data);
{% endhighlight %}

The function signature assumes that we are either working with a static function, or a member function of the appropriate signature. The latter will probably be common: I myself had some `ObjModel` class with a `FinalizeCreation(void* data)` function that I wanted to call - and quickly realized that this won't work. We can't use member functions with our C-style interface, and the other option would be some less-than-ideal thing like:

{% highlight cpp %}
static ObjModel* currModelPtr = nullptr;
static void FinalizeCreationStaticFn(void* data) {
    currModelPtr->FinalizeCreation(data);
}

void main(int argc, char* argv[]) {
    PluginManager& manager = PluginManager::GetPluginManager();
    manager.LoadPlugin("resource_context.dll");
    resource_api = reinterpret_cast<ResourceContext_API*>(manager.RetrieveAPI(RESOURCE_CONTEXT_API_ID));
    // register our creation/destruction functions:
    resource_api->RegisterFileTypeFactory("OBJ", &ResourceTestScene::LoadObjFile, 
        &ResourceTestScene::DestroyObjFileData);
    // doing our usual setup, etc, get to asset loading - but first need to set "currModelPtr"
    currModelPtr = &ResourceTestScene.ObjModel;
    // okay now we can load it
    resource_api->LoadFile("OBJ", HouseObjFile.c_str(), FinalizeCreationStaticFn);
}
{% endhighlight %}

Gross. For one, this just feels vaguely "wrong", like a code smell. It can definitively become bad, though: if we're loading multiple models, we now have to make sure to update that static instance pointer before each model is loaded. Luckily the fix is simple - and I nearly wanted to slap myself upside the head when a coworker pointed it out to me (bless him, though). We can add one more parameter to our signal function, and store one more item in our asset loader:

{% highlight cpp %}
using SignalFunctor = void(*)(void* object_instance, void* loaded_data);
struct loadRequest {
    loadRequest(ResourceData& dest) : destinationData(dest) {}
    ResourceData& destinationData;
    void* instancePtr;
    SignalFunctor signal;
};
{% endhighlight %}

Now we can call our signal function with a user-designated instance pointer, which allows us to easily call a member function but still operate and interface
to this system through a C-style interface. When calling our signal function from the resource loader, upon completion of the loading step, we just pass in
`instancePtr` to `signal`, and the user can do something like this:

{% highlight cpp %}
static void FinalizeCreationFn(void* instance, void* data) {
    ObjModel* model_ptr = reinterpret_cast<ObjModel*>(instance);
    model_ptr->FinalizeCreation(data);
}

void main(int argc, char* argv[]) {
    // all of our previous steps remain the same
    // however, we now just submit one extra parameter to our loading function: an instance pointer
    resource_api->LoadFile("OBJ", HouseObjFile.c_str(), &ResourceTestScene.ObjModel, 
        FinalizeCreationStaticFn);
}
{% endhighlight %}

And just like that, one of the last major potential problems with our asset loader system has been solved!

### Afterthoughts and Conclusion

This particular plugin was one that really began to test my resolve - it presented me with a couple unique challenges, both in terms of new topics I had never really dived into before (the deeper details of threading), and challenges related to my design choices. Having to interface with a shared library and keep things C-style was still fairly new to me, and handling things like member functions without the implicit instance pointer proved to be a challenge that took some thought.

There's still ample room for improvement - but it works well enough as is for now, and I'm glad that I took the time to design this particular system as multithreaded from the get-go. I cannot imagine that going back and retrofitting a single-threaded system to be multithreaded would've been much fun. Regardless, any comments or criticism would be appreciated - I have a lot to learn, and I can't do so without having my mistakes made abundantly clear!

To close out, here's an example video of me testing this very plugin on Mac OSX - which served as verification that Vulkan (shimmed to Metal thanks to MoltenVk) was working on Mac, and that I wasn't going to have to deal with any weird bugs (thank heck) on Mac. Here, we asynchronously load three items: a skybox texture from a  `.dds` file, a `.png` texture for the house, and the `.obj` mesh data for the house:

<iframe width="400" height="300" 
src="https://giant.gfycat.com/CautiousKlutzyAcornweevil.webm">
</iframe>


